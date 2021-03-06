---
layout: post
title: 语音识别中的 CTC 算法详解
author: Phree
date: 2018-3-22
tag: [语音识别, 算法]
---

## CTC(Connectionist Temporal Classiﬁcation)
在神经网络训练中用于训练序列化的数据, 比如说语音, 手写文字(虽然手写文字是一张图, 但不同的文字之间可以认为是按一定序列进行输入的, 因为要考虑前后的约束关系). 在数据量剧增的今天, 我们可以非常容易地获取大量语音数据, 但遗憾的是这些语音通常没有非常仔细的标注. 如何李用深度学习和大量数据的优势是语音识别领域的一个难题. CTC 的提出为使用深度神经网络进行语音识别奠定了基础.
### 介绍
在语音识别领域存在的一个问题是, 我们的训练数据往往只是语音片段和对应的文本, 语音序列和文本长度不同, 并且不是对齐的, 也就是, 我们不知道语音中的哪一帧是属于一个字, 也不知道一个字的长短. 这是因为进行对齐标注是一件非常耗费人力和时间的事情. 并且人与人的语速不同, 导致不可能使用一种强制的规则来强制确定一个字的长短. 这种难以对齐的问题也出现在手写字体识别中, 因为每个字占的大小也是不确定的.
CTC 正是为了解决这个问题而被提出的. CTC 用于把语音经过处理以后得到的特征映射为文本, 而对特征提取的过程并没有限制, 也即是说可以使用任何模型接到 CTC 中得到结果.

![alignment]({{ site.baseurl }}/assets/img/ctc/alignment.svg)

#### 形式化问题
CTC 网络将输入划分成固定长度的帧. 我们的目标是对给定符号输出符号集(如汉字表, 字母表) 

$$ \boldsymbol{L^*} $$ 

和训练数据集, $$ \mathcal{D} $$, 训练集中每一个样本 $$ \mathbf{(x,z)} $$包含一个由T个向量组成的 $$ m $$ 维语音序列

$$
\mathbf{x}=\{x_1, x_2,...,x_T\}
$$

和输出文本

$$
\mathbf{z}=\{z_1,z_2,...,z_U\}
$$

希望找到映射 
$$
\mathbf{z}=h(\mathbf{x})
$$ .
事实上这个映射并不容易直接得到, CTC 通过一个巧妙的算法来得到这个映射.
CTC 网络先将输入的 $$ m $$ 维长度为 $$ T $$ 向量序列 $$ \mathbf{x} $$ 映射为输出的 $$ n $$ 维长度为$$T$$的向量序列 $$ \mathbf{y} $$. $$ \mathbf{y}=\{\mathbf{y^1, y^2, ..., y^T}\} $$,

$$
\mathbf{y^t}=\{y^t_1, y^t_2, ..., y^t_m\} 
$$

其中 $$ y^t_k $$ 表示$$t$$时刻网络的第 $$ k $$ 个输出, 通常解释为输出低 $$ k $$个符号的概率.
这个过程表示为
$$
\mathscr{N}:(\mathbb{R}^m)^T\mapsto (\mathbb{R}^n)^T 
$$. 
通常由一个 RNN 网络来完成, 如上图所示的 Alignment.
如此我们可以看到, 一个输出 $$ \mathbf{y} $$, 可以对应一条路径, 我们将其表示为 $$ \pi $$. 每一条路径的概率为

$$ p(\pi|\mathbf{x})=\prod _{t=1}^Ty^t_{\pi_t}, \forall \pi \in L^T $$

即一条路径的概率等于路径上各个符号输出概率的乘积(事实上, 这里是假设输出符号之间相互独立).
随后 CTC 需要对于连续相同的输出进行合并. 但是这种方法会出现一些问题, 比如说对于单词 hello, 输出的$$\mathbf{y}$$可能是 heelllooo, 进行合并以后会得到 helo 而不是 hello. 为了解决这个问题, CTC 中引入了一个特殊的 < blank > 字符, 最后这个字符是会被移去的. 我们期望它输出 heel< blank >loo, 最后就可以得到所期望的 hello. 使用< blank >还解决了语音中的停顿的问题, 同一段时间没有输出时, 也输出< blank >字符. 这时符号集$$L'=L\cup <blank>$$我们将这个合并和删除< blank >的过程记为$$\mathcal{B}:L^{'T}\mapsto L^{\le T}$$, 将这个过程的输出记为$$l$$, 有$$\mathbf{l}=\mathcal{B}(\pi)$$显然这是一个多对一映射, 即可能有多个可能的路径对应一种可能的文本输出.
实际上, 我们所希望得到的就是

$$
h(\mathbf{x})=\arg \max_{l\in L^{\le T}}p(\mathbf{l|x})
$$

其中

$$
p(\mathbf{l}|\mathbf{x})=\sum_{\pi}p(\pi|x), \pi \in \{\pi|\mathcal{B}(\pi)=\mathbf{l}\}
$$

即在所有可能的输出文本序列中找到概率最大的那一个, 而每一个输出文本序列的概率可以通过边沿概率分布公式得到, 即所有可能得到该输出文本的路径的概率之和.

### Loss Function
事实上直接优化得到上式也不是一件容易的事情. 因为这意味着我们需要遍历所有的路径来得到各个可能输出文本的概率. CTC 论文中提到两种方法可以简化这个步骤.
第一种方法比较简单, 我们可以简单地认为

$$
h(\mathbf{x})\approx \mathcal{B}(\pi^*)
$$

其中,

$$
\pi^*=\arg \max _{\pi\in N^t}p(\pi|\mathbf{x})
$$

直观上理解就是直接找概率最大的路径, 认为其对应的输出一般也是概率最大的. 这样我们其实只需要选择每个$$t$$中出现概率最大的符号. 可以解释为希望找一个成绩最好的班级, 但查看每一个学生的成绩又太过于繁琐, 于是只看成绩最好的学生的成绩.
从上面的例子可以看出, 这种方法事实上过于简化了, 很多情况下我们可能会遇到这种情况. 于是论文中还提出了前缀搜索解码(Prefix Search Decoding), 这种方法的结果普遍好于第一种方法.
事实上如果我们考量所有可能的路径, 我们会发现其实有很多路径是重复计算的, 有很多路径也是明显不可能的, 这启发我们使用动态规划和剪枝来去除冗余计算和淘汰不可能的路径. Prefix Search Decoding 算法类似于 HMM 模型中的前向-后向算法(F-B Algorithm).
我们首先定义

$$
\alpha_t(x)\equiv \sum_{\mathcal{B}(\pi_{1:t})=\mathbf{1:s}}\prod_{t'=1}^{t}y^{t'}_{\pi_{t'}}
$$

解释为经过$$t$$步, 得到输出文本序列$$\mathbf{l}_{1:s}$$的概率.
从而我们可以得到递推关系
...未完待续...

CTC 假设在给定输入的情况下, 输出是条件独立的(事实上在一些情况下该假设是不成立的, 因此这也是 CTC 的一个主要缺点). 根据条件概率公式, 可以把它分解为. 每一种输出序列都可以看成是一条路径, 可以将其视为计算不同路径概率. 如果我们遍历所有路径计算概率, 计算量将是非常巨大的, 举个例子, 假设输出有 10 种不同的符号, 一个语音序列被分拆成 40 帧, 那么可能的路径将是 $$10^{40}$$! 这是不可接受的. 幸运的是, 经过一些分析我们可以发现这种暴力算法中包含了很多冗余的计算, 熟悉算法的朋友可能已经发现了, 可以通过动态规划来高效地完成这一计算.
对于每一帧, 输出的概率分布 $$P(a_t|X)$$ 通常用 RNN 来实现.
