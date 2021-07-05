---
title: 任务调度器
date: 2021-03-30 18:38:26
tags:
    - 算法
    - 优化
categories:
    - 技术
---

又是新一周的《数字孪生系统》课堂，当我正昏昏欲睡时，前排的朋友回头戳我
“来，做道题清醒一下”
题面如下：
<!--more-->
>给你一个用字符数组 tasks 表示的 CPU 需要执行的任务列表。其中每个字母表示一种不同种类的任务。任务可以以任意顺序执行，并且每个任务都可以在 1 个单位时间内执行完。在任何一个单位时间，CPU 可以完成一个任务，或者处于待命状态。
>然而，两个 相同种类 的任务之间必须有长度为整数 n 的冷却时间，因此至少有连续 n 个单位时间内 CPU 在执行不同的任务，或者在待命状态。

>你需要计算完成所有任务所需要的 最短时间 。
>示例 1：
>输入：tasks = ["A","A","A","B","B","B"], n = 2
>输出：8
>解释：A -> B -> (待命) -> A -> B -> (待命) -> A -> B
     在本示例中，两个相同类型任务之间必须间隔长度为 n = 2 的冷却时间，而执行一个任务只需要一个单位时间，所以中间出现了（待命）状态。
>示例 3：
>输入：tasks = ["A","A","A","A","A","A","B","C","D","E","F","G"], n = 2
>输出：16
>解释：一种可能的解决方案是：
     A -> B -> C -> A -> D -> E -> A -> F -> G -> A -> (待命) -> (待命) -> A -> (待命) -> (待命) -> A 
>来源：力扣（LeetCode）
>链接：https://leetcode-cn.com/problems/task-scheduler

我带着困惑读完题后，戳他“这题能做，但不敢保证是最短时间”
他说“你不觉得这个题目是个数学优化问题吗？”
我无奈“这能算数学优化么...再说这优化题我哪会解啊，满足约束后就贪心”
作为数学优化方向的研究生，那我们就从优化的角度来分析下这道题吧~
第一步，分析题意。问题的**输入**是task_list，其元素为若干可重复的大写英文字母；问题需要根据task_list求解action_list，action_list为CPU状态序列，其包含task_list所有元素与"sleep"，**约束条件**是action_list相同元素间的间隔必须大于n；问题**目标**为满足该约束的action_list的长度最小值。
显然这道题与连续优化无关...但我还是要把它放到运筹的框架下去分析...从理论上分析，我们可以根据task_list生成满足约束条件的所有可行解，然后从可行解中选取最优解。然而这实在不是一个好主意，因为可行解理论上来说是无穷的，比如让CPU长时间处于sleep状态，因此我们的目标转化为**最小化CPU的sleep次数**。
让我们换个视角来看CPU的action_list，把它视为一个动力系统，对于每一个CPU时刻，我们需要根据一个policy去决策CPU的状态M(sleep或任务态)。那么，这个policy应该如何制定呢？
我认为，policy分为两步：
- 根据当前时刻的action_list和间隔范围n，对照task_list确定满足间隔约束的候选集
- 查找候选集中，选择当前**剩余任务数最大**的任务作为CPU当前状态M；从直觉上来说，我们要尽可能使得剩余种类的任务数均匀，这样才能减少间隔约束出现的可能。
接下来，我们需要考虑边界条件。如果当前候选集为空，CPU状态则为"sleep"。需要考虑到的是，这里满足间隔约束的候选集，并不是当前状态真正的候选集，其必须满足**该任务的余量大于0**，如果该任务的余量小于0，那么CPU仍进入sleep

数学的部分就到此结束啦（虽然我还不知道这求出来的是不是最小值...但我确定一定能给出满足约束的可行解...），接下来我们考虑问题的代码实现。
首先，我们需要知道，这个系统的终止时刻是什么？是task_list长度吗？显然不是，应该是action_list长度。action_list是动态增大的过程，那么action_list合适停止生长呢？应该是task_list剩余任务数为0时（你品，你细品，绝对不是task_list的长度）。
所以我们需要有个while循环，这个while循环的次数，是action_list的长度，也是最短的任务执行时间。
okay，这个问题的大框架我们就这样确定了：

``` python
    action_list = []
    while remain_tasks > 0:
        action = take_policy(action_list, n, task_list)
        action_list.append(action)
    print(len(action_list))
```
那么接下来，就涉及到如何维护remain_tasks和实施policy。我们分析一下需要的接口

> initial TaskManager from task_list
> pop task from TaskManager into action_list
> maintain remain_tasks from TaskManager
> decide current state for action_list based on policy

okay，那么看起来我们需要组织下TaskManager，这里我选择用python的dict作为其核心数据结构，前三点全部实现为TaskManager的类属性/方法，最后的决策过程我选择用函数来实现。

TaskManager实现如下：
``` python
class TaskManager:
    def __init__(self, task_lists):
        self.remain_tasks = len(task_lists)
        self.task_dict = dict.fromkeys(task_lists, 0)
        for item in task_lists:
            self.task_dict[item] = self.task_dict[item] + 1
    def pop_task(self, task):
        if self.task_dict[task] > 0:
            self.task_dict[task] = self.task_dict[task] - 1
            self.remain_tasks = self.remain_tasks - 1
        else:
            raise ValueError("Couldn't pop from empty item!")
```

decision_process过程实现如下：

``` python
import operator
def decision_process(action_list, n, cpu_timer, TM):
    schedule_lo = max(0, cpu_timer - n)
    schedule_hi = max(0, cpu_timer - 1) + 1
    block_set = set(action_list[schedule_lo:schedule_hi])
    # generate cand_dict
    cand_dict = {}
    for key,val in TM.task_dict.items():
        if key in block_set:
            continue
        else:
            cand_dict[key] = val
    # decide action
    action = "sleep"
    if len(cand_dict) == 0:
        return action
    else:
        best_cand, best_val = max(cand_dict.items(), key=operator.itemgetter(1))
        if best_val == 0:
            return action
        else:
            action = best_cand
            TM.pop_task(action)
            return action
```
这里用了operator的itemgetter函数，用于获取对象对应维度的数据，以
``` python
best_cand, best_val = max(cand_dict.items(), key=operator.itemgetter(1))
```
为例，cand_dict.items()为(key,val)数据类型，operator.itemgetter(1)获取val作为函数变量，传递给max（大概这里可以理解成泛型）。
总调度过程如下：
``` python
def schedule_process(task_list, n):
    TM = TaskManager(task_list)
    action_list = []
    cpu_timer = 0
    while TM.remain_tasks > 0:
        action = decision_process(action_list, n, cpu_timer, TM)
        action_list.append(action)
        cpu_timer = cpu_timer + 1
    return action_list, cpu_timer
```

写个用例测试下：

``` python
if __name__ == '__main__':
    task_list = ["A","A","A","A","A","A","B","C","D","E","F","G"]
    n = 2
    action_list, cpu_timer = schedule_process(task_list, n)
    print(action_list)
    print(cpu_timer)
```

输出为：
``` python
['A', 'B', 'C', 'A', 'D', 'E', 'A', 'F', 'G', 'A', 'sleep', 'sleep', 'A', 'sleep', 'sleep', 'A']
16
```
关于这道题，我能想到的其他优化点，应该是确定候选集：这一步是可以充分利用上一步信息，在n很大的情形下，候选集每次最多更新两位（unique的删除最远位，增加最新位），这里可以维护个数据结构，记录的是当前cand_set里的元素及其距离当前cpu_timer最近的id，然后再根据两步移动来更新。不过题面写n<100，我觉得就不用那么极致地追求优化了...

虽然这只是一道普通算法题，但对我而言，这道题能否做出来并不重要，我在意的是用自己的思维体系与知识结构去解读、分析。这道题让我联想到这学期《强化学习》和旁听的《最优化理论》中的DP Process，每一步的state, action和policy是如何影响动力系统变化的。其实数学理论的动规很有意思（可惜我错过郦老师的第一节动规课了...），虽然我本人并没刷过计算机动态规划的题（手动笑哭），所以我也不保证这题贪心就对~
最后想吐槽下自己的python代码风格...我觉得写得很有c++的感觉...
附上完整代码：
[上课原始版代码](https://pastebin.ubuntu.com/p/M7sTDbNMMQ/)这是我在上课时云里雾里写的，当时设计的还要排序...后来跟前排的朋友讨论意识到应该重新设计下数据结构，max就够了。
[当前版本代码](https://pastebin.ubuntu.com/p/VnTQmCFscv/)