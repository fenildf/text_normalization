# Text Normalization for English

## 问题描述
所谓“文本正则”，即将手写形式的文本转换成语音形式的文本。

例子：

- 手写：`A baby giraffe is 6ft tall and weighs 150lb.`
- 语音：`A baby giraffe is six feet tall and weighs one hundred fifty pounds.`

## 调研
目前kernel上主要以收集大辞典的方法为主流。

基于RNN的方法在paper中提到说效果不佳。

token的所有类别：`['PLAIN', 'PUNCT', 'DATE', 'LETTERS', 'CARDINAL', 'VERBATIM', 'DECIMAL', 'MEASURE', 'MONEY', 'ORDINAL', 'TIME', 'ELECTRONIC', 'DIGIT', 'FRACTION', 'TELEPHONE', 'ADDRESS']`。
其中，`'PLAIN', 'PUNCT'`是输出和输入相同的类别。但训练数据中，只有`'PUNCT'`的输出和输入完全相同，`'PLAIN'`的输入和输出相同的个数是：7317175，总共个数是：7353693。比例是：0.995034059757458。这里后续需要继续调查。

在训练集中，`'PLAIN', 'PUNCT'`所占的比例是0.931。

## 解决方法进化记录
### 2017-11-2

![]()

0.9937-198/489

使用kernel中提供的方法尝试了一下。大致上就是从训练语料中提取一个大辞典，这个词典包括所有词以及其映射。不同的方法引进了更多的额外训练语料。

### 2017-11-5

![]()

0.9164

用xgboost训练了一个二分类器，判别某个token是不是属于`'PLAIN'+'PUNCT'`。是则输入和输出相同，否则输出`""`。得到这个准确率。

### 2017-11-6

0.9835

用xgboost训练了一个三分类器，判别某个token是否属于`'PLAIN', 'PUNCT'`还有其他。

### 2017-11-8

最终发现还是要使用全分类器`full_classifier`，即16个类别全都分全。然后针对每个类别手动设计normalize的过程`full_replace`和`replace_by_rule`。并使用`test/test_one_class.py`一个一个类别调准确率。这一阶段准确率的提升主要来自对每个单独类别的normalize的优化。

准确率：0.9908->0.9921->0.9937->0.9942->0.9951->0.9954->0.9956

排名：75/554，已经进入铜牌区域，但也进入了瓶颈，单独类别的转换准确率很难找到大幅提升的改进点。

使用脚本`full_replace_train.py`发现所有的训练数据，如果知道其分类，那么normalize的准确率是：`9913201 / 9918441 = 0.9994716911659807`。也就是说，错误的数目只有5k+了，继续修改normalize上升空间有限。

### 2017-11-11

转换思路，继续提高分类器的准确率。

#### 思路一：增加人为判定规则
根据对数据的分析，发现分类器有几类典型错误：
- 1.被误认为是`cardinal`的实际上的`date`的年份。这类情况特征明显，长度是4的数字都判为`date`即可。
- 2.被误认为是`plain`的实际上的`electronic`的网站，比如`baidu.com`之类的，写一个简单的规则判断。

上面总结的规则都集中在`patch_classifier.py`脚本中，在分类器给出判定结果之后再跑一遍，作为对分类器的修正。

#### 思路二：直接提升模型准确率

- 1.**特征工程**：token的长度；是否是一句话中第一个token。目前特征工程对模型准确率没有明显提升。`v2`。
- 2.**调整权重**：因为本次任务是非常**unbalanced**的分类任务，因此最好对于不同类别的数据，在计算loss的时候，采用不同的权重。`v3`。
- 3.**调参**：关键参数：`max_depth`， `round_num`。`v2`。

## 其他信息

### 使用到的第三方包

- `roman`:罗马数字和阿拉伯数字的转换。
- `num2words`:文字和数字的转换，支持序数的数字。
- `inflect`:单词复数形式的转换等。
