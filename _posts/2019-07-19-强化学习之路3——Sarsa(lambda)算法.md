---
layout:     post   				    # 使用的布局（不需要改）
title:      强化学习之路3——Sarsa(lambda)算法				# 标题 
subtitle:   记录自己的强化学习之路 #副标题
date:       2019-07-19 				# 时间
author:     XF 						# 作者
header-img: img/post-bg-2015.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - 强化学习
---

# Sarsa(lambda)算法
## 知识点
最近在学强化学习，看了不少的教程，还是觉得莫烦大神的强化学习教程写的不错。所以，特意仔细研究莫烦的RL代码。在这贴上自己的理解。
<br>莫烦RL教程：<https://morvanzhou.github.io/tutorials/machine-learning/reinforcement-learning/>
<br>代码：<https://github.com/MorvanZhou/Reinforcement-learning-with-tensorflow/tree/master/contents>

![image](https://i.loli.net/2019/07/19/5d317c577880688423.png)
其实 lambda 就是一个衰变值, 他可以让你知道离奖励越远的步可能并不是让你最快拿到奖励的步, 所以我们想象我们站在宝藏的位置, 回头看看我们走过的寻宝之路, 离宝藏越近的脚印越看得清, 远处的脚印太渺小, 我们都很难看清, 那我们就索性记下离宝藏越近的脚印越重要, 越需要被好好的更新. 和之前我们提到过的 奖励衰减值 gamma 一样, lambda 是脚步衰减值, 都是一个在 0 和 1 之间的数.

下面是Sarsa(lambda)算法的伪代码：
![Sarsa(lambda)算法伪代码](https://i.loli.net/2019/07/19/5d317e4e4377169676.png)

## 2.迷宫游戏——Sarsa(lambda)算法
```python
import numpy as np
import pandas as pd


class RL(object):
    def __init__(self, action_space, learning_rate=0.01, reward_decay=0.9, e_greedy=0.9):
        self.actions = action_space  # a list
        self.lr = learning_rate
        self.gamma = reward_decay
        self.epsilon = e_greedy

        self.q_table = pd.DataFrame(columns=self.actions, dtype=np.float64)

    def check_state_exist(self, state):
        if state not in self.q_table.index:
            # append new state to q table
            self.q_table = self.q_table.append(
                pd.Series(
                    [0]*len(self.actions),
                    index=self.q_table.columns,
                    name=state,
                )
            )

    def choose_action(self, observation):
        self.check_state_exist(observation)
        # action selection
        if np.random.rand() < self.epsilon:
            # choose best action
            state_action = self.q_table.loc[observation, :]
            # some actions may have the same value, randomly choose on in these actions
            action = np.random.choice(state_action[state_action == np.max(state_action)].index)
        else:
            # choose random action
            action = np.random.choice(self.actions)
        return action

    def learn(self, *args):
        pass


# backward eligibility traces
class SarsaLambdaTable(RL):
    def __init__(self, actions, learning_rate=0.01, reward_decay=0.9, e_greedy=0.9, trace_decay=0.9):
        # 继承RL父类
        super(SarsaLambdaTable, self).__init__(actions, learning_rate, reward_decay, e_greedy)

        # backward view, eligibility trace.
        self.lambda_ = trace_decay
        self.eligibility_trace = self.q_table.copy()    # 记录回合的每一步
    # 检查State是否存在于Q表
    def check_state_exist(self, state):
        if state not in self.q_table.index:
            # 添加新的状态到Q表
            to_be_append = pd.Series(
                    [0] * len(self.actions),
                    index=self.q_table.columns,
                    name=state,
                )
            self.q_table = self.q_table.append(to_be_append)

            # 同时将State更新到eligibility_trace
            self.eligibility_trace = self.eligibility_trace.append(to_be_append)

    def learn(self, s, a, r, s_, a_):
        self.check_state_exist(s_)
        q_predict = self.q_table.loc[s, a]
        if s_ != 'terminal':
            q_target = r + self.gamma * self.q_table.loc[s_, a_]  # next state is not terminal
        else:
            q_target = r  # next state is terminal
        error = q_target - q_predict

        # increase trace amount for visited state-action pair

        # Method 1: 对于经历过的 state-action, 让他+1, 证明他是得到 reward 路途中不可或缺的一环
        # self.eligibility_trace.loc[s, a] += 1

        # Method 2:对于最近经历过的一次 (s,a1), 让他+1, (s,a0)置0，之前经历的乘一个衰减
        # 其实Method 2相当于对Method 1进行了数值为1的限幅
        self.eligibility_trace.loc[s, :] *= 0
        self.eligibility_trace.loc[s, a] = 1

        # 更新Q表
        self.q_table += self.lr * error * self.eligibility_trace

        # 更新eligibility_trace
        self.eligibility_trace *= self.gamma*self.lambda_
```

Method 1和Method 2的不同之处可以用这张图来概括:
![Method 1和Method 2区别](https://i.loli.net/2019/07/19/5d317f98690c659764.png)
这是针对于一个 state-action 值按经历次数的变化. 最上面是经历 state-action 的时间点, 第二张图是使用这种方式所带来的 “不可或缺性值”:
```python
self.eligibility_trace.ix[s, a] += 1
```
下面图是使用这种方法带来的 “不可或缺性值”:
```python
self.eligibility_trace.ix[s, :] *= 0; self.eligibility_trace.ix[s, a] = 1
```
实验证明选择下面这种方法会有更好的效果. 大家也可以自己玩一玩, 试试两种方法的不同表现.



