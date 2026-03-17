---
layout: post
title: "Learning-LLaMA-Decoder-Only"
date: 2026-03-17 10:00:00 +0800
categories: [til]
tags: [LLaMA,RoPE]
---

# 2026-03-17 学习日志：Transformer / 归一化 / RoPE 系统梳理



---

## 目录

- [1. 绝对位置编码是什么](#1-绝对位置编码是什么)
- [2. 全连接神经网络（FCNN / MLP）](#2-全连接神经网络fcnn--mlp)
- [3. CNN：卷积神经网络](#3-cnn卷积神经网络)
- [4. MoE：混合专家模型](#4-moe混合专家模型)
- [5. 归一化总览：BN / LN / RMSNorm](#5-归一化总览bn--ln--rmsnorm)
- [6. RMSNorm：LLaMA 常用的均方根层归一化](#6-rmsnormllama-常用的均方根层归一化)
- [7. RMSNorm（RN）和 LayerNorm（LN）的区别](#7-rmsnormrn和-layernormln-的区别)
- [8. Pre-Norm 是什么，为什么公式里要加 h](#8-pre-norm-是什么为什么公式里要加-h)
- [9. LayerNorm 与 BatchNorm 在张量维度上的区别](#9-layernorm-与-batchnorm-在张量维度上的区别)
- [10. BatchNorm 常见用在什么网络里](#10-batchnorm-常见用在什么网络里)
- [11. Self-Attention 的 score 到底是什么](#11-self-attention-的-score-到底是什么)
- [12. RoPE：旋转位置编码](#12-rope旋转位置编码)
- [13. RoPE 为什么能体现相对位置](#13-rope-为什么能体现相对位置)
- [14. RoPE 的二维详细推导](#14-rope-的二维详细推导)
- [15. 八股速记](#15-八股速记)

---

## 1. 绝对位置编码是什么

### 1.1 背景

Transformer 的自注意力机制本身只看 token 两两之间的相似度，如果不给位置信息，它并不知道：

- 谁在前谁在后
- 第几个 token 在哪里
- 两个 token 相隔多远

所以需要位置编码。

### 1.2 绝对位置编码的定义

**绝对位置编码（Absolute Positional Encoding）** 是指：

> 给序列中的每一个绝对位置（第 1 个、第 2 个、第 3 个……）分配一个位置向量，然后把这个位置向量加到 token embedding 上。

写成公式就是：

```math
x_i = \text{TokenEmbedding}(w_i) + \text{PosEmbedding}(i)
```

其中：

- `w_i` 是第 `i` 个 token
- `PosEmbedding(i)` 是第 `i` 个位置对应的位置向量

### 1.3 常见做法

#### （1）固定的正弦余弦位置编码

Transformer 原论文常用：

```math
PE(pos, 2i) = \sin\left(\frac{pos}{10000^{2i/d}}\right)
```

```math
PE(pos, 2i+1) = \cos\left(\frac{pos}{10000^{2i/d}}\right)
```

#### （2）可学习位置编码

直接学习一个位置向量表：

```math
P \in \mathbb{R}^{L \times d}
```

第 `i` 个位置取出 `P_i` 加到 token 上。

### 1.4 优缺点

**优点：**

- 简单直观
- 容易实现
- 明确告诉模型“第几个位置”

**缺点：**

- 更偏向绝对位置，不够自然地表达相对距离
- 超出训练长度时，外推能力通常一般
- 与 self-attention 的“两两关系”不完全匹配

---

## 2. 全连接神经网络（FCNN / MLP）

### 2.1 定义

全连接神经网络（Fully Connected Neural Network）指：

> 前一层的每一个神经元，都和后一层的每一个神经元相连。

### 2.2 单个神经元公式

```math
y = f(w_1x_1 + w_2x_2 + \cdots + w_nx_n + b)
```

其中：

- `x`：输入
- `w`：权重
- `b`：偏置
- `f`：激活函数
- `y`：输出

### 2.3 结构

- 输入层
- 隐藏层
- 输出层

### 2.4 特点

**优点：**

- 结构简单
- 适合表格数据 / 结构化数据
- 是很多深度学习结构的基础

**缺点：**

- 参数量大
- 对图像、序列等结构化强的数据不够友好
- 容易丢失局部关系

---

## 3. CNN：卷积神经网络

### 3.1 直观理解

CNN 可以理解成：

> 用一个小窗口在图像上滑动，专门找局部模式，比如边缘、线条、纹理、局部形状，再逐层组合成更高级的特征。

### 3.2 和全连接的本质区别

全连接：

- 把整张图“摊平”
- 所有像素彼此乱连
- 参数很多

CNN：

- 用局部感受野
- 参数共享
- 保留空间局部结构

### 3.3 常见模块

- 卷积层：提取局部特征
- 激活函数：增加非线性表达能力
- 池化层：降维、保留显著特征
- 全连接层：最后做综合判断

### 3.4 形象比喻

CNN 像一群视觉侦探：

- 第一层看边缘
- 中间层看纹理、局部形状
- 更深层看眼睛、耳朵、轮子等部件
- 最后输出整体分类结果

---

## 4. MoE：混合专家模型

### 4.1 定义

MoE（Mixture of Experts）核心思想是：

> 不是让所有参数都同时参与计算，而是先由一个门控 / 路由模块判断当前输入更适合哪些专家，再只激活少数专家处理。

### 4.2 组成

- Router / Gate：调度员
- Experts：专家子网络

### 4.3 形象理解

MoE 像医院分诊台：

- 先判断病人更适合哪一科
- 再把任务分给少数几个最合适的专家

### 4.4 优点

- 总参数量可以很大
- 单次前向不一定很贵
- 稀疏激活，兼顾容量和效率

### 4.5 问题

- 专家负载不均衡
- 路由错误会影响效果
- 训练与工程实现更复杂

---

## 5. 归一化总览：BN / LN / RMSNorm

归一化的核心目标是：

> 把激活值的尺度控制在合适范围，提高训练稳定性和收敛速度。

常见三类：

- BatchNorm（BN）
- LayerNorm（LN）
- RMSNorm

---

## 6. RMSNorm：LLaMA 常用的均方根层归一化

### 6.1 定义

RMSNorm 先计算向量的均方根：

```math
\text{RMS}(x) = \sqrt{\frac{1}{d}\sum_{i=1}^{d}x_i^2 + \epsilon}
```

然后做缩放：

```math
\text{RMSNorm}(x) = \gamma \odot \frac{x}{\text{RMS}(x)}
```

### 6.2 直观理解

RMSNorm 不关心均值是不是 0，它只关心：

> 这组数整体“幅度”是不是太大或太小。

可以类比成“调音量”：

- 不改内容
- 不改语气
- 只把音量调到合理范围

### 6.3 为什么 LLaMA 常用它

- 比 LayerNorm 更轻量
- 计算更简单
- 对大模型训练通常已经足够稳定
- 与 Pre-Norm + 残差结构配合很好

---

## 7. RMSNorm（RN）和 LayerNorm（LN）的区别

> 这里 RN 默认指 RMSNorm。

### 7.1 LayerNorm

```math
\mathrm{LN}(x)=\gamma \odot \frac{x-\mu}{\sqrt{\sigma^2+\epsilon}}+\beta
```

它做两件事：

1. 减均值（中心化）
2. 除标准差（控制尺度）

### 7.2 RMSNorm

```math
\mathrm{RMSNorm}(x)=\gamma \odot \frac{x}{\sqrt{\frac{1}{d}\sum_i x_i^2+\epsilon}}
```

它只做一件事：

1. 按 RMS 控制整体幅度

### 7.3 一句话区别

- **LayerNorm**：去中心 + 控尺度
- **RMSNorm**：只控尺度，不去中心

### 7.4 直观比喻

- LayerNorm：先把人拉回队伍中心，再统一动作幅度
- RMSNorm：不管人站在哪，只要求动作别太大别太小

---

## 8. Pre-Norm 是什么，为什么公式里要加 h

LLaMA 的 Pre-Norm 风格常写成：

```math
h = h + \text{Attention}(\text{RMSNorm}(h))
```

```math
h = h + \text{FFN}(\text{RMSNorm}(h))
```

### 8.1 什么是 Pre-Norm

Pre-Norm 指：

> 先做归一化，再进入子层（Attention / FFN）

对应形式：

```math
h = h + \text{Sublayer}(\text{Norm}(h))
```

与之对应的是 Post-Norm：

```math
h = \text{Norm}(h + \text{Sublayer}(h))
```

### 8.2 为什么要“加个 h”

这个 `+ h` 是**残差连接（Residual Connection）**。

作用：

- 保留原始信息
- 让子层学习“增量修正”
- 让深层网络更容易训练
- 有利于梯度传播

### 8.3 直观理解

`h` 像当前版本的草稿，子层输出像修改意见：

- 没有残差：直接重写整篇
- 有残差：在原稿上增量修改

---

## 9. LayerNorm 与 BatchNorm 在张量维度上的区别

对于 Transformer 中常见的张量：

```math
(B,\ T,\ D)
```

其中：

- `B`：batch size
- `T`：序列长度
- `D`：hidden size / d_model

### 9.1 LayerNorm

对每个 `(b, t)` 位置上的整个 `D` 维向量做归一化，即：

```math
X[b, t, :]
```

自己和自己比。

### 9.2 BatchNorm

通常对固定特征维，在 batch 维上做统计，即跨样本看分布。

### 9.3 一句话理解

- **LayerNorm**：每个 token 自己管自己
- **BatchNorm**：同一特征维，大家一起统计

### 9.4 为什么 Transformer 更偏爱 LN / RMSNorm

- 不依赖 batch 大小
- 更适合变长序列
- 更适合自回归生成
- 推理时 batch 很小时也稳定

---

## 10. BatchNorm 常见用在什么网络里

### 10.1 最常见：CNN

典型结构：

```math
\text{Conv} \rightarrow \text{BatchNorm} \rightarrow \text{ReLU}
```

常见于：

- ResNet
- DenseNet
- MobileNet 的不少版本
- 各种图像分类 / 检测 / 分割 backbone

### 10.2 也可用于 MLP

在传统全连接网络中也会见到：

```math
\text{Linear} \rightarrow \text{BatchNorm} \rightarrow \text{Activation}
```

### 10.3 在 GAN 中常见

Generator / Discriminator 的部分结构里曾经非常常见。

### 10.4 在 Transformer / LLM 中不主流

因为：

- 强依赖 batch 统计
- 对变长序列和自回归生成不友好
- 推理时 batch 小不稳定

---

## 11. Self-Attention 的 score 到底是什么

Transformer 里的注意力打分公式是：

```math
\text{Score}(Q,K)=QK^\top
```

更完整的 Scaled Dot-Product Attention 为：

```math
\text{Attention}(Q,K,V)=\text{softmax}\left(\frac{QK^\top}{\sqrt{d_k}}\right)V
```

### 11.1 单个元素的写法

第 `m` 个 query 与第 `n` 个 key 的打分：

```math
\text{score}_{m,n}=q_m^\top k_n
```

缩放后：

```math
\text{score}_{m,n}=\frac{q_m^\top k_n}{\sqrt{d_k}}
```

### 11.2 为什么要除以 `sqrt(d_k)`

因为维度大时点积值容易过大，softmax 会过尖，梯度不稳定。

### 11.3 自注意力中的 Q / K / V 来自哪里

```math
Q=XW_Q,\quad K=XW_K,\quad V=XW_V
```

因为都来自同一个输入 `X`，所以叫 self-attention。

---

## 12. RoPE：旋转位置编码

### 12.1 一句话定义

RoPE（Rotary Position Embedding）不是把位置向量加到输入上，而是：

> 对 Q、K 的相邻维度两两成对做旋转，让 attention score 天然带上相对位置信息。

### 12.2 核心思想

对每个位置 `m`，把 Q、K 中的维度按两两分组：

```math
(x_{2i}, x_{2i+1})
```

并按角度 `m\theta_i` 旋转：

```math
\begin{bmatrix}
x'_{2i}\\
x'_{2i+1}
\end{bmatrix}
=
\begin{bmatrix}
\cos(m\theta_i) & -\sin(m\theta_i)\\
\sin(m\theta_i) & \cos(m\theta_i)
\end{bmatrix}
\begin{bmatrix}
x_{2i}\\
x_{2i+1}
\end{bmatrix}
```

### 12.3 频率为什么是多尺度

不同维度对使用不同频率：

```math
\theta_i = 10000^{-2i/d}
```

作用：

- 高频：更敏感于局部、短距离
- 低频：更适合长距离、全局关系

### 12.4 RoPE 作用在哪里

通常流程为：

```math
Q=XW_Q,\quad K=XW_K,\quad V=XW_V
```

```math
\tilde Q = \text{RoPE}(Q),\quad \tilde K = \text{RoPE}(K)
```

```math
\text{Attention}(\tilde Q,\tilde K,V)=\text{softmax}\left(\frac{\tilde Q\tilde K^\top}{\sqrt{d_k}}\right)V
```

### 12.5 为什么只作用在 Q、K，不作用在 V

因为位置主要影响的是注意力权重，而权重由 `QK^T` 决定；`V` 是被加权汇总的内容载体。

---

## 13. RoPE 为什么能体现相对位置

这是 RoPE 面试最核心的一点。

### 13.1 单看一个 token

位置 `m` 的 query：

```math
\tilde q_m = R_m q
```

位置 `n` 的 key：

```math
\tilde k_n = R_n k
```

你会感觉这只是“每个 token 自己转了一下”。

### 13.2 真正关键在 QK 点积

attention score 是：

```math
\tilde q_m^\top \tilde k_n
=
(R_m q)^\top (R_n k)
```

利用旋转矩阵性质：

```math
R_m^\top R_n = R_{n-m}
```

所以：

```math
\tilde q_m^\top \tilde k_n
=
q^\top R_{n-m} k
```

这说明最终只和：

```math
n-m
```

有关，而不单独依赖 `m` 或 `n`。

### 13.3 一句话总结

> RoPE 虽然是按绝对位置分别旋转 Q 和 K，但在 QK 点积里，旋转角会相减，所以最终体现的是相对位置差。

---

## 14. RoPE 的二维详细推导

设二维 query 和 key：

```math
q=\begin{bmatrix} q_1 \\ q_2 \end{bmatrix},\qquad
k=\begin{bmatrix} k_1 \\ k_2 \end{bmatrix}
```

位置 `m` 上的 query 旋转为：

```math
\tilde q_m =
\begin{bmatrix}
q_1\cos(m\theta)-q_2\sin(m\theta)\\
q_1\sin(m\theta)+q_2\cos(m\theta)
\end{bmatrix}
```

位置 `n` 上的 key 旋转为：

```math
\tilde k_n =
\begin{bmatrix}
k_1\cos(n\theta)-k_2\sin(n\theta)\\
k_1\sin(n\theta)+k_2\cos(n\theta)
\end{bmatrix}
```

### 14.1 内积展开

```math
\tilde q_m^\top \tilde k_n
=
(q_1\cos m\theta-q_2\sin m\theta)(k_1\cos n\theta-k_2\sin n\theta)
```

```math
\quad +
(q_1\sin m\theta+q_2\cos m\theta)(k_1\sin n\theta+k_2\cos n\theta)
```

展开整理后得到：

```math
\tilde q_m^\top \tilde k_n
=
(q_1k_1+q_2k_2)\cos((m-n)\theta)
+
(q_1k_2-q_2k_1)\sin((m-n)\theta)
```

于是位置只通过：

```math
(m-n)\theta
```

出现，说明只依赖相对位置差。

### 14.2 更紧凑的矩阵形式

```math
\tilde q_m^\top \tilde k_n
=
(R(m\theta)q)^\top(R(n\theta)k)
```

```math
=
q^\top R(m\theta)^\top R(n\theta)k
```

```math
=
q^\top R((n-m)\theta)k
```

其中用到了：

- `R(\phi)^T = R(-\phi)`
- `R(a)R(b)=R(a+b)`

这就是 RoPE 体现相对位置的根本原因。

---

## 15. 八股速记

### 15.1 全连接神经网络

- 全连接：前一层每个神经元都连到后一层每个神经元
- 公式：`y = f(Wx+b)`
- 优点：简单、适合结构化数据
- 缺点：参数多，不擅长图像局部结构

### 15.2 CNN

- 核心：局部感受野 + 参数共享
- 卷积层提取局部特征
- 池化层保留显著信息、降低计算量
- 比全连接更适合图像

### 15.3 MoE

- 核心：不是所有参数都激活，而是路由到少数专家
- 组件：Router + Experts
- 优点：容量大、计算稀疏
- 缺点：路由和负载均衡复杂

### 15.4 BatchNorm / LayerNorm / RMSNorm

- BN：跨 batch 统计，常用于 CNN
- LN：对单个 token 的 hidden 维做归一化
- RMSNorm：只按 RMS 缩放，不减均值

### 15.5 RMSNorm vs LayerNorm

- LN：去中心 + 控尺度
- RMSNorm：只控尺度
- LLaMA 常用 RMSNorm，因为更轻量

### 15.6 Pre-Norm

- 形式：`h = h + Sublayer(Norm(h))`
- 含义：先归一化，再进子层
- `+h` 是残差连接，保留旧信息并让子层学习增量修正

### 15.7 Self-Attention

- 打分：`QK^T`
- 完整式：`softmax(QK^T / sqrt(d_k))V`
- 单个元素：`score_{m,n} = q_m^T k_n`

### 15.8 绝对位置编码

- 给每个绝对位置一个位置向量
- 加到 token embedding 上
- 优点：简单
- 缺点：更偏绝对位置，长文本外推一般

### 15.9 RoPE 是什么

- 对 Q、K 的相邻维度两两成对做旋转
- 不直接把位置向量加到 embedding 上
- 用旋转角表示位置

### 15.10 RoPE 为什么能体现相对位置

- 位置 `m` 的 Q 旋转 `mθ`
- 位置 `n` 的 K 旋转 `nθ`
- 在点积中变成 `R_m^T R_n = R_{n-m}`
- 所以 score 只依赖 `n-m`

### 15.11 RoPE 为什么只作用在 Q、K

- 因为位置影响的是注意力权重
- 权重由 `QK^T` 决定
- `V` 只是被加权聚合的内容

### 15.12 RoPE 优缺点

**优点：**

- 相对位置建模自然
- 无额外大参数
- 易实现
- 与 attention 兼容好

**缺点：**

- 超长上下文仍会退化
- 远距离可能出现周期性问题
- 实践中常配合 scaling 改造

### 15.13 最该背的三句话

1. **RoPE 不是把位置加到 embedding 上，而是对 Q、K 做位置相关旋转。**
2. **RoPE 虽然按绝对位置旋转，但在 QK 点积中最终只和相对位置差 `n-m` 有关。**
3. **LLaMA 常用 RMSNorm + Pre-Norm + RoPE，这三者分别解决尺度稳定、深层训练稳定和位置信息建模问题。**

---

## 今日一句话总结

今天的核心主线是：

> 从最基础的神经网络结构（FCNN / CNN / MoE），一路梳理到 Transformer 中的归一化（BN / LN / RMSNorm）、残差与 Pre-Norm、自注意力打分公式，以及 RoPE 如何通过“对 QK 做旋转”在 attention 中自然体现相对位置。

