---
title: 稀疏矩阵乘法——Julia实现
date: 2021-05-24 15:34:56
tags:
    - 数值计算
    - Julia
categories:
    - 技术
---

五月初关注到Julia社区的稀疏矩阵乘法的可逆运算项目，其目标是使用**通用可逆语言**[NiLang](https://github.com/GiggleLiu/NiLang.jl)对稀疏矩阵乘法进行新的实现，借助NiLang自动微分框架使得稀疏矩阵乘法的反向传播、链式法则更自然地导出。
该项目的难度在于综合权衡稀疏矩阵**乘法的高效性**与NiLang的语言特性（程序中每一步均**可逆**，变量的分配与释放），在讨论NiLang的语言特性前，我们先考虑稀疏矩阵乘法的高效实现。
稀疏矩阵的特点在于其元素中有大量零元，因此在做稀疏矩阵与向量乘法速度很快；而这样的高效需要依托于其特殊的数据结构——在存储上，尽可能节约空间（省略0元）；在数据读取上，尽可能快。
本文结合Julia的SparseArrays对稀疏矩阵乘法的实现进行一些基本的讨论，其文章结构遵循：稀疏矩阵的数据结构-->稀疏矩阵乘法实现（矩阵-向量乘，共轭转置矩阵-向量乘）。
<!--more-->
## SparseCSC(稀疏列压缩存储)

Julia按列优先进行数据存储，因此其稀疏矩阵的存储数据结构为稀疏列压缩存储，而python等行优先存储语言则可能使用SparseCSR进行行压缩存储。
对于一个$m \times n $矩阵**M**，其稀疏列压缩存储的形式为三个一维数组，分别记为**V ROW_INDEX COL_INDEX**,其分别存储如下信息
- V 存储**M**所有非零元(节约存储空间)
- ROW_INDEX 存储V中元素对应的矩阵**M**行坐标
- COL_INDEX 存储矩阵M的压缩列信息；具体而言，对于列数col，col_start = COL_INDEX[col], col_end =  COL_INDEX[col+1]。则V[col_start, col_end]所取元素则对应col列中的元素。这种存储方式，使得提取矩阵M的col列达到O(1)的复杂度，因此特别适用于列展开的矩阵乘法提速。

上面的说明或许过于抽象，这里以python的SparseCSR实现示意图进行相应讲解（实在感慨于该图的生动形象）；首先将矩阵中的非零元数据按照行优先(对于SparseCSC则按照列优先)排序存储至data中，对于data中的每个元素，依次记录其列坐标对应至indices中的相应位置；最后在indptr中记录压缩行信息，其递推关系如下：

``` python
indptr[0] = 0 
indptr[k+1] = indptr[k] + cnt(row[k])
```

![avatar](https://i.stack.imgur.com/12bPL.png)

因此，如果indptr[k+1] != indptr[k]  （也即该列非零向量），则可以通过getindptr(A)[k]:(getindptr(A)[k+1] - 1)来获取原矩阵的第k行所有非零元素在data中对应的元素下标。如果说得更通俗些，indptr通过存储行元素的初始位置、末尾位置来压缩行元素的存储空间；对于SparseCSC同理，其通过压缩列元素的初始位置、末尾位置来压缩列元素的存储空间。

### SparseArrays数据结构

Julia对SparseCSC进行了相应的实现，其类声明如下

``` julia
struct SparseMatrixCSC{Tv,Ti<:Integer} <: AbstractSparseMatrix{Tv,Ti}
    m::Int                  # Number of rows
    n::Int                  # Number of columns
    colptr::Vector{Ti}      # Column j is in colptr[j]:(colptr[j+1]-1)
    rowval::Vector{Ti}      # Row indices of stored values
    nzval::Vector{Tv}       # Stored values, typically nonzeros
end
```

可以看到，nzval对应矩阵的非零元素；rowval对应非零元素的相应行坐标；colptr对应矩阵的列压缩（偏移信息）。我们可以看到，colptr使得访问稀疏矩阵的列元素非常简单快速，而访问稀疏矩阵的行会非常缓慢。这一点启发我们，在设计稀疏矩阵乘法时，**应尽可能利用SparseCSC格式的列参与运算，而避免提取行参与运算**。

## 稀疏矩阵乘法

在介绍稀疏矩阵乘法实现之前，我们先考虑一般性的矩阵乘法；

### 朴素矩阵乘法

$AB$的朴素矩阵乘遵循如下法则：

A按行展开：$A = (\vec a_1, \vec{a_2}, \dots, \vec{a_m})^T$
B按列展开：$B = \left(\boldsymbol{b_1}, \boldsymbol{b_2}, \dots, \boldsymbol{b_n}\right)$
内积计算$\vec{a_i} \boldsymbol{b_j} = \sum_{k=1}^{p} a_{i k} b_{k j}$

矩阵的朴素乘法没有充分利用SIMD（单指令多数据流）的特性，同时行展开会造成缓存穿透，命中率低降低计算效率，因此我们从矩阵外积展开的角度考虑如何利用SIMD与列访问的缓存命中

### 矩阵外积展开

 $AB$的外积展开矩阵乘遵循如下法则：
$A$按列展开：$A = \left(\boldsymbol{a_1}, \boldsymbol{a_2}, \dots, \boldsymbol{a_k}\right)$
$B$按行展开：$B = \left(\vec{b_1}, \vec{b_2}, \dots, \vec{b_k}\right)^\mathrm{T}$
外积计算$\boldsymbol{a_i}\vec{b_j}$

## Julia稀疏矩阵实现  

### 稀疏矩阵乘法

外积计算的每一步$\boldsymbol{a_i}\vec{b_j}$计算出完整的矩阵截面，k个截面相加即可得到完整的矩阵；然而在实际实现的过程中，每一个最小内部循环只能算出其中一行或一列向量，为了充分Julia的列优先存储原则，我们可以这样考虑：
对于out矩阵，最外层循环按列k进行计算；对于稀疏矩阵A，内层循环按其列col进行计算；最内层循环则对于固定的B[col,k]将其与矩阵A[row_idx, col]进行相乘，并将结果写入C[row_idx, k]；又C的元素应为所有A的列与B相乘元素加和，因此为+=运算；这样，在所有运算中，都是按列对矩阵进行提取，高效性得以保证。

其对应代码如下：

``` julia
function mul!(C::StridedVecOrMat, A::AbstractSparseMatrixCSC, B::Union{StridedVector,AdjOrTransStridedOrTriangularMatrix}, α::Number, β::Number)
    # 矩阵size检测，是否抛出异常
    size(A, 2) == size(B, 1) || throw(DimensionMismatch())
    size(A, 1) == size(C, 1) || throw(DimensionMismatch())
    size(B, 2) == size(C, 2) || throw(DimensionMismatch())
    # A的非零值
    nzv = nonzeros(A)
    # A的非零值行索引
    rv = rowvals(A)
    if β != 1
        β != 0 ? rmul!(C, β) : fill!(C, zero(eltype(C)))
    end
    # 按列计算C的结果
    for k = 1:size(C, 2)
        # 按列对A进行抽取
        @inbounds for col = 1:size(A, 2)
            # 固定B[col,k]
            αxj = B[col,k] * α
            # 稀疏矩阵快速乘 仅计算非零元的结果
            for j = getcolptr(A)[col]:(getcolptr(A)[col + 1] - 1)
            # 遍历col执行加和操作
                C[rv[j], k] += nzv[j]*αxj
            end
        end
    end
    C
end
```


### 稀疏共轭转置矩阵乘法

然而，尽管外积展开的计算方式能够加速计算，然而这种计算逻辑并不是刻板的，对于矩阵乘法而言，在保证计算结果正确的情形下，应以访存效率为第一优先级，以稀疏共轭转置矩阵乘法的实现为例

``` julia
function mul!(C::StridedVecOrMat, adjA::Adjoint{<:Any,<:AbstractSparseMatrixCSC}, B::Union{StridedVector,AdjOrTransStridedOrTriangularMatrix}, α::Number, β::Number)
    # 共轭转置惰性计算
    A = adjA.parent
    size(A, 2) == size(C, 1) || throw(DimensionMismatch())
    size(A, 1) == size(B, 1) || throw(DimensionMismatch())
    size(B, 2) == size(C, 2) || throw(DimensionMismatch())
    # 获取A的非零值
    nzv = nonzeros(A)
    # 获取A的非零值所在行索引(对应共轭转置的列索引)
    rv = rowvals(A)
    if β != 1
        β != 0 ? rmul!(C, β) : fill!(C, zero(eltype(C)))
    end
    # 按列计算C的输出结果
    for k = 1:size(C, 2)
        # 按列提取A的非零值（此时按行提取A共轭转置的非零值）
        @inbounds for col = 1:size(A, 2)
            tmp = zero(eltype(C))
            # A共轭转置的行元素
            for j = getcolptr(A)[col]:(getcolptr(A)[col + 1] - 1)
                # 与B的对应元素列相乘
                # 引入tmp
                tmp += adjoint(nzv[j])*B[rv[j],k]
            end
            C[col,k] += tmp * α
        end
    end
    C
end
```

显然，在做矩阵A的共轭转置与B相乘时，我们并不会真的在内存中创建A的共轭转置矩阵，而是采用**惰性求值**的思想，只有在A的共轭转置必须时，才会对其元素进行求共轭转置。
因此，这里我们仍使用A的元素，因此天然地得到被乘矩阵的行，进而使用朴素矩阵乘法；这里引入tmp也别有用意：如果直接用C[col,k]对tmp进行替换，则每一步都要乘以$\alpha$，增大了计算量。

## Julia稀疏矩阵可逆计算

GiggleLiu在他的[博客](https://nextjournal.com/giggle/how-to-write-a-program-differentiably)中给出了使用NiLang进行稀疏矩阵乘法可逆计算的实现，如下

``` julia
using NiLang
using SparseArrays: SparseMatrixCSC, AbstractSparseMatrix, nonzeros, rowvals, getcolptr

@i function mul!(C::StridedVecOrMat, A::AbstractSparseMatrix, B::StridedVector{T}, α::Number, β::Number) where T
    @safe size(A, 2) == size(B, 1) || throw(DimensionMismatch())
    @safe size(A, 1) == size(C, 1) || throw(DimensionMismatch())
    @safe size(B, 2) == size(C, 2) || throw(DimensionMismatch())
    nzv ← nonzeros(A)
    rv ← rowvals(A)
    if (β != 1, ~)
        @safe error("only β = 1 is supported, got β = $(β).")
    end
    # Here, we close the reversibility check inside the loop to increase performance
    @invcheckoff for k = 1:size(C, 2)
        @inbounds for col = 1:size(A, 2)
            αxj ← zero(T)
            αxj += B[col,k] * α
            for j = getcolptr(A)[col]:(getcolptr(A)[col + 1] - 1)
                C[rv[j], k] += nzv[j]*αxj
            end
            αxj -= B[col,k] * α
        end
    end
end
```

对于可逆计算，此处不做过多介绍（因为我也只懂皮毛...具体可以参考NiLang的[官方文档](https://github.com/GiggleLiu/NiLang.jl)、[论文](https://arxiv.org/abs/2003.04617)与GiggleLiu的[知乎](https://zhuanlan.zhihu.com/p/191845544)），这里重点提两个必须遵循的编程范例

- **所有操作必须可逆；** 常见的可逆运算有+= $\rightarrow$ $\leftarrow$等，但是*=不可逆（因为零元的存在，导致无法推断）；当然if语句通过if (precond, aftercond)也同样能够实现可逆；while for控制块也根据末态实现了相应的可逆机制
- **操作解耦；** 尽可能地操作解耦，比如不要出现adjoint(nzv[j])，复合运算会导致NiLang无法推断，因此你可能需要先分配变量，之后再释放

### 实现中遇到的问题

我自然也想动手实现下稀疏矩阵乘法的可逆计算，那么先立个小目标实现个共轭转置的计算吧，给出代码如下

``` julia
using NiLang
using SparseArrays: SparseMatrixCSC, AbstractSparseMatrix, nonzeros, rowvals, getcolptr, Adjoint, AbstractSparseMatrixCSC

@i function mymul!(C::StridedVecOrMat, adjA::Adjoint{<:Any,<:AbstractSparseMatrixCSC}, B::StridedVector{T}, α::Number, β::Number) where T
    A ← adjA.parent
    @safe size(A, 2) == size(C, 1) || throw(DimensionMismatch())
    @safe size(A, 1) == size(B, 1) || throw(DimensionMismatch())
    @safe size(B, 2) == size(C, 2) || throw(DimensionMismatch())
    nzv ← nonzeros(A)
    rv ← rowvals(A)
    
    if (β != 1, ~)
        @safe error("only β = 1 is supported, got β = $(β).")
    end
    # Here, we close the reversibility check inside the loop to increase performance
    @invcheckoff for k = 1:size(C, 2)
        @inbounds for col = 1:size(A, 2)
            tmp ← zero(T)
            for j = getcolptr(A)[col]:(getcolptr(A)[col + 1] - 1)
                tmprv ← zero(T)
                tmprv += nzv[j]
                tmprvad ← zero(T)
                tmprvad += adjoint(tmprv)
                tmp += tmprvad*B[rv[j],k]
            end
            C[col, k] += tmp * α
        end
    end
end
```

bechmark测试发现并没有原始方法的效率高...

原始方法

``` julia
@benchmark SparseArrays.mul!($(copy(out)), $adjoint(sp1), $v, 0.5+0im, 1)
BenchmarkTools.Trial: 
  memory estimate:  48 bytes
  allocs estimate:  1
  --------------
  minimum time:     173.800 μs (0.00% GC)
  median time:      177.100 μs (0.00% GC)
  mean time:        198.213 μs (0.00% GC)
  maximum time:     1.782 ms (0.00% GC)
  --------------
  samples:          10000
  evals/sample:     1
```

可逆版本

```julia
BenchmarkTools.Trial: 
  memory estimate:  144 bytes
  allocs estimate:  2
  --------------
  minimum time:     241.200 μs (0.00% GC)
  median time:      246.500 μs (0.00% GC)
  mean time:        264.895 μs (0.00% GC)
  maximum time:     3.260 ms (0.00% GC)
  --------------
  samples:          10000
  evals/sample:     1
```

我猜测可能效率低下的原因在于**内部循环创建临时变量**的问题，但我似乎又觉得这些变量是必要的，为了遵循NiLang的可逆解耦原则。还希望可以得到大佬的指教~

### Gvar的报错

虽然受到正向检测的重创，但我依然有勇气要让它接受梯度的检验，不妙的是，共轭转置乘法的检验似乎遇到了问题

```julia
using NiLang.AD
@benchmark (~mymul!)($(GVar(copy(out))), $(GVar(adjoint(sp1))), $(GVar(v)), $(GVar(0.5)), 1)

MethodError: no method matching (::Inv{typeof(mymul!)})(::Vector{Complex{GVar{Float64, Float64}}}, ::SparseMatrixCSC{Complex{GVar{Float64, Float64}}, Int64}, ::Vector{Complex{GVar{Float64, Float64}}}, ::GVar{Float64, Float64}, ::Int64)
Closest candidates are:
  (::Inv{typeof(mymul!)})(::StridedVecOrMat{T} where T, ::Adjoint{var"#s34", var"#s33"} where {var"#s34", var"#s33"<:AbstractSparseMatrixCSC}, ::StridedVector{T}, ::Number, ::Number) where T at In[34]:4

```

看起来，是没有匹配的方法，紧接着我对GVar进行了测试

``` julia
using NiLang.AD: GVar, grad
typeof(GVar(sp1))
SparseMatrixCSC{Complex{GVar{Float64, Float64}}, Int64}

typeof(GVar(adjoint(sp1)))
SparseMatrixCSC{Complex{GVar{Float64, Float64}}, Int64}
```

发现GVar可能对adjoint(sp1)进行了求值，因此其种类为SparseMatrixCSC，而非Adjoint类，导致我实现的方法无法匹配；当然，我不清楚我的推断是否正确，又或者，可逆计算共轭转置的稀疏矩阵乘法的实现是不必要的？既然Gvar无法识别出Adjoint Transpose，那么只需要正向；该项目只需要对其他乘法算子进行实现，例如稀疏矩阵为右乘矩阵或kron积之类？

## 参考文献

NiLang[官方文档](https://github.com/GiggleLiu/NiLang.jl)

Nilang[论文](https://arxiv.org/abs/2003.04617)

GiggleLiu的[blog](https://nextjournal.com/giggle/how-to-write-a-program-differentiably)