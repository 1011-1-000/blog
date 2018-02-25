---
layout: post
title: "蒙提霍尔悖论源码验证"
date: 2018-02-25 22:20:15 +0800
comments: true
categories: Probability
tags: [Probability, Monty]
---

蒙提霍尔悖论又称三门问题[Monty Hall Problem](https://en.wikipedia.org/wiki/Monty_Hall_problem)，这个问题出自美国的电视游戏节目Let's Make a Deal。问题名字来自该节目的主持人蒙提·霍尔（Monty Hall）。在这个秀上有三扇门，其中有一扇门打开后可以获得一辆汽车，而其余的两扇门打开后则是山羊。

<!--more-->

#### 游戏规则

游戏的步骤如下：

- 参加游戏的人选定其中的一扇门
- 主持人选择另外两扇门中的一扇，且打开后只能是山羊，而不能是汽车
- 给参加游戏的人一个是否交换剩下两扇门的机会
- 游戏者做出决定，并打开门

这个悖论的关键在于：挑战者是应该选择交换还是选择坚持？这两种做法有没有区别？

#### 从理论的角度分析

一般来说，大多数的挑战者在选择后会认为其实换不换无所谓，其概率是相等的。但是实际上，如果交换的话会有更大的机率获得汽车。现在我们从理论的角度去考虑这个问题：

首先，我们假设有三扇门，分别为门1，门2， 门3，且挑战者选择了门1。

很显然，汽车在每扇门后的概率为：

|            |  门1  |  门2  |  门3  |
| :--------: | :--: | :--: | :--: |
| 对应门后有汽车的概率 | 1/3  | 1/3  | 1/3  |

按照游戏规则，接下来主持人需要为挑战者去除一个干扰项：

|            |         门1         |    门2    |    门3    |
| :--------: | :----------------: | :------: | :------: |
| 对应门后有汽车的概率 |        1/3         |   1/3    |   1/3    |
|  去除一个干扰项   | 去除门2：1/2， 去除门3：1/2 | 只能去除门3：1 | 只能去除门2：1 |

这一步是由挑战者确定是否交换，对于不交换的情况，那只有门1是汽车的时候，挑战者才能拿到汽车的奖励，此时的概率为1/3，而对于交换的情况，只要不是门1的情况，都可以获得奖励，概率为2/3。如下表：

|             |                  门1                  |           门2            |           门3            |
| :---------: | :----------------------------------: | :---------------------: | :---------------------: |
| 对应门后有汽车的概率  |                 1/3                  |           1/3           |           1/3           |
|   去除一个干扰项   |          去除门2：1/2， 去除门3：1/2          |        只能去除门3：1         |        只能去除门2：1         |
| 不交换，获得汽车的概率 | P(获得汽车) = 1/3 \* 1/2 + 1/3 \* 1/2 =1/3 |       P(获得汽车) = 0       |       P(获得汽车) = 0       |
| 交换后获得汽车的概率  |             P(获得汽车) = 0              | P(获得汽车) = 1/3 * 1 = 1/3 | P(获得汽车) = 1/3 * 1 = 1/3 |

由此可以看出，直觉中**换不换概率相等**的认识是错误的。

#### 程序模拟源码

前面一节对蒙提霍尔悖论做了一些简单的分析，这一部分会用程序来模拟，并得出了相似的结果。源码如下：

```python
# monty problem
# code is not optimal, but it can demonstrate the problem

import random
def open_the_door( doors, player_choice ):

  # choose which door should be open
  sample = [0,1,2]

  if doors[player_choice]:
    sample.remove(player_choice)
    open_door = random.choice(sample)
  else:
    for index, value in enumerate(doors):
      if index != player_choice and not value:
        open_door = index

  return open_door

def monty_problem( total_times, exchange = False ):

  bingo = 0

  for i in range(total_times):
    
    doors = [False, False, False]
    doors[random.randint(0,2)] = True
    player_choice = random.randint(0,2)
    open_door = open_the_door(doors, player_choice)

    if exchange:
      difference = set([0, 1, 2]) - set([player_choice, open_door])
      player_choice = list(difference)[0]

    if doors[player_choice]:
      bingo += 1

  print('P(exchange={}) = {}'.format(exchange ,bingo / total_times))

if __name__ == '__main__':
  monty_problem(100000, False)
  monty_problem(100000, True)
```

运行结果如下：

```
P(exchange=False) = 0.33178
P(exchange=True) = 0.66669
```

从这里可以看出，不交换和交换的概率分别接近在上一部分的推理结果。这一部分请自行验证。

#### 总结

其实这个问题还可以从另外一个角度，凭直觉给出答案。比如说，现在不仅仅是只有3扇门，而是有100扇，甚至是1000扇门，那么在你选择了其中的一扇门后，打开其余不是汽车的98或998扇门，那么这个时候，你是换还是不换呢？
