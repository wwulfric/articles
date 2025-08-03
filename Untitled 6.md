llama2.c 源码



读取二进制文件：model，tokenizer，sampler（分别是什么，debug看一下各个成分）

结合https://github.com/opensourceai/spark-plan/blob/master/article/2019/11/%E5%9B%BE%E8%A7%A3GPT-2(%E5%8F%AF%E8%A7%86%E5%8C%96Transformer%E8%AF%AD%E8%A8%80%E6%A8%A1%E5%9E%8B).md 等，把GPT2（或者llama2）的模型图示出来  https://github.com/wdndev/llm_interview_note/blob/main/02.%E5%A4%A7%E8%AF%AD%E8%A8%80%E6%A8%A1%E5%9E%8B%E6%9E%B6%E6%9E%84/llama%202%E4%BB%A3%E7%A0%81%E8%AF%A6%E8%A7%A3/llama%202%E4%BB%A3%E7%A0%81%E8%AF%A6%E8%A7%A3.md 这个链接也不错

https://github.com/RahulSChand/llama2.c-for-dummies 这个更好



encode：如何把输入最终编码到 token list

generate：逐个计算token。注1:prompt里的token也要参与计算，注意力需要从一开始就load进来。

核心方法：forward，前向传播。先拿到token的嵌入（从嵌入表里查询）。

Q：嵌入表来自模型文件，嵌入表是训练出来的吗





llama2.c视角的公式表达



按照架构每一层的具体实现



kvcache 是如何cache的
