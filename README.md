# Joint-MRC

[TOC]

## Description

本着简洁、通用的原则，本项目对常规MRC任务做了一个集成，集检索、阅读为一体，在只考虑常规MRC的基础上在[莱斯杯：全国第二届“军事智能机器阅读”挑战赛](<https://www.kesci.com/home/competition/5d142d8cbb14e6002c04e14a>)获得第6名成绩（大赛任务部分问题需要多步推理）。

**模型特性**：

* 同时集成检索、阅读两个任务，联合优化学习；
* 同时适合常规MRC与大范围检索场景，包括长篇章与多篇章检索；
* 对分段算法采用动态规划做优化，保证最小化答案信息丢失风险；
* 可以方便地扩展为多阶段检索，动态地适应不同的篇章数量、长度。

 

> 虽然本项目用在了比赛上，但是笔者并没有对比赛数据集本身做过多的探索（除了预处理标注和多问题拆分），而是希望做一个通用的检索阅读框架，希望能够处理大范围的阅读检索，只是刚好也能在这个比赛数据集上取得比较好的成绩。
>

  

## Requirements

本项目运行在Python3.6下，3.7以上是否能运行没有尝试过。

```
tensorflow_gpu==1.12.0
keras==2.2.4
```

 

> 项目中的[transformer_contrib](https://github.com/caishiqing/transformer_contrib)包是笔者集成了[CyberZHG](https://github.com/CyberZHG)实现的transformer系列模型，为了减小依赖并提高集成度干脆做了一个封装，其中更改了作者 keras_bert.tokenizer.rematch 的源码，源码的实现是基于编辑距离来匹配每一个最优子片段，复杂度为O(L^2)随文章长度几何倍增长，效率非常低，优化后的实现利用了BERT分词的规则给逆推回去，在比赛数据的预处理上是源码速度的100倍。

 

## Framework

对于检索阅读而言，任务的主要难点有以下几个：

* 如何合并两个任务，尤其是两个任务的熵相差巨大的情况下，如何调和两个任务的loss；
* 在篇章长度不确定的情况下，如何设计通用的分段策略以降低总体风险；
* 当检索范围特别大时，例如有多篇章且篇章长度都很长，如何处理大量的负样本。

 

### 分段算法

由于MRC任务很多篇章都会超过BERT的最大长度，因此绝大多数情况下需要做截断。但是简单的截断会有一定风险，可能会把答案截断，又或者答案处于片段的边缘，这样会缺乏上下文信息。那么要怎么截断合适呢？

考虑以下情形：假设最大长度设为500，有一段篇章长度600，这时候要如何截断？常规情况下肯定就在长度500处截断呗，那么问题来了，剩下的长度100的片段如何处理？直接作为第二段吗？那也太挫了，很可能答案就被截断了（如果在500附近）。那么我们不难想到一个策略，就是允许一定的交叉，即0-500为第一段，100-600为第二段，只要答案不是特别长，不管是在第二段的边界上还是在第二段之外，都会落在第一段，反之一样一定落在第二段，再或许正好在中间，那么两段都包含答案，如果是这样那再好不过，最终提取到答案的机会会更大。

上面描述的情形可能比较简单，用规则就可以解决了，那如果长度是1100呢？500处截断，然后500-1000截断，又剩下100咋处理？可能有人立马联想到可以允许交叉啊，第三段从600到1100呗，那么第二第三段有很大的交叉自然没问题，但第一第二段之间的答案很有可能被截断或者丢失上下文信息，总体风险还是很高。那么最好的解决方案是把第二段往前移动一些，那么问题来了，移动多少合适呢？如果移动得太多，第二第三段之间就可能会断开，那就得需要第四段，段落就会变得过于冗余。那你一定会想到，刚好移动到第一第二段之间的交叉和第二第三段之间的交叉相等就可以了呗。没错，这样是最好的分段设计，而且操作也很容易实现，但是问题是我们不能直接完全按照固定窗口大小来截取啊，这样很有可能把句子截断。

一个合理的先验假设是答案一般不会跨句，也就是要么在一个句子内部，要么包含一个或多个完整的句子。基于这样的假设我们需要以句子为单位做规划，我们的目标是在保证覆盖全文以及段落长度限制条件的约束下，使分段结果具有最小的冗余度并且最小化丢失答案信息的风险（使答案至少落在一个段落中，而且落在越中间的位置越好，避免在段落边缘丢失上下文信息）。当篇章的长度继续增加，需要更多的段落来截断时，组合的空间也会爆炸式增长，这时候就轮到我们的动态规划算法出场了。

<p align = 'center'>
<img src = 'images/cut_bp.jpg' height = '200px'>
</p>

我们通过构建一个有向无环图来完成整个规划，如上图所示，上层是对篇章使用标点符号分割后的句子序列，我们规定最长子段落是满足最大长度限制条件下的最长连续句子子序列，即再添加下一句就会超出最大长度，如图中所示的Para1{Sent1, Sent2, Sent3}，我们以所有合法的子段落作为节点，按顺序将具有交叉或相连的子段落之间用有向边连接，边的权重为两个段落之间的交叉度

<p align = 'center'><a href="https://www.codecogs.com/eqnedit.php?latex=w_{ij}=c_{ij}^2" target="_blank"><img src="https://latex.codecogs.com/gif.latex?w_{ij}=c_{ij}^2" title="w_{ij}=c_{ij}^2" /></a></p>

其中c_ij为两个段落之间的交叉文本长度，我们需要规划一条连接首尾节点的路径以最小化目标

<p align = 'center'><a href="https://www.codecogs.com/eqnedit.php?latex=w_{ij}=c_{ij}^2" target="_blank"><img src="https://latex.codecogs.com/gif.latex?F_{1->M}=arg\,\min_{S} \sum_{(i->j)\subset S}w_{ij}=\quad arg\,\min_{S}\sum_{(i->j)\subset S}||c_{ij}||_2^2 ,\forall (i->j)\subset S" /></a></p>

之所以用二范式的度量方式，是因为路径总权重的二范式既能度量总体冗余度的大小，又能控制不同段落之间的交叉均衡（降低丢失答案信息的风险）。

我们可以递归地定义子问题的解

<p align = 'center'><a href="https://www.codecogs.com/eqnedit.php?latex=w_{ij}=c_{ij}^2" target="_blank"><img src="https://latex.codecogs.com/gif.latex?F_{i->j}=F_{i->j-1}+w_{j-1,j}" /></a></p>

然后使用动态规划求解原问题的解。分段算法的代码定义在utils.data_utils.split_text中，有兴趣的可以仔细研读。以下是分段算法的意识示例

```python
text = "今夕何夕兮，搴舟中流。今日何日兮，得与王子同舟。蒙羞被好兮，不訾诟耻。心几烦而不绝兮，得知王子。山有木兮木有枝，心悦君兮君不知。"
sub_texts, starts = split_text(text, maxlen=30, greedy=False)
"""
sub_texts = [
	"今夕何夕兮，搴舟中流。今日何日兮，得与王子同舟。",
	"今日何日兮，得与王子同舟。蒙羞被好兮，不訾诟耻。",
	"羞被好兮，不訾诟耻。心几烦而不绝兮，得知王子。",
	"心几烦而不绝兮，得知王子。山有木兮木有枝，心悦君兮君不知。"
]
"""
```

 

> 这个算法对于特别长的篇章，如比赛中的篇章作用不是很明显，但是对于中等长度的篇章作用比较明显。在初赛训练集的统计上，本算法的答案丢失率为0.

 

### 多任务调和

为什么要用多任务学习：只用一个模型完成多项任务，简洁爽快。而且笔者认为问题的答案是否存在于篇章中，与在篇章中具体什么位置，这两个任务在直觉上是有关联的，可以相互促进。

然而本项目要解决的另一个难点就是怎么调和多个任务的学习过程，相信有很多人在使用多任务学习时会遇到不同任务之间的loss很难调和，有的收敛很快，有的收敛很慢甚至不收敛。在这里笔者会分享自己的经验，并且会结合测试结果给出解释。

在解释如何调和多任务的loss之前，我们还有一个问题要解决，那就是怎么合并检索与阅读两个任务，毕竟检索任务同时依赖于正负样本，而阅读任务只依赖于正样本。其实只有一个很简单的trick，就是把负样本序列的标签全部置0，相当于副样本不会产生阅读任务的loss，这个操作不会对检索任务产生影响，只是会稍微减慢阅读任务的收敛速度，因为每个batch的loss是在样本上做平均的，而阅读任务的loss只来源于正样本却要对样本总量做平均，相当于在正常的情况下乘了一个小于0的系数，你可以理解为阅读任务的loss权重减小了一些。另外本项目为了保障训练稳定，在每轮迭代过程中使用了均衡采样，保证每个batch中都有一半的正例和一半的负例（所以loss和acc都会比正常水平低一半，不用感觉奇怪），代码详见utils.train_utils.generate_data.

让我们回到多任务调和问题上，当一个二分类任务与一个多分类（有可能是很多类别，比如阅读任务的指针网络，假设平均长度为500）合并时，明显两个任务的难度不一样，二分类的熵为log2=0.69，所以你在刚开始训练的时候基本上会发现初始loss是0.6+，而多分类的熵log500=6.21，比二分类不确定性要大得多，从直觉上来讲可以说多任务更难训练，或者需要更多的时间、更多的step才能收敛，常规的训练曲线如下图所示

<p align = 'center'>
<img src = 'images/train1.png' height = '600px'>
</p>

可以很明显地看出检索任务收敛得非常快，而阅读任务还没有完全收敛，检索完全带不动阅读，这样子导致我们选择的模型的多个输出有的已经过拟合了，有的还是欠拟合，效果自然很差。那最直观的想法就是限制检索任务的收敛速度，而加快阅读任务的收敛速度，这可以通过更改多个任务的loss权重来实现。我们可以极端地把nsp的loss权重直接置0，那么nsp就不收敛，从原来很陡的下降曲线变为了一条平线，这自然太过于极端了，我们要的是将nsp的loss权重设置为一个较小的值，相比于阅读任务的权重比例要低得多，这样nsp的收敛曲线就会平缓一些（慢一些），我们将[nsp, start, end]三个任务的权重设置为[0.1, 0.4, 0.4]，会得到下面的收敛曲线

<p align = 'center'>
<img src = 'images/train2.png' height = '600px'>
</p>

可以发现收敛速度的差异小了不少，nsp任务的收敛拐点后移了，而start和end任务的收敛点前移了，但是nsp任务的收敛速度相对还是更快一点，我们希望的是多个任务的收敛速度一致，这样我们选择的模型对于每一个子任务都是最优的。我们继续调整loss权重，这一次设置为[0.04, 0.48, 0.48]，然后得到了下面的曲线

<p align = 'center'>
<img src = 'images/train3.png' height = '600px'>
</p>

此时的收敛速度基本一致了。对于多任务学习而言，loss权重是一个很重要的超参数，那么到底要如何设置loss权重呢？首先从经验上和测试结果来说，任务越复杂、熵越大的输出对应的loss权重需要更高，具体可以参考熵的比例，比如在本例中检索和问答任务熵的比例大致为1:10，那么可以以这个比例作为loss权重的参考进行微调，因为即使同样是两个二分类任务，也会因为任务难度的不同在收敛速度上有一些差距。

 

### 多阶段检索

考虑如果是检索范围特别大的情形，比如本次比赛每个问题有5篇候选文档，每篇文档的平均长度达到8000多字，那么即便使用均衡采样，当模型收敛时也只能学习到极少部分的负样本，即使用集成也无法遍历所有的负样本（在初赛笔者就发现集成的效果甚微，主要就是检索精度限制了集成效果，虽然初赛单模单阶段已经能进入前十）。

另一方面，笔者发现虽然检索模型的top1精度很难提高，但是top2和top3的覆盖率还是很高的，top3已经达到99+，因此自然想到分阶段检索过滤，第一阶段粗选，第二阶段精选。在实际的第一阶段过滤中本项目并没有直接按照top2或者top3截取，而是根据分数的分布情况动态截取，因为大部分样本top1就已经很明显了，只有部分样本答案的分布比较模糊，需要取topk，甚至有的需要更多候选，所以用固定的topN不是一个高效的方法。具体的方法定义在process._retrieve中。

通过调节参数，可以控制覆盖率与负样本的比例，最终笔者调节到正负比例接近1:1的同时达到99+的覆盖率，因为平衡的比例对于下游的二分类检索任务效果更好。此时本项目多任务学习的一大优势就体现出来，因为模型同时具备了检索与阅读任务，所以第一阶段与第二阶段完全可以使用同一种模型，只是训练的数据分布不同。第一阶段可以通过检索与阅读的概率分数综合排序，也可以直接用检索分数排序，影响并不大。

当第一阶段训练得到一个模型之后，第二阶段只需要对这个模型比较难以区分的样本做二次训练，专注于难分的样本（是不是有点boosting的感觉），同时没有样本不均衡的影响。需要注意的是第二阶段的训练数据和推断数据都需要经过同一个第一阶段模型做过滤，保证第二阶段的输入分布式一致的，具体过程示意如下

<p align = 'center'>
<img src = 'images/两阶段检索阅读.jpg' height = '396px'>
</p>
 


### Ranking

到此为止，我们便完成了模型的所有训练工作，通过推断我们可以得到每个问题在相关的所有段落上的检索分数（阳性概率）与答案分数（边界概率），实际上模型输出的是start和end两个概率分布，我们规定答案分数由下式融合而得

<p align = 'center'><a href="https://www.codecogs.com/eqnedit.php?latex=w_{ij}=c_{ij}^2" target="_blank"><img src="https://latex.codecogs.com/gif.latex?P_{ans}=exp \left(\frac{w_slogP_s+w_elogP_e}{w_s+w_e}\right)" /></a></p>

这样的融合方式比直接求平均最终分数要高出0.3个百分点左右，可能的原因是这种形式提高了结果的置信度，因为log函数对越小的值越敏感，换句话说如果其中有一项概率比较低，会拉低总体分数。w_s和w_e分别代表start和end的权重比例，我们默认为1:1.

我们现在有了检索和阅读两个分数，可以综合这两个分数对答案排序

<p align = 'center'><a href="https://www.codecogs.com/eqnedit.php?latex=w_{ij}=c_{ij}^2" target="_blank"><img src="https://latex.codecogs.com/gif.latex?P=exp \left(\frac{w_{nsp}logP_{nsp}+w_{ans}logP_{ans}}{w_{nsp}+w_{ans}}\right)" /></a></p>

其中w_nsp和w_ans是检索和阅读的分数权重，这是一对超参数（config.fuse_weights），并且当检索范围特别大时结果对这个超参数比较敏感，需要将nsp的权重比例调得很高，比赛中笔者使用的比例为 [0.98, 0.02]，之所以保留ans一部分权重是因为如果全部使用检索分数结果会有所下降，笔者分析可能是有少部分答案特别模糊，检索分数很接近，需要使用阅读分数来消歧。当篇章长度较小、负样本很少时影响并不大，甚至可以直接用阅读分数排序（[0, 1]）。

最终我们会对问题对应的所有篇章所有段落中的答案排序，如果有多篇章一定要设置q_id来关联。

<p align = 'center'>
<img src = 'images/ranking.png' height = '248px'>
</p>
 


## DataSet

每个数据集具有相同的格式，每个数据集包括train.json、valid.json、test.json三个文件，数据格式为json，每行一个样本，包含一段文本和对应的一个或多个问题，结构如下：

```
{
    "qas":[
        {
            "question": "张艺谋电影《英雄》中秦始皇是哪位内地男演员扮演的?",
            "answer": "陈道明",
            "answer_start": 20,
            "answer_end": None,
            "q_id": "12345",
        },
        ...
    ],
    "context":"在表演上，张丰毅表示：不能简单地和先前陈道明在《英雄》里扮演的秦始皇相比。"
}
```

所有数据集已经过清洗和统一格式，其中[WebQA](https://kexue.fm/archives/4338)来自百度，[Les](https://zhpmatrix.github.io/2018/08/27/machine-reading-comprehension/)来自于28所，[CMRC](https://worksheets.codalab.org/worksheets/0x92a80d2fab4b4f79a2b4064f7ddca9ce)来自科大讯飞。

其中"answer_end"和"q_id"为选填字段，"answer_end"可以没有或者为空，但"answer_start"为必填字段；对于训练集或者单文档问题"q_id"可以没有，但是对于多文档测试集需要使用"q_id"关联文档段落与排序。

项目中的数据集需要组织成这种形式，为了避免预处理不必要的麻烦，笔者提供不同的已经处理好的数据集：

* [CMRC](https://worksheets.codalab.org/worksheets/0x92a80d2fab4b4f79a2b4064f7ddca9ce)数据集，提取码：rkpp，[百度网盘](https://pan.baidu.com/s/1W7rSqeLuU2qc37ZdUTP3vw)
* [WebQA](https://kexue.fm/archives/4338)数据集，提取码：t8ed，[百度网盘](https://pan.baidu.com/s/107DBjRQ-waa22JP2O0RyHA)
* Les-simple（[莱斯杯](https://www.kesci.com/home/competition/5d142d8cbb14e6002c04e14a)单答案无推理）数据集，提取码：5tgv，[百度网盘](https://pan.baidu.com/s/1bWFRoV-Qg-fCKVx0STa0ug)
* Les-totle（[莱斯杯](https://www.kesci.com/home/competition/5d142d8cbb14e6002c04e14a)完整数据集），提取码：irp0，[百度网盘](https://pan.baidu.com/s/1FK-LJYFugJ2IAJRaYKqh2Q)

 

## Quick Start

### Train

项目在joint_mrc目录下，可以直接下载后将joint_mrc作为工作区，也可以使用git工具

```
git clone https://github.com/caishiqing/joint-mrc.git
cd joint-mrc/joint_mrc
```

./datasets目录存放数据集，可以将多份不同的数据集（按照上一节的结构组织）一起放到这个目录下，预处理程序会同时加载和混合所有数据集，也可以只放一个数据集；./models目录默认存放所有模型，包括预训练参数与微调后的模型参数，预训练模型可以是bert、bert-wwm、roberta等，只要符合tensorflow版本的bert文件格式；config.py定义了所有的配置参数，具体见脚本中的注释。

将所有的数据和模型文件有放置妥当之后，配置好相关的参数，先对数据集预处理生成输入数据：

```shell
python process.py
```

此时默认在./datasets目录下生成process.pkl文件，然后运行训练命令：

```shell
python main.py --action train --gpu 0
```

如果需要使用多GPU并行训练，直接运行

```shell
python main.py --action train --gpu 0,1,2,3
```

可以用上一节Les-simple数据集作测试。需要集成的话运行

```shell
python main.py --action train --gpu 0,1,2,3 --ensemble --k_fold 5
```

这样会通过重新划分数据集训练5个模型（时间有点长）。

而遇到像本次比赛这样检索范围非常大的情况，需要做两阶段检索过滤的话，只需将上述训练完的模型（单模）放入./models/retrieve目录下（或者你自己制定路径），然后将上述命令重新运行一遍（注意包括运行process.py，此时会重新生成一个过滤后的训练数据文件processed.pkl）。可以用上一节Les-totle数据集复现我队的结果。

 

> 多GPU并行有一个问题，就是第一块GPU既要存放参数和汇总梯度操作，又要执行前向后向计算，显存很容易爆掉，而其他的GPU只需要做前向后向运算，显存都用不完，导致batch设不大，总体效率很低，可以通过将config.first_gpu设置为False来优化，即第一块GPU只做梯度汇总使用，这样可以在增加很多块GPU卡的情况下也不会爆。另外注意config.batch_size指的是每一块GPU的batch_size大小。

 

### Test

执行验证测试命令

```shell
python main.py --action eval --gpu 0
```

默认验证./datasets目录下所有数据集的valid.json数据，默认使用./models目录下的第一个hdf5模型文件，也可以指定验证文件和模型文件

```shell
python main.py --action eval --gpu 0 --file_path ./datasets/Les/test.json --model_path ./models/xxxxx.hdf5
```

加载和测试模型的示例在test.py 中

```python
from utils.io_utils import load_mrc
from model import MRC
import tensorflow as tf
from config import config

if __name__ == '__main__':
    import os
    os.environ['CUDA_VISIBLE_DEVICES'] = '0'
    gpu_options = tf.GPUOptions(allow_growth=True)
    sess = tf.Session(config=tf.ConfigProto(gpu_options=gpu_options))
        
    model = load_mrc(config.model_dir)
    try:
        retrieve = load_mrc(config.retrieve_dir)
    except:
        retrieve = None
    # load model
    mrc = MRC(config, model, retrieve)
    
    q = '姚明有多高？'
    c1 = '姚明身高226cm，被称为小巨人。'
    c2 = '姚明的父亲叫姚志源，身高208cm'
    params = [
        {'q_id': '1', 'question': q, 'context': c1},
        {'q_id': '1', 'question': q, 'context': c2},
        ]
    
    result = mrc.infer(params)
    print(result)
```

输出结果：

```
{'1': {'answer': '226cm', 'score': 0.9794161463725082}}
```

 

> 本项目对长篇章、多篇章效果比较好，对于CMRC或WebQA这样的常规MRC数据集效果不明显，但是框架也是适用的，对于这些篇章较短、负样本特别少的数据集，建议直接把检索的权重（包括loss_weights和fuse_weights）都设置成0，直接用阅读子任务也能获得较好的结果，当然也可以联合学习，但是效果相差不大。同时检索范围越大、负样本越多，检索分数的fuse_weight需要越高。

 

## Experiment

| 模型            | Les-simple | Les-totle | CMRC | WebQA |
| --------------- | ---------- | --------- | ---- | ----- |
| bert            |            |           |      |       |
| bert-wwm-ext    |            |           |      |       |
| roberta-wwm-ext | 0.8544     |           |      |       |

同样是Rouge-L，但是评估函数细节可能与比赛官方不太一样，所以会有一些出入。其他的结果之后慢慢补上~

 

> 从决赛环节可以看出来，本算法在第一轮常规MRC（单答案无推理）任务上排第3，在模型集成没有满状态的情况下（磁盘还有很多空余）表现已经非常出色了，即本算法把基础任务做到了极致，而在后两轮多答案与推理题中都是排第6，虽说不是很好，但是也能看出本算法具有不错的稳定性。