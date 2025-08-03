## 5.10 时间序列交叉验证

https://otexts.com/fpp3/tscv.html 原文地址

https://otexts.com/fpp3cn/ 已有中文版，那是不是可以考虑写一个 Python 版本。https://forecastegy.com/posts/time-series-cross-validation-python/ python 的交叉验证。



时间序列交叉验证是一种更复杂的训练/测试集版本。在这个过程中，有一系列的测试集，每个测试集只包含一个观测值。相应的训练集只包含在形成测试集的观测值之前发生的观测值。因此，在构建预测时不能使用未来的观测值。由于在小训练集上无法得到可靠的预测，最早的观测值不被视为测试集。

下图展示了一系列的训练集和测试集，其中蓝色观测值组成了训练集，橙色观测值组成了测试集。

https://otexts.com/fpp3/fpp_files/figure-html/cv1-1.png

预测准确度是通过对测试集进行平均计算得出的。这个过程有时被称为“滚动预测起点评估”，因为预测的“起点”会随着时间的推移而滚动。

随着时间序列预测的进行，一步预测可能不如多步预测相关。在这种情况下，基于滚动预测起点的交叉验证程序可以进行修改，以允许使用多步误差。假设我们对能够产生良好的 44 步预测的模型感兴趣。那么相应的图表如下所示。

https://otexts.com/fpp3/fpp_files/figure-html/cv4-1.png

在下面的例子中，我们将通过时间序列交叉验证和残差准确度来比较准确度。函数 `stretch_tsibble()` 用于创建多个训练集。在这个例子中，我们从长度为 `.init=3` 的训练集开始，并通过增加连续训练集的大小来进行比较，增量为 `.step=1` 。

```R
# Time series cross-validation accuracy
google_2015_tr <- google_2015 |>
  stretch_tsibble(.init = 3, .step = 1) |>
  relocate(Date, Symbol, .id)
google_2015_tr
#> # A tsibble: 31,875 x 10 [1]
#> # Key:       Symbol, .id [250]
#>    Date       Symbol   .id  Open  High   Low Close Adj_Close  Volume   day
#>    <date>     <chr>  <int> <dbl> <dbl> <dbl> <dbl>     <dbl>   <dbl> <int>
#>  1 2015-01-02 GOOG       1  526.  528.  521.  522.      522. 1447600     1
#>  2 2015-01-05 GOOG       1  520.  521.  510.  511.      511. 2059800     2
#>  3 2015-01-06 GOOG       1  512.  513.  498.  499.      499. 2899900     3
#>  4 2015-01-02 GOOG       2  526.  528.  521.  522.      522. 1447600     1
#>  5 2015-01-05 GOOG       2  520.  521.  510.  511.      511. 2059800     2
#>  6 2015-01-06 GOOG       2  512.  513.  498.  499.      499. 2899900     3
#>  7 2015-01-07 GOOG       2  504.  504.  497.  498.      498. 2065100     4
#>  8 2015-01-02 GOOG       3  526.  528.  521.  522.      522. 1447600     1
#>  9 2015-01-05 GOOG       3  520.  521.  510.  511.      511. 2059800     2
#> 10 2015-01-06 GOOG       3  512.  513.  498.  499.      499. 2899900     3
#> # ℹ 31,865 more rows
```

`.id` 列提供了一个新的键，用于指示不同的训练集。可以使用 `accuracy()` 函数来评估在训练集上的预测准确性。

```R
# TSCV accuracy
google_2015_tr |>
  model(RW(Close ~ drift())) |>
  forecast(h = 1) |>
  accuracy(google_2015)
# Training set accuracy
google_2015 |>
  model(RW(Close ~ drift())) |>
  accuracy()
```

这里是表格



正如预期的那样，残差的准确度测量值较小，因为相应的“预测”是基于对整个数据集进行拟合的模型，而不是真正的预测。

选择最佳预测模型的一个好方法是使用时间序列交叉验证计算出具有最小均方根误差（RMSE）的模型。

### 示例：使用交叉验证的预测时间范围准确度

google_2015 子集是 gafa_stock 数据的一个子集，如图5.9所示，包括2015年纳斯达克交易所上谷歌公司每个交易日的收盘股价。

下面的代码评估了1到8步预测漂移的预测性能。图表显示，随着预测时间范围的增加，预测误差也增加，这是我们所预期的。

```R
google_2015_tr <- google_2015 |>
  stretch_tsibble(.init = 3, .step = 1)
fc <- google_2015_tr |>
  model(RW(Close ~ drift())) |>
  forecast(h = 8) |>
  group_by(.id) |>
  mutate(h = row_number()) |>
  ungroup() |>
  as_fable(response = "Close", distribution = Close)
fc |>
  accuracy(google_2015, by = c("h", ".model")) |>
  ggplot(aes(x = h, y = RMSE)) +
  geom_point()
```

https://otexts.com/fpp3/fpp_files/figure-html/CV-accuracy-plot-1.png
图5.24：以预测时间跨度为函数的漂移方法应用于谷歌收盘股价的均方根误差（RMSE）。

