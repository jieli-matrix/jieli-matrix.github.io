---
title: 回文数乘法之数学技巧优化
date: 2021-03-26 16:35:25
tags:
    - 算法
    - 优化
categories:
    - 技术
---

  这周《数字孪生系统》课上有道题目很有意思，记录下自己的优化方案与遇到的坑。
  定义如下回文数乘法：
  ABCDEF * n = FEDCBA，试举出所有满足回文数乘法的六位数。
  例子：
  100001 * 1 = 100001
  219978 * 4 = 879912

  思路：最简单的思路，自然是对ABCDEF每一位进行暴力枚举，然后对n从1-9进行判断，是否满足回文乘法的要求。粗暴估算下，复杂度为O(N^7)，其中N=10。
  当然，即使是暴力，也需要注意细节：A位和F位需要从1枚举，以保证ABCDEF和FEDCBA**首位不为0**，为合法的六位数。

  那么，这个问题有优化的空间吗？
  我决定从约束条件入手，对搜索空间进行剪枝，仔细观察这个等式
  $$ABCDEF \times n = FEDCBA$$
  进行如下拆分
  $$(ABC\times 10^3 + DEF) \times n = (FED \times 10^3 + CBA)$$
  等式两边同时对$10^3$取余，得
  $$DEF \times n \text{mod}(10^3) = CBA$$
  通过取余变换，我们便可以对搜索空间进行剪枝，其伪代码如下：
  ``` python
  # i: def j: cba
  for i in range(100, 1000):
          for n in range(1,10):
              j = i * n % 10^3
              store i,j,n
  ```
 根据伪代码分析，改进后的复杂度为仅为O(N^4)，其中N=10，之后再对候选集里的元素进行回文乘法验证即可。这里我想强调的是，我从回文乘法的定义推导得到的取余式，是回文乘法数的**必要条件**，因此可以用于剪枝，如果是**充分条件**，过强的约束则可能错失部分可行解（运筹优化上头...）。
于是我开开心心的写好代码：

``` python
def cal_palid(sep):
    # 通过剪枝计算候选集
    cand_list = []
    sep = sep
    lo = int(sep / 10)
    hi = sep 
    # i: DEF j: CBA
    # ABCDEF * n = FEDCBA
    for i in range(lo, hi):
        for n in range(1,10):
            j = i * n % hi 
            if j > lo:
                cand_list.append((i, j, n))
    return cand_list 
def check_ans(i, j, k):
    # 利用回文乘法定义验证
    abc_str = ''.join(reversed(str(j)))
    def_str = str(i)
    abcdef_str = abc_str + def_str
    rev_abcdef_str = ''.join(reversed(abcdef_str))
    abcdef = int(abcdef_str)
    rev_abcdef = int(rev_abcdef_str)
    if abcdef * k == rev_abcdef:
        return True
    return False
```
输出了下合法回文数的个数，902，和旁边同学用暴力枚举的结果一样，看来是对的！
不过，且慢，真的对吗？
- 1 i的枚举范围是100-999吗？
- 2 j需要大于100吗？
仔细思考，取余变换后，合法六位数的约束条件是：
- 1 ABC > 100
- 2 FED > 100
然而，枚举的i代表DEF，从100开始枚举，则忽略了部分可行解，以100001为例，ABC=100，DEF=001，显然，1不在i枚举的范围内。
聪明的做法自然是，对FED从100-999进行枚举，然后转换成DEF，然后算出CBA的值，根据CBA求出ABC，判断ABC是否大于100，如果大于，则添加到候选集中。因此正确的剪枝函数如下：
``` python
def cal_palid(sep):
    cand_list = []
    sep = sep
    lo = int(sep / 10)
    hi = sep 
    # i: FED j: CBA
    # rev_i: DEF rev_j: ABC
    # ABCDEF * n = FEDCBA
    int_wd = len(str(sep)) - 1
    for i in range(lo, hi):
        for n in range(1,10):
            rev_i = int_reverse(i, int_wd)
            j = rev_i * n % hi
            rev_j = int_reverse(j, int_wd) 
            if rev_j >= lo:
                cand_list.append((rev_j, rev_i, n))
    return cand_list
```
这里又涉及到一个常见的代码基本功：如何实现数字反转？
最直观的想法，当然是先把数字变成字符串，然后对字符串进行反转。也有数学点的方法，模10取余法。复杂度差距不大，于是我改完了bug，开心地实现了int_reverse函数：
``` python
def int_reverse(num):
    num_str = str(num)
    revnum_str = ''.join(reversed(num_str))
    revnum = int(revnum_str) 
    return revnum
```
然后重新跑了一遍结果，发现变成了812。欸，为什么漏了可行解呢？
继续考虑100001这个例子，当算出j=1时，我们对CBA进行reverse，得到的结果是100吗？根据我实现的int_reverse来看，结果还是1！因为没有对0进行补齐！对于所有小于100的j，int_reverse的翻转结果ABC都是小于100的数，漏了可行解！
因此，需要对int_reverse进行修正，使用格式化字符串的思想：

``` python
def int_reverse(num, width):
    num_str = str(num).zfill(width)
    revnum_str = ''.join(reversed(num_str))
    revnum = int(revnum_str) 
    return revnum
```

这样，int_reverse就能返回正确的结果，我又跑了一遍代码，还是902。（所以改了两遍bug等于没改bug？？？）

当然，这道题目还提到八位数，应该如何考虑？其实很简单，还是对半分，而且我也不需要重新写个八位数回文数乘法，只需要传入sep=$10^4$即可。
如果是奇数位的回文数乘法呢？这时候中心位的处理就较为麻烦，我的想法是在剪枝过程中对中心位不做限制，仅约束前后(n-1)/2位，最后在验证回文乘的时候对中心位进行枚举。
以下是该问题的全部代码，通过修改sep的值，即可实现偶位数的回文乘法。
``` python
def int_reverse(num, width):
    num_str = str(num).zfill(width)
    revnum_str = ''.join(reversed(num_str))
    revnum = int(revnum_str) 
    return revnum
def cal_palid(sep):
    cand_list = []
    sep = sep
    lo = int(sep / 10)
    hi = sep 
    # i: FED j: CBA
    # rev_i: DEF rev_j: ABC
    # ABCDEF * n = FEDCBA
    int_wd = len(str(sep)) - 1
    for i in range(lo, hi):
        for n in range(1,10):
            rev_i = int_reverse(i, int_wd)
            j = rev_i * n % hi
            rev_j = int_reverse(j, int_wd) 
            if rev_j >= lo:
                cand_list.append((rev_j, rev_i, n))
    return cand_list
    
def check_ans(rev_j, rev_i, k):
    # calculate reverse str
    abc_str = str(rev_j)
    def_str = str(rev_i)
    abcdef_str = abc_str + def_str
    rev_abcdef_str = ''.join(reversed(abcdef_str))
    
    abcdef = int(abcdef_str)
    rev_abcdef = int(rev_abcdef_str)
    if abcdef * k == rev_abcdef:
        return True
    return False
def recon_palid(cand_list):
    ans_list = []
    for item in cand_list:
        rev_j, i, k = item 
        if check_ans(*item):
            ans_list.append(item)
    return ans_list

if __name__ == '__main__':
    sep = 10000
    cand_list = cal_palid(sep)
    ans_list = recon_palid(cand_list)
    int_wd = len(str(sep)) - 1
    for item in ans_list:
        rev_j, rev_i, k = item
        item = (str(rev_j), str(rev_i).zfill(int_wd), k)
        print(item)
```
当然，如果是短时间内骗分的话，最快的思路当然是把这道题退化成求回文数...回文数就是回文数乘法的充分条件...因为回文数乘1还是它本身...

最后，是一些题外话。其实我之前是很烦看OJ题的，因为我一直觉得除了为了应付找工作面视，没什么意义。我其实也不太擅长数据结构与算法，因为我也不知道这能用在哪。但最近看了[华为21软挑赛题](https://zhuanlan.zhihu.com/p/356155386)（我当然是没空参加...），尽管购买服务器与虚拟机分配都可以用线性规划去解（实现迁移），从数学建模角度来说，非常容易；但真正去实现这个系统的调度，涉及到虚拟机类、服务器类、调度类的交互，以及如果线性规划问题没有解，还需要实现一套贪心算法，通过对资源容量排序返回最富余的机器，这个过程中涉及增删改查，高效的操作离不开底层的数据结构。不过有人可能会说，这个交给工程师去做就好了，数学系的操心算法就好...然而我的过往经验是，与其让别人实现还不如自己实现，因为工程师可能觉得你就在纸上谈兵...
读研以后，我渐渐意识到数学和数学思想很重要。我本科的时候被强行灌输数学，但当时我并不真正觉得数学有用，那时候相比于计算机，数学理论太虚无缥缈了...但在论文量积累的过程中，我会渐渐发现，尽管都是优秀的算法工作，哪些只是trick，哪些是从本质上去考虑问题，而触及到本质时，往往是涉及到数学，需要的是复杂性、稳定性分析，而不是仅仅在数据集上跑个准确率。
希望两年后的自己，能透过问题的表象，抽象出底层的数学模型，会用数学解决问题，会用计算理论给出量化的分析，而不是迷失于一时的技巧。