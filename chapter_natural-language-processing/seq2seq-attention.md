# 编码器—解码器（seq2seq）和注意力机制


在基于词语的语言模型中，我们使用了[循环神经网络](../chapter_recurrent-neural-networks/rnn-gluon.md)。它的输入是一段不定长的序列，输出却是定长的，例如一个词语。然而，很多问题的输出也是不定长的序列。以机器翻译为例，输入是可以是英语的一段话，输出可以是法语的一段话，输入和输出皆不定长，例如

> 英语：They are watching.

> 法语：Ils regardent.

当输入输出都是不定长序列时，我们可以使用编码器—解码器（encoder-decoder）或者seq2seq。它们分别基于2014年的两个工作：

* Cho et al., [Learning Phrase Representations using RNN Encoder-Decoder for Statistical Machine Translation](https://www.aclweb.org/anthology/D14-1179)
* Sutskever et al., [Sequence to Sequence Learning with Neural Networks](https://papers.nips.cc/paper/5346-sequence-to-sequence-learning-with-neural-networks.pdf)

以上两个工作本质上都用到了两个循环神经网络，分别叫做编码器和解码器。编码器对应输入序列，解码器对应输出序列。下面我们来介绍编码器—解码器的设计。


## 编码器—解码器

编码器和解码器是分别对应输入序列和输出序列的两个循环神经网络。我们通常会在输入序列和输出序列后面分别附上一个特殊字符'&lt;eos&gt;'（end of sequence）表示序列的终止。

### 编码器

编码器的作用是把一个不定长的输入序列转化成一个定长的背景向量$\mathbf{c}$。该背景向量包含了输入序列的信息。常用的编码器是循环神经网络。

我们回顾一下[循环神经网络](../chapter_recurrent-neural-networks/rnn-scratch.md)知识。假设循环神经网络单元为$f$，在$t$时刻的输入为$\mathbf{x}_t, t=1, \ldots, T$，隐含层变量

$$\mathbf{h}_t = f(\mathbf{x}_t, \mathbf{h}_{t-1}) $$

编码器的背景向量

$$\mathbf{c} =  q(\mathbf{h}_1, \ldots, \mathbf{h}_T)$$

一个简单的背景向量是该网络最终时刻的隐含层变量$\mathbf{h}_T$。
我们将这里的循环神经网络叫做编码器。

#### 双向循环神经网络

编码器的输入既可以是正向传递，也可以是反向传递。如果输入序列是$\mathbf{x}_1, \mathbf{x}_2, \ldots, \mathbf{x}_T$，在正向传递中，最早到最终时刻的输入分别是$\mathbf{x}_1, \mathbf{x}_2, \ldots, \mathbf{x}_T$，隐含层变量

$$\overrightarrow{\mathbf{h}}_t = f(\mathbf{x}_t, \overrightarrow{\mathbf{h}}_{t-1}) $$


而反向传递中，最早到最终时刻的输入分别是$\mathbf{x}_T, \mathbf{x}_{T-1}, \ldots, \mathbf{x}_1$，而隐含层变量的计算变为

$$\overleftarrow{\mathbf{h}}_t = f(\mathbf{x}_t, \overleftarrow{\mathbf{h}}_{t+1}) $$




当我们希望编码器的输入既包含正向传递信息又包含反向传递信息时，我们可以使用双向循环神经网络。例如，给定输入序列$\mathbf{x}_1, \mathbf{x}_2, \ldots, \mathbf{x}_T$，按正向传递，它们在循环神经网络的隐含层变量分别是$\overrightarrow{\mathbf{h}}_1, \overrightarrow{\mathbf{h}}_2, \ldots, \overrightarrow{\mathbf{h}}_T$；按反向传递，它们在循环神经网络的隐含层变量分别是$\overleftarrow{\mathbf{h}}_1, \overleftarrow{\mathbf{h}}_2, \ldots, \overleftarrow{\mathbf{h}}_T$。在双向循环神经网络中，时刻$i$的隐含层变量可以把$\overrightarrow{\mathbf{h}}_i$和$\overleftarrow{\mathbf{h}}_i$连结起来。

```{.python .input}
# 连结两个向量。
from mxnet import nd
h_forward = nd.array([1, 2])
h_backward = nd.array([3, 4])
h_bi = nd.concat(h_forward, h_backward, dim=0)
h_bi
```

### 解码器

编码器最终输出了一个背景向量$\mathbf{c}$，该背景向量编码了输入序列$\mathbf{x}_1, \mathbf{x}_2, \ldots, \mathbf{x}_T$的信息。

假设训练数据中的输出序列是$\mathbf{y}_1, \mathbf{y}_2, \ldots, \mathbf{y}_{T^\prime}$，我们希望表示每个$t$时刻输出的既取决于之前的输出又取决于背景向量。之后，我们就可以最大化输出序列的联合概率

$$\mathbb{P}(\mathbf{y}_1, \ldots, \mathbf{y}_{T^\prime}) = \prod_{t^\prime=1}^{T^\prime} \mathbb{P}(\mathbf{y}_{t^\prime} \mid \mathbf{y}_1, \ldots, \mathbf{y}_{t^\prime-1}, \mathbf{c})$$


并得到该输出序列的损失函数

$$- \log\mathbb{P}(\mathbf{y}_1, \ldots, \mathbf{y}_{T^\prime})$$

为此，我们使用另一个循环神经网络作为解码器。解码器使用函数$p$来表示单个输出$\mathbf{y}_{t^\prime}$的概率

$$\mathbb{P}(\mathbf{y}_{t^\prime} \mid \mathbf{y}_1, \ldots, \mathbf{y}_{t^\prime-1}, \mathbf{c}) = p(\mathbf{y}_{t^\prime-1}, \mathbf{s}_{t^\prime}, \mathbf{c})$$

其中的$\mathbf{s}_t$为$t^\prime$时刻的解码器的隐含层变量。该隐含层变量

$$\mathbf{s}_{t^\prime} = g(\mathbf{y}_{t^\prime-1}, \mathbf{c}, \mathbf{s}_{t^\prime-1})$$

其中函数$g$是循环神经网络单元。


## 注意力机制

在以上的解码器设计中，各个时刻使用了相同的背景向量。如果解码器的不同时刻可以使用不同的背景向量呢？

以英语-法语翻译为例，给定一对输入序列“they are watching”和输出序列“Ils regardent”，解码器在时刻1可以使用更多编码了“they are”信息的背景向量来生成“Ils”，而在时刻2可以使用更多编码了“watching”信息的背景向量来生成“regardent”。这看上去就像是在解码器的每一时刻对输入序列中各个部分分配不同的注意力。这也是注意力机制的由来。它最早[由Bahanau等在2015年提出](https://arxiv.org/abs/1409.0473)。

现在，对上面的解码器稍作修改。我们假设时刻$t^\prime$的背景向量为$\mathbf{c}_{t^\prime}$。那么解码器在$t^\prime$时刻的隐含层变量

$$\mathbf{s}_{t^\prime} = g(\mathbf{y}_{t^\prime-1}, \mathbf{c}_{t^\prime}, \mathbf{s}_{t^\prime-1})$$


令编码器在$t$时刻的隐含变量为$\mathbf{h}_t$，解码器在$t^\prime$时刻的背景向量为

$$\mathbf{c}_{t^\prime} = \sum_{t=1}^T \alpha_{t^\prime t} \mathbf{h}_t$$


也就是说，给定解码器的当前时刻$t^\prime$，我们需要对解码器中不同时刻的隐含层变量求加权平均。而权值也称注意力权重。它的计算公式是

$$\alpha_{t^\prime t} = \frac{\exp(e_{t^\prime t})}{ \sum_{k=1}^T \exp(e_{t^\prime k}) } $$

而$e_{t^\prime t} \in \mathbb{R}$的计算为：

$$e_{t^\prime t} = a(\mathbf{s}_{t^\prime - 1}, \mathbf{h}_t)$$

其中函数$a$的设计有多种方法。在[Bahanau的论文](https://arxiv.org/abs/1409.0473)中，

$$e_{t^\prime t} = \mathbf{v}^\top \tanh(\mathbf{W}_s \mathbf{s}_{t^\prime - 1} + \mathbf{W}_h \mathbf{h}_t)$$

其中的$\mathbf{v}, \mathbf{W}_s, \mathbf{W}_h$是需要学习的模型参数。


## 结论

* 编码器-解码器（seq2seq）的输入和输出可以都是不定长序列。
* 在解码器上应用注意力机制可以在解码器的每个时刻使用不同的背景向量。每个背景向量相当于对输入序列的不同部分分配了不同的注意力。


## 练习

* 了解其他的注意力机制设计。例如论文[Effective Approaches to Attention-based Neural Machine Translation](https://nlp.stanford.edu/pubs/emnlp15_attn.pdf)。

* 除了机器翻译，你还能想到seq2seq的哪些应用？

**吐槽和讨论欢迎点**[这里](https://discuss.gluon.ai/t/topic/4523)