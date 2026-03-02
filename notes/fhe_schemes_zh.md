# 全同态加密（FHE）方案对比

## 概述

全同态加密（Fully Homomorphic Encryption）允许在加密数据上直接计算，解密后的结果与在明文上执行相同计算的结果一致。

```
FHE.Enc(a) ⊕ FHE.Enc(b) = FHE.Enc(a + b)
FHE.Enc(a) ⊗ FHE.Enc(b) = FHE.Enc(a × b)
```

这与 **Pedersen Commitment 的加法同态**有本质区别，后者只支持加法：

```
Pedersen:  Com(a) + Com(b) = Com(a + b)     ✅ 加法
           Com(a) × Com(b) = ???             ❌ 无乘法
           Com(a) > Com(b)  = ???             ❌ 无比较

FHE:       Enc(a) + Enc(b) = Enc(a + b)     ✅ 加法
           Enc(a) × Enc(b) = Enc(a × b)     ✅ 乘法
           Enc(a) > Enc(b) → Enc(bool)       ✅ 比较（取决于具体方案）
```

---

## 核心概念

### 噪声与 Bootstrapping

FHE 密文包含**噪声**，每次运算噪声都会增长。运算次数过多后，噪声会淹没明文导致解密失败。

**Bootstrapping** 是"刷新"密文的过程 — 同态地执行解密电路以降低噪声。这是 FHE 中开销最大的操作。

```
新鲜密文：     噪声 = 小     ✅ 可解密
多次运算后：   噪声 = 大     ⚠️ 接近上限
Bootstrapping 后：噪声 = 小  ✅ 已刷新，可继续计算
```

### Leveled HE vs Fully HE

- **Leveled HE**：支持固定次数的运算（深度），无需 bootstrapping。参数需根据计算深度预设。
- **Fully HE**：通过 bootstrapping 支持无限次运算。更灵活但更慢。

---

## TFHE（环面全同态加密）

### 工作原理

TFHE 在**环面** T = R/Z（实数模 1）上运算。使用 LWE（Learning With Errors）样本加密单个比特或小整数：

```
LWE 密文：(a₁, a₂, ..., aₙ, b)
其中 b = Σ(aᵢ · sᵢ) + e + m · q/p

s: 密钥向量
e: 小噪声
m: 明文消息
q: 密文模数
p: 明文模数
```

### 核心特性：可编程 Bootstrapping（PBS）

TFHE 的标志性创新是**可编程 bootstrapping**：在刷新密文噪声的同时，还能执行一个查表函数。这意味着：

```
// 一次 bootstrapping 操作中：
Enc(x) → Enc(f(x))     // 同时刷新噪声 + 计算任意函数 f

// 例：与阈值比较
Enc(balance) → Enc(balance > 100)  // 一次 PBS 搞定
```

### 特征

- **操作数类型**：比特、整数（2-bit、4-bit、8-bit，原生最大约 16-bit）
- **Bootstrapping 速度**：现代硬件上约 10-20ms / 次
- **优势**：通过 PBS 实现任意运算，非常灵活
- **劣势**：大整数运算较慢，每次操作都需要 bootstrapping
- **代表项目**：Zama（fhEVM、Concrete、TFHE-rs）

### 区块链关联

Zama 的 **fhEVM** 以 TFHE 为核心方案，提供：
- 加密无符号整数类型（`euint8`、`euint16`、`euint32`、`euint64`）
- 同态运算：`+`、`-`、`*`、`<`、`>`、`==`、`if/else`
- 类 Solidity 开发体验：`TFHE.add(a, b)`、`TFHE.lt(a, b)` 语法

---

## BGV/BFV

### 工作原理

BGV 和 BFV 是密切相关的方案，基于 **Ring-LWE** 问题。在多项式环上加密模 t 的整数：

```
密文：(c₀, c₁) ∈ Rq × Rq
其中 Rq = Zq[x]/(xⁿ + 1)

解密：m = (c₀ + c₁ · s) mod q mod t
```

### 核心特性：SIMD 批处理

通过**中国剩余定理（CRT）**，单个密文可以并行编码**多个明文值**到不同"槽位"：

```
一个密文 = [v₁, v₂, v₃, ..., vₙ]   // n 个值打包在一起

// 操作按元素并行应用到所有槽位：
Enc([a₁, a₂, ..., aₙ]) + Enc([b₁, b₂, ..., bₙ]) = Enc([a₁+b₁, a₂+b₂, ..., aₙ+bₙ])
```

这使 BGV/BFV 在**批量处理**大量值时极为高效。

### 特征

- **操作数类型**：模 t 整数（可以很大），通过 SIMD 槽位批处理
- **优势**：大批量加法和乘法非常快
- **劣势**：比较/分支需要比特级分解（开销大）；有深度限制
- **代表库**：Microsoft SEAL、HElib、OpenFHE
- **BGV vs BFV**：BGV 在每次运算后缩放噪声（scale-then-encrypt）；BFV 在解密时处理噪声（encrypt-then-scale）。BFV 实现更简单，BGV 可以更高效。

### 区块链关联

较少用于链上计算，因为：
- 比较运算开销大（而余额检查是核心操作）
- SIMD 批处理在批量处理时优势明显，对单笔交易帮助有限
- 可能适用于批量结算或批量证明聚合

---

## CKKS

### 工作原理

CKKS 将**实数/复数**（浮点近似值）编码到多项式环中。与 BGV/BFV 精确整数运算不同，CKKS 把加密噪声视为近似误差的一部分：

```
编码：(3.14159, 2.71828, ...) → 多项式
加密：多项式 → 密文

// 计算后结果是近似的：
Enc(3.14) + Enc(2.71) ≈ Enc(5.85)   // 不是精确的 5.85000...
```

### 核心特性：近似算术

CKKS 可以高效处理类浮点运算：

```
// 加密数据上的矩阵乘法：
Enc(A) × Enc(B) ≈ Enc(A × B)

// 加密数据上的机器学习推理：
Enc(input) → model → Enc(prediction)
```

### 特征

- **操作数类型**：实数/复数（近似）
- **优势**：天然适配 ML 推理、科学计算、统计分析
- **劣势**：结果是近似的 — 误差会累积；不适合精确算术
- **代表库**：Microsoft SEAL、Lattigo、OpenFHE

### 区块链关联

**不适合区块链余额运算** — 金融交易需要精确算术。100.00 的余额在零值操作后必须仍然精确是 100.00，而不是 99.99999997。

---

## 方案对比

| 维度 | TFHE | BGV/BFV | CKKS |
|------|------|---------|------|
| 明文类型 | 比特 / 小整数 | 模 t 整数 | 近似实数 |
| 加法 | ✅（需 bootstrapping） | ✅（原生，快） | ✅（原生，快） |
| 乘法 | ✅（需 bootstrapping） | ✅（增加深度） | ✅（增加噪声） |
| 比较 | ✅（通过 PBS，高效） | ⚠️（比特分解，开销大） | ❌（近似，不可靠） |
| 分支 / if-else | ✅（通过 PBS） | ⚠️（开销大） | ❌ |
| 精确算术 | ✅ | ✅ | ❌（近似） |
| 批处理（SIMD） | ❌ | ✅（主要优势） | ✅ |
| Bootstrapping 速度 | 快（~10-20ms） | 慢（秒级） | 中等 |
| 单次运算开销 | 较高（每次都需 bootstrap） | 较低（可批处理） | 较低（可批处理） |
| 最适场景 | 任意整数逻辑 | 批量整数算术 | ML / 统计 |
| 区块链适配 | ✅ 最佳 | ⚠️ 可用于批处理 | ❌ 不适合 |

---

## 为什么 TFHE 是区块链的首选候选

1. **精确整数算术**：金融操作需要精确 — 不能有近似误差
2. **高效比较运算**：余额检查（`balance >= amount`）是核心操作；TFHE 通过可编程 bootstrapping 高效处理
3. **灵活运算**：任何布尔或整数函数都可以通过 PBS 表达
4. **Zama 生态**：fhEVM 为 EVM 兼容链提供了现成的集成路径
5. **单笔操作模型**：区块链交易通常是单笔处理（非批量），与 TFHE 的单次运算模型一致，而非 BGV/BFV 的 SIMD 批处理

---

## FHE vs Pedersen Commitment（总结）

对于 Altius 隐私模块：

| | Pedersen Commitment | FHE (TFHE) |
|---|---|---|
| 提供什么 | 将值隐藏在 commitment 中 | 将值加密以供计算 |
| 支持运算 | 仅加法 | 任意（加减乘比较分支） |
| 谁来计算 | 没有人对 committed 值计算；用户通过 ZK 证明属性 | 执行引擎在密文上计算 |
| 密钥管理 | 无（用户自己记住 randomness） | 全网密钥（pk/evk/sk）+ 门限分片 |
| Phase 1（隐藏金额） | ✅ 足够 | 杀鸡用牛刀 |
| Phase 2（加密 DEX 等） | ❌ 不够 | ✅ 必需 |
