---
title: 一个简单的通过 sql 查询 prometheus 的转换器实现
date: 2025-10-14 14:00
categories: [技术]
tags: [prometheus, sql, calcite]
math: false
---

## 背景

可观测的一个重要的趋势是链路、日志、指标的统一存储和使用。大模型时代，在业务层实现链路、日志、指标的统一查询语言将是一个重要的过渡方案。本文尝试实现一个简单的通过 sql 查询 prometheus 指标的工具。

## 实现步骤

经调研，我决定基于 [Apache Calcite](https://calcite.apache.org/)，实现 sql 解析并对接真实的 promql 查询终端。

Apache Calcite 是一个优秀的动态数据管理框架，提供了如：SQL 解析、SQL 校验、SQL 查询优化、SQL 生成以及数据连接查询等典型数据库管理功能[^1]。

在Calcite中，查询处理分为逻辑计划和物理计划两个层次，这种分离使得优化更加灵活和模块化。逻辑计划描述了查询要做什么，而不关心具体如何执行。它是对SQL语句的抽象表示，专注于查询的语义而非执行细节。物理计划描述了查询如何执行，包含了具体的算法选择和执行策略。它是逻辑计划的具体实现，考虑了底层存储系统的特性和约束。

我的思路是，利用calcite自带的 sql 解析和逻辑计划分析能力，手动解析最终的物理计划逻辑，并翻译为 prom ql 和实际的查询动作。

### 效果预期

```sql
select `app_name`, `timestamp` , `value` from prometheus.system_metrics_cpu_util where app_name = 'my_app' and hostname = 'host1' and `timestamp` >= 1696118400000 and `timestamp` <= 1696204800000
```

应翻译为 prometheus 查询

```promql
system_metrics_cpu_util{app_name = 'my_app', hostname = 'host1'}
```

并查询出 JSON/CSV 数据，示例如下

```csv
app_name, timestamp, value
my_app, 1696118460000, 0.01
my_app, 1696118520000, 0.02
```

查询的方式是标准的 jdbc 连接和查询语法。如下图所示

```java
Properties info = new Properties();
info.setProperty(CalciteConnectionProperty.LEX.camelName(), Lex.MYSQL.name());
// 获取连接
Connection connection = DriverManager.getConnection("jdbc:calcite:", info);
// 获取Calcite连接
CalciteConnection calciteConnection = connection.unwrap(CalciteConnection.class);
// 获取根模式
SchemaPlus rootSchema = calciteConnection.getRootSchema();

// 注册Prometheus Schema
String dstStr = """
                      [
          {
            "tableName": "system_metrics_cpu_util",
            "fields": [
              {"fieldName": "timestamp", "fieldType": "BIGINT"},
              {"fieldName": "app_name", "fieldType": "VARCHAR"},
              {"fieldName": "hostname", "fieldType": "VARCHAR"},
              {"fieldName": "value", "fieldType": "DOUBLE"}
            ]
          }
        ]
        """;
List<DataSourceTable> dataSourceTables = JSON.parseArray(dstStr, DataSourceTable.class);
Sql2PrometheusSchema promSchema = new Sql2PrometheusSchema(dataSourceTables, false);
rootSchema.add("prometheus", promSchema);
// 创建语句对象
Statement statement = connection.createStatement();

// 执行查询语句
String sql = "select `timestamp` , `value` from prometheus.system_metrics_cpu_util where app_name = 'my_app' and hostname = 'host1' and `timestamp` >= 1696118400000 and `timestamp` <= 1696204800000";
ResultSet rs = statement.executeQuery(sql);
List<Map<String, Object>> list = new ArrayList<>(); // 转换 rs 为最终结果
ResultSetMetaData metaData = rs.getMetaData();
int columnCount = metaData.getColumnCount();
```



### 设计并实现一个 schema

从上面代码的使用可以看到，我们需要先实现一个 Sql2PrometheusSchema。calcite 里的 schema 和 Postgres 的schema的概念比较像。在PostgreSQL中，schema是一个逻辑容器，用于组织和管理数据库中的对象（如表、视图、函数、索引等）。它类似于一个命名空间，可以将相关的数据库对象分组在一起。在 MySQL中，类似的概念是 database。

```java
public class Sql2PrometheusSchema extends AbstractSchema {
    private final Map<String, Table> tableMap = new HashMap<>();
    public Sql2PrometheusSchema(List<DataSourceTable> dataSourceTables, boolean enableQuery) {
        if (Objects.isNull(dataSourceTables)) {
            throw new IllegalArgumentException("dataSourceTables can not be null");
        }
        for (DataSourceTable dataSourceTable : dataSourceTables) {
            String tableName = dataSourceTable.getTableName();
            tableMap.put(tableName, new Sql2PrometheusTable(tableName, dataSourceTable.getFields(), enableQuery));
        }
    }
    @Override
    protected Map<String, Table> getTableMap() {
        return tableMap;
    }
}
```

注意到我们在初始化 Sql2PrometheusSchema 的时候，传入了表的定义 `List<DataSourceTable> dataSourceTables`，表的定义对应不同的 promtheus 的指标和 label，一个 promtheus 指标对应一张表，每个指标的所有 label 即为表的字段。每个表还有额外的2个字段：`timestamp` 和 `value`。

```java
public class DataSourceTable {
    private String tableName;
    private java.util.List<DataSourceTableField> fields;
}
public class DataSourceTableField {
    private String fieldName;
    private String fieldType; // e.g., "STRING", "DOUBLE", "LONG"
    private Boolean metricField; // 是否为指标字段
}
```



### 设计并实现一个 table

`Sql2PrometheusTable` 是该方案的核心。table 的scan方法就是执行解析后的物理计划的地方。只需要解析物理计划，并翻译成 prometheus 的查询语句，然后将查询动作包装成一个Enumerator即可。

```java
public class Sql2PrometheusTable extends AbstractTable implements FilterableTable {
    private static final String TIMESTAMP_FIELD = "timestamp";
    @Getter
    private final String metricTable;
    private final List<DataSourceTableField> fields;
    private final boolean enableQuery;
    private final Map<String, Integer> fieldIndexMap = new HashMap<>();
    private final Map<Integer, String> indexFieldMap = new HashMap<>();
    @Getter
    private boolean active;
    @Getter
    private String query;
    @Getter
    private String activeQuery;

    public Sql2PrometheusTable(String metricTable, List<DataSourceTableField> fields, boolean enableQuery) {
        this.metricTable = metricTable;
        this.fields = fields;
        this.enableQuery = enableQuery;
        for (int i = 0; i < fields.size(); i++) {
            fieldIndexMap.put(fields.get(i).getFieldName(), i);
            indexFieldMap.put(i, fields.get(i).getFieldName());
        }
    }


    @Override
    public Enumerable<Object[]> scan(DataContext root, List<RexNode> filters) {
        this.active = true;
        JavaTypeFactory typeFactory = root.getTypeFactory();

        return new AbstractEnumerable<>() {
            public Enumerator<Object[]> enumerator() {
                // 在这里处理过滤逻辑
                // 具体实现取决于filters中的条件
                return new Sql2PrometheusEnumerator(buildMetricsQueryRequest(typeFactory, filters), fieldIndexMap);
            }
        };
    }


    @Override
    public RelDataType getRowType(RelDataTypeFactory relDataTypeFactory) {
        // 这里应该返回 table 的 schema
        RelDataTypeFactory.Builder builder = relDataTypeFactory.builder();
        for (DataSourceTableField field : fields) {
            SqlTypeName fieldType = SqlTypeName.get(field.getFieldType());
            Objects.requireNonNull(fieldType, "fieldType is null");
            builder.add(field.getFieldName(), fieldType);
        }
        return builder.build();
    }

    private MetricsQueryRequest buildMetricsQueryRequest(JavaTypeFactory typeFactory, List<RexNode> filters) {
        StringBuilder labelSelectors = new StringBuilder(metricTable).append("{");
        List<RexNode> filtersRemoveTs = new ArrayList<>();
        List<TimeRange> timeRanges = new ArrayList<>();
        for (RexNode filter : filters) {
            TimestampVisitor timestampVisitor = new TimestampVisitor(fieldIndexMap.get(TIMESTAMP_FIELD), typeFactory);
            filtersRemoveTs.add(filter.accept(timestampVisitor));
            timeRanges.add(timestampVisitor.getTimeRange());
        }
        if (timeRanges.isEmpty()) {
            throw new IllegalArgumentException("Time range is required");
        }
        TimeRange timeRange = timeRanges.getFirst();

        for (RexNode filter : filtersRemoveTs) {
            PromQLConverterVisitor visitor = new PromQLConverterVisitor(typeFactory, indexFieldMap);
            String labelSelector = filter.accept(visitor);
            if (!labelSelector.isEmpty()) {
                labelSelectors.append(labelSelector).append(", ");
            }
        }

        if (labelSelectors.lastIndexOf(",") == labelSelectors.length() - 2) {
            labelSelectors.delete(labelSelectors.length() - 2, labelSelectors.length());
        }
        labelSelectors.append("}");
        this.query = labelSelectors.toString();
        return new MetricsQueryRequest(this.query, timeRange.start(), timeRange.end(), this.enableQuery);
    }
}
```

### 包装成Enumerator

上面的scan方法返回的是一个Enumerator：`Sql2PrometheusEnumerator`。其入参即 prometheus 的 [query range](https://prometheus.io/docs/prometheus/latest/querying/api/#range-queries) 参数。

```java
public class MetricsQueryRequest {
    private final String query;
    private final long start;
    private final long end;
}
```

在Enumerator里把 prom 查询包装起来，并提供Enumerator接口。

```java
public class Sql2PrometheusEnumerator implements Enumerator<Object[]> {
    private Iterator<Object[]> iterator;
    private final Map<String, Integer> fieldIndexMap;
    private Object[] currentElement;  // 用于存储当前元素的成员变量

    public Sql2PrometheusEnumerator(MetricsQueryRequest metricsQueryRequest, Map<String, Integer> fieldIndexMap) {
        this.fieldIndexMap = fieldIndexMap;
        long start = metricsQueryRequest.getStart();
        long end = metricsQueryRequest.getEnd();
        String query = metricsQueryRequest.getQuery();
        System.out.println("MetricsQueryRequest: " + query + ", start: " + start + ", end: " + end);
        // todo 执行 prom 查询，并将查询结果转换为 Iterator
        this.iterator = fetchData(metricsQueryRequest).iterator();
    }

    @Override
    public Object[] current() {
        if (currentElement == null) {
            throw new NoSuchElementException("No current element, call moveNext() first.");
        }
        return currentElement;
    }

    @Override
    public boolean moveNext() {
        if (iterator.hasNext()) {
            currentElement = iterator.next();
            return true;
        }
        currentElement = null;  // 当没有更多元素时，清除当前元素
        return false;
    }

    @Override
    public void reset() {
        iterator = convertData().iterator();
        currentElement = null;  // 重置时也应清除当前元素
    }

    @Override
    public void close() {
    }
}
```



## 解析物理计划

介绍完实现步骤，我们再来重点看下物理计划的解析。物理计划解析的入口是`Sql2PrometheusTable` 的buildMetricsQueryRequest方法。

### 时间戳提取

sql 里是包含时间的过滤条件的，但是 prometheus 的查询语法里，时间戳范围和 query 是分开的。因此，我们的第一步是从物理计划里把时间戳相关的查询提取出来。即`TimestampVisitor`。

```java
public class TimestampVisitor extends RexShuttle {
    /**
     * 时间戳列的索引
     */
    private final int tsIndex;
    private final RexBuilder rexBuilder;
    private Long start = null;
    private Long end = null;

    public TimestampVisitor(int tsIndex, JavaTypeFactory typeFactory) {
        // Pass true to visit each node only once
        this.tsIndex = tsIndex;
        this.rexBuilder = new RexBuilder(typeFactory);
    }

    @Override
    public RexNode visitCall(RexCall call) {
        if (isTimestampFilter(call)) {
            RexNode right = call.getOperands().get(1);

            setTimeRange(call.getOperator(), (RexLiteral) right);
            return rexBuilder.makeLiteral(true);
        }
        return super.visitCall(call);
    }

    private boolean isTimestampFilter(RexCall call) {
        if (call.getOperands().size() != 2) {
            return false;
        }

        // 检查是否包含时间戳列
        RexNode left = call.getOperands().getFirst();
        if (left instanceof RexInputRef ref) {
            return Objects.equals(tsIndex, ref.getIndex());
        }
        return false;
    }

    private void setTimeRange(SqlOperator sqlOperator, RexLiteral literal) {
        SqlKind kind = sqlOperator.getKind();
        if (SqlKind.GREATER_THAN.equals(kind) || SqlKind.GREATER_THAN_OR_EQUAL.equals(kind)) {
            // ts >, ts >=
            log.debug("greater than");
            this.start = Optional.ofNullable((BigDecimal) literal.getValue()).map(BigDecimal::longValue).orElse(null);
        } else if (SqlKind.LESS_THAN.equals(kind) || SqlKind.LESS_THAN_OR_EQUAL.equals(kind)) {
            // ts <, ts <=
            log.debug("less than");
            this.end = Optional.ofNullable((BigDecimal) literal.getValue()).map(BigDecimal::longValue).orElse(null);
        } else if (SqlKind.SEARCH.equals(kind)) {
            // ts between
            if (literal.getValue() instanceof Sarg<?> sarg) {
                log.debug("sarg: {}", sarg);
                Range<BigDecimal> range = (Range<BigDecimal>) sarg.rangeSet.span();
                this.start = range.lowerEndpoint().longValue();
                this.end = range.upperEndpoint().longValue();
            }
        }
    }

    public TimeRange getTimeRange() {
        // start 不能是 null，但是 end 可以是 null
        Objects.requireNonNull(this.start, "start is null");
        if (Objects.isNull(end)) {
            this.end = new Date().getTime();
        }
        return new TimeRange(this.start, this.end);
    }
}
```

### label查询条件拼接

第二步，是将除了时间戳之外的 where 查询条件拼接为 prometheus 查询的 label 条件。支持的 where 条件包含 =、!=、like、not like，分别对应 prometheus 查询的 =、!=、=~、!~。

```java
public class PromQLConverterVisitor extends RexVisitorImpl<String> {
    private final Map<Integer, String> indexFieldMap;

    public PromQLConverterVisitor(JavaTypeFactory typeFactory, Map<Integer, String> indexFieldMap) {
        super(true);
        this.indexFieldMap = indexFieldMap;
    }

    @Override
    public String visitCall(RexCall call) {
        SqlOperator operator = call.getOperator();

        if (operator == SqlStdOperatorTable.AND) {
            return handleAndCondition(call);
        } else if (operator == SqlStdOperatorTable.OR) {
            return handleOrCondition(call);
        } else {
            return handleComparisonOperator(call);
        }
    }

    @Override
    public String visitInputRef(RexInputRef inputRef) {
        return indexFieldMap.get(inputRef.getIndex());
    }

    @Override
    public String visitLiteral(RexLiteral literal) {
        if (literal.isAlwaysTrue()) {
            return "";
        }
        if (literal.getValue() instanceof Sarg) {
            return handleSargLiteral(literal);
        }
        Object value = literal.getValue2();
        return value != null ? value.toString() : "null";
    }

    public String visitNode(RexNode node) {
        if (node instanceof RexCall) {
            return visitCall((RexCall) node);
        } else if (node instanceof RexInputRef) {
            return visitInputRef((RexInputRef) node);
        } else if (node instanceof RexLiteral) {
            return visitLiteral((RexLiteral) node);
        }
        return "";
    }

    private String handleAndCondition(RexCall call) {
        List<String> conditions = new ArrayList<>();
        for (RexNode operand : call.getOperands()) {
            String condition = visitNode(operand);
            if (!condition.isEmpty()) {
                conditions.add(condition);
            }
        }
        return String.join(",", conditions);
    }

    private String handleOrCondition(RexCall call) {
        List<List<String>> orGroups = new ArrayList<>();
        for (RexNode operand : call.getOperands()) {
            List<String> branchConditions = extractBranchConditions(operand);
            if (!branchConditions.isEmpty()) {
                orGroups.add(branchConditions);
            }
        }
        return combineOrConditions(orGroups);
    }

    private List<String> extractBranchConditions(RexNode node) {
        List<String> conditions = new ArrayList<>();
        if (node instanceof RexCall call) {
            if (call.getOperator() == SqlStdOperatorTable.AND) {
                for (RexNode operand : call.getOperands()) {
                    String condition = visitNode(operand);
                    if (!condition.isEmpty()) {
                        conditions.add(condition);
                    }
                }
            } else {
                String condition = visitNode(node);
                if (!condition.isEmpty()) {
                    conditions.add(condition);
                }
            }
        }
        return conditions;
    }

    private String handleComparisonOperator(RexCall call) {
        if (call.getOperands().size() != 2) {
            return "";
        }

        String columnName = visitNode(call.getOperands().get(0));
        String value = visitNode(call.getOperands().get(1));
        String operator = convertOperator(call.getOperator());
        if (Strings.isNullOrEmpty(columnName) || Strings.isNullOrEmpty(value) || Strings.isNullOrEmpty(operator)) {
            return "";
        }
        return String.format("%s%s\"%s\"", columnName, operator, escapeValue(value));
    }

    private String combineOrConditions(List<List<String>> orGroups) {
        Map<String, Set<String>> labelValueMap = new HashMap<>();

        for (List<String> group : orGroups) {
            for (String condition : group) {
                String[] parts = condition.split("=");
                if (parts.length == 2) {
                    String label = parts[0];
                    String value = parts[1].replaceAll("\"", "");
                    labelValueMap.computeIfAbsent(label, k -> new HashSet<>()).add(value);
                }
            }
        }

        List<String> combined = new ArrayList<>();
        for (Map.Entry<String, Set<String>> entry : labelValueMap.entrySet()) {
            String label = entry.getKey();
            Set<String> values = entry.getValue();
            if (values.size() == 1) {
                combined.add(String.format("%s=\"%s\"", label, values.iterator().next()));
            } else {
                combined.add(String.format("%s=~\"%s\"",
                        label,
                        String.join("|", values)));
            }
        }

        return String.join(",", combined);
    }

    private String convertOperator(SqlOperator operator) {
        if (operator == SqlStdOperatorTable.EQUALS) {
            return "=";
        } else if (operator == SqlStdOperatorTable.NOT_EQUALS) {
            return "!=";
        } else if (operator == SqlStdOperatorTable.LIKE || operator == SqlStdOperatorTable.SEARCH) {
            return "=~";
        } else if (operator == SqlStdOperatorTable.NOT_LIKE) {
            return "!~";
        }
        return "";
    }

    private String escapeValue(String value) {
        return value.replace("\"", "\\\"");
    }

    private String handleSargLiteral(RexLiteral literal) {
        Sarg<NlsString> sarg = (Sarg<NlsString>) literal.getValue();
        Objects.requireNonNull(sarg, "Sarg value is null");
        // 处理范围类型
        return handleSargPoints(sarg);
    }

    private String handleSargPoints(Sarg<NlsString> sarg) {
        List<String> points = new ArrayList<>();

        for (Range<NlsString> range : sarg.rangeSet.asRanges()) {
            if (RangeSets.isPoint(range)) {
                points.add(range.lowerEndpoint().getValue());
            } else {
                log.error("Invalid range: {}", range);
            }
        }

        if (points.size() == 1) {
            return String.format("%s", points.getFirst());
        } else {
            // 多个点使用正则表达式匹配
            return String.format("%s", String.join("|", points));
        }
    }

    private String formatValue(Object value) {
        if (value == null) {
            return "null";
        }
        if (value instanceof String) {
            return "\"" + escapeValue((String) value) + "\"";
        }
        return value.toString();
    }
}
```

至此，我们实现了一个建议的通过 sql 查询 prometheus 数据的工具。打印日志为：

```
MetricsQueryRequest: system_metrics_cpu_util{app_name="my_app",hostname="host1"}, start: 1696118400000, end: 1696204800000
```





[^1]: https://strongduanmu.com/blog/apache-calcite-learning-materials.html 挺不错的系列文章
