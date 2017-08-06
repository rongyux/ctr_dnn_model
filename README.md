# 点击率预估

以下是本例目录包含的文件以及对应说明:

```
├── README.md               # 本教程markdown 文档
├── dataset.md              # 数据集处理教程
├── infer.py                # 预测脚本
├── network_conf.py         # 模型网络配置
├── reader.py               # data reader
├── train.py                # 训练脚本
└── utils.py                # helper functions
└── avazu_data_processer.py # 示例数据预处理脚本
```

## 背景介绍

CTR(Click-Through Rate，点击率预估)
是对用户点击一个特定链接的概率做出预测，是广告投放过程中的一个重要环节。精准的点击率预估对在线广告系统收益最大化具有重要意义。

当有多个广告位时，CTR 预估一般会作为排序的基准，比如在搜索引擎的广告系统里，当用户输入一个带商业价值的搜索词（query）时，系统大体上会执行下列步骤来展示广告：

1.  获取与用户搜索词相关的广告集合
2.  业务规则和相关性过滤
3.  根据拍卖机制和 CTR 排序
4.  展出广告

### 输入／输出

训练：python train.py

预估：python infer.py

输入样本：

id,click,hour,C1,banner_pos,site_id,site_domain,site_category,app_id,app_domain,app_category,device_id,device_ip,device_model,device_type,device_conn_type,C14,C15,C16,C17,C18,C19,C20,C21

1000009418151094273,0,14102100,1005,0,1fbe01fe,f3845767,28905ebd,ecad2386,7801e8d9,07d7df22,a99f214a,ddd2926e,44956a24,1,2,15706,320,50,1722,0,35,-1,79
10000169349117863715,0,14102100,1005,0,1fbe01fe,f3845767,28905ebd,ecad2386,7801e8d9,07d7df22,a99f214a,96809ac8,711ee120,1,0,15704,320,50,1722,0,35,100084,79

### LR vs DNN

下图展示了 LR 和一个 \(3x2\) 的 DNN 模型的结构：

<p align="center">
<img src="images/lr_vs_dnn.jpg" width="620" hspace='10'/> <br/>
Figure 1. LR 和 DNN 模型结构对比
</p>

LR 的蓝色箭头部分可以直接类比到 DNN 中对应的结构，可以看到 LR 和 DNN 有一些共通之处（比如权重累加），
但前者的模型复杂度在相同输入维度下比后者可能低很多（从某方面讲，模型越复杂，越有潜力学习到更复杂的信息）；
如果 LR 要达到匹敌 DNN 的学习能力，必须增加输入的维度，也就是增加特征的数量，
这也就是为何 LR 和大规模的特征工程必须绑定在一起的原因。

LR 对于 DNN 模型的优势是对大规模稀疏特征的容纳能力，包括内存和计算量等方面，工业界都有非常成熟的优化方法；
而 DNN 模型具有自己学习新特征的能力，一定程度上能够提升特征使用的效率，
这使得 DNN 模型在同样规模特征的情况下，更有可能达到更好的学习效果。

本文后面的章节会演示如何使用 PaddlePaddle 编写一个结合两者优点的模型。


## Wide & Deep Learning Model

谷歌在 16 年提出了 Wide & Deep Learning 的模型框架，用于融合适合学习抽象特征的 DNN 和 适用于大规模稀疏特征的 LR 两种模型的优点。


### 模型简介

Wide & Deep Learning Model\[[3](#参考文献)\] 可以作为一种相对成熟的模型框架使用，
在 CTR 预估的任务中工业界也有一定的应用，因此本文将演示使用此模型来完成 CTR 预估的任务。

模型结构如下：

<p align="center">
<img src="images/wide_deep.png" width="820" hspace='10'/> <br/>
Figure 2. Wide & Deep Model
</p>

模型左边的 Wide 部分，可以容纳大规模系数特征，并且对一些特定的信息（比如 ID）有一定的记忆能力；
而模型右边的 Deep 部分，能够学习特征间的隐含关系，在相同数量的特征下有更好的学习和推导能力。
