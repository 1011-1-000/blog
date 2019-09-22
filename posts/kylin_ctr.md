---
layout: post
title: "Kylin在CTR计算中的实践"
date: 2019-09-21 22:20:15 +0800
comments: true
mathjax: true
categories: architecture
tags: [kylin,financial]

---

> 本文根据FIC对亿级数据的查询优化的经验整理，通过Kylin，我们将90%数据透视运算的查询性能控制在秒级以内，而对复杂的计算也可以在3s内给出响应。
>
> 阅读本文，需对Kylin的相应概念有了解。

<!--more-->

#### 背景

金融里关于Contribution to return 的计算是一个复杂的过程，但在业务分析端却需要从各个维度有针对性的对投资者所持有的资产组合做一个全面的剖析，包括从行业，区域，时间等。其计算公式如下：

$$
CTR_i = \frac{\sum_{t=t_s}^{t_e}{\frac{v_{i,t}}{V_t}\ln{R_t}}}{\frac{\sum_{t}\ln{R_t}}{\prod_{t}{(1 + R_t)} - 1}}
$$

- $t_s$, $t_e$ 为选中的时间区间
- $v_{i,t}$ 为某支股票在某个时间点的Value add
- $V_t$ 为整个资产组合在某个时间点的Value add
- $R_t$ 为整个资产组合在某个时间点的简单收益
- $\ln{R_t}$为整个资产组合在某个时间点的对数收益
- $CTR_i$ 为某支股票在 $t_s$ ,  $t_e$ 时间区间内给整个资产组合所带来收益的贡献度。

#### 数据

初期为了快速实验Kylin的查询性能,从股票的角度分析每支股票对整个资产组合的贡献度，我们依据市场数据构建资产组合13支，整个交易数据约3000万条,大小为7G左右。表名portfolio_transactions,主要字段如下：

|    字段名    |        基数        | 数据类型 |
| :----------: | :----------------: | :------: |
| PORTFOLIO_ID |         13         | Integer  |
|   DATE_KEY   | 5028(20年历史数据) |   Date   |
|   FIC_CODE   |        3626        |  String  |
|  VALUE_ADD   |      21605536      |  Double  |
|     BMV      |      24086807      |  Double  |
|     EMV      |      26886652      |  Double  |

#### 解决过程

分析上述公式，计算过程大致分为两步：

1. 按时间分组分别计算出每天的$R_t$, $V_t$, $\frac{\sum_{t}\ln{R_t}}{\prod_{t}{(1 + R_t)} - 1}$
2. 再将取出的数据按每支股票分组，计算在整个时间区间里的 $CTR_i$

##### 初次使用

Kylin目前的计算只针对单列，且只支持如下几种计算：

- SUM
- MIN
- MAX
- COUNT
- COUNT_DISTINCT
- TOP_N
- EXTENDED_COLUMN
- PERCENTILE

所以我们只能构建以PORTFOLIO_ID, DATE_KEY, FIC_CODE为维度，以VALUE_ADD, BMV, EMV为度量的Cube：

| Cube名称               | 维度                             | 度量                               | Cube大小 |
| ---------------------- | -------------------------------- | ---------------------------------- | -------- |
| Portfolio_transactions | PORTFOLIO_ID, DATE_KEY, FIC_CODE | SUM(VALUE_ADD), SUM(BMV), SUM(EMV) |          |

再利用已经SUM的指标进一步计算$R_t$, $V_t$,利用该方式，缩减了计算分子中$\ln{R_t}$, $V_t$以及分母中$\frac{\sum_{t}\ln{R_t}}{\prod_{t}{(1 + R_t)} - 1}$的时间。我们将使用Cube与原始计算做了对比：

| 资产组合 | 时间区间                | 原始计算性能 | Cube  |
| -------- | ----------------------- | ------------ | ----- |
| 1006     | 2008-01-01 ~ 2009-01-01 | 4.6 s        | 4.1s  |
| 1006     | 2008-01-01 ~ 2009-01-01 | 7 s          | 6.2 s |
| 1006     | 2008-01-01 ~ 2011-01-01 | 10 s         | 7.3 s |

从上面可以看出来，随着时间区间的增大，Cube的性能与原始计算的方式的差距在逐渐增大。但该阶段我们的查询仍然是一个比较耗时的操作。

##### 回到业务

上面的耗时操作仍然集中在第二步，为了进一步提高性能，我们从业务角度对整个数据集做了调整，额外增加一列将每支股票当天对整个资产组合的$\frac{v_{i,t}}{V_t}\ln{R_t}$部分先提前算出，之后利用Kylin，将用户选择时间段的数据聚合。大幅减少第二步的所用的时间。为此我们构建了2个Cube:

| Cube  | 维度                             | 度量                                          | Cube 大小 |
| ----- | -------------------------------- | --------------------------------------------- | --------- |
| Cube1 | PORTFOLIO_ID, DATE_KEY, FIC_CODE | VALUE_ADD, BMV, EMV                           |           |
| Cube2 | PORTFOLIO_ID, DATE_KEY, FIC_CODE | $\frac{v_{i,t}}{V_t}\ln{R_t}$ - CTR numerator | 455MB     |

这样的调整之后，整个计算耗时被压缩到**300ms**左右。

#### 压力测试

在后面的测试中，我们将数据集扩大5倍增加到1.6亿条记录,并添加一张维度表Equity_Basic_
Info, 主要字段如下：

| 字段                   | 基数 | 类型   |
| ---------------------- | ---- | ------ |
| SECTOR_CLASSIFICATION1 |      | STRING |
| SECTOR_CLASSIFICATION2 |      | STRING |
| SECTOR_CLASSIFICATION3 |      | STRING |
| CITY                   |      | STRING |

并将维度表与事实表Portfolio_Transactions关联，在此基础上分别做了CTR的计算, Count Distinct 操作，从多个维度对整个数据集做透视，平均性能如下：

| CTR   | Count Distinct |
| ----- | -------------- |
| 2.5 s | 100 ms         |

到这里大数据集的查询性能已经基本上可以满足我们业务分析需求。

#### Cube优化

Cube的设置会极大的影响到整个Cube的构建时间，存储空间和查询性能等。以下是Cube设置时几种常见的优化。

##### 事实表分区

对数据库熟悉的同学对分区的概念应该也是了解的，利用物理分区可以大幅度的提升查询性能，在使用Kylin构建Model时，也有同样的设置，因为我们一般是对一个相对较大的数据集进行处理，而数据也会随着时间逐渐增长，分区的划分会很好的提升整个系统的查询性能。

##### Cube剪枝

我们知道N个维度的Cube构建，会产生$2^n$个Cuboid, 如有三个普通的维度A、B、C，那么维度的组合有2的3次方个，分别是{空、A、B、C、AB、BC、AC、ABC}，而这里可能有许多维度并不需要，比如查询中一定会有B，或ABC中有层级关系。针对这样的情况，我们不需要将所有的Cuboid都进行构建。Kylin提供了三种方式对生成的Cube进行前枝：

- Mandatory(强制维度) 这里的维度意味着每次的查询都会在Group by 中出现，每一个这里的维度可以将整个Cuboid的构建减少一半。
- Hierarchy(层级维度) 在某些情况下，维度之间是有层级关系，而不满足这个层级关系的Cuboid我们则不需要构建，比如前面例子中的SECTOR_CLASSIFICATION1， SECTOR_CLASSIFICATION2， SECTOR_CLASSIFICATION3就是一个层级关系，我们不需要构建SECTOR_CLASSIFICATION2， SECTOR_CLASSIFICATION1， SECTOR_CLASSIFICATION3的组合。
- Joint(联合维度) 这里出现的维度，是指在查询中一定会同时的出现的分组。比如用户的查询语句中仅仅会出现 group by A, B, C，而不会出现 group by A, B 或者 group by C 等等这样的维度组合。这种情况下就可以将A、B、C三者设为联合维度。而Kylin也就只会构建ABC，不会构建AB,AC等。

##### RowKey

Kylin 以 Key-Value 的方式将 Cube 存储到 HBase 中，其中Key是Rowkey，也是在HBase中的Key,因此一个好的Key设置是查询优化的关键。其中Rowkey是由各维度的值拼接而成的；为了更高效地存储这些值，Kylin 会对它们进行编码和压缩；每个维度均可以选择合适的编码（Encoding）方式，默认采用的是字典（Dictionary）编码技术；字段支持的基本编码类型如下：

- dict：适用于大部分字段，默认推荐使用，但在超高基情况下，可能引起内存不足的问题；
- boolean：适用于字段值为true, false, TRUE, FALSE, True, False, t, f, T, F, yes, no, YES, NO, Yes, No, y, n, Y, N, 1, 0；
- integer：适用于字段值为整数字符，支持的整数区间为[ -2^(8N-1), 2^(8N-1)]；
- date：适用于字段值为日期字符，支持的格式包括yyyyMMdd、yyyy-MM-dd、yyyy-MM-dd HH:mm:ss、yyyy-MM-dd HH:mm:ss.SSS，其中如果包含时间戳部分会被截断；
- fix_length：适用于超高基场景，将选取字段的前 N 个字节作为编码值，当 N 小于字段长度，会造成字段截断，当 N 较大时，造成 RowKey 过长，查询性能下降，只适用于 varchar 或 nvarchar 类型；

另外RowKey的顺序可以按照如下规则进行优化：

- 需要进行过滤的维度放在前面
- 同样需要的过滤的维度，则将基数大的放在前面

#### 总结

Kylin的核心仍然是基于预计算，合理的将业务需求与Kylin的优势结合起来才是解决问题的关键。所以在Cube的构建过程中，开发人员需要业务人员进行沟通，理解业务需求，而不是单纯的从技术角度来解决所有问题。

