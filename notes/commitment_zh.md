# Pedersen Commitment vs Poseidon Hash Commitment

## 概述

在 ZK 隐私协议中，commitment（承诺）用于隐藏 UTXO 的内容（金额、所有者等），同时保证内容不可篡改。常见的两种方案是 **Pedersen Commitment** 和 **Poseidon Hash Commitment**。

---

## Pedersen Commitment

### 原理

基于椭圆曲线上的离散对数困难问题：

```
C = v·G + r·H
```

- `v`：被承诺的值（如转账金额）
- `r`：随机致盲因子（blinding factor），保证隐藏性
- `G`, `H`：椭圆曲线上的两个独立生成元（generator point），且 `H` 相对于 `G` 的离散对数未知

### 核心特性：加法同态

```
C(v1, r1) + C(v2, r2) = (v1·G + r1·H) + (v2·G + r2·H)
                       = (v1 + v2)·G + (r1 + r2)·H
                       = C(v1 + v2, r1 + r2)
```

这意味着**链上可以直接验证余额守恒**，无需 ZK proof：

```
C_input1 + C_input2 == C_output1 + C_output2
// 合约层做 EC 点加法即可验证
```

### 安全属性

- **Hiding**（隐藏性）：因为有随机致盲因子 `r`，给定 `C` 无法反推 `v`
- **Binding**（绑定性）：承诺后无法找到另一个 `(v', r')` 使得 `C(v, r) = C(v', r')`（基于离散对数假设）

### 在 ZK 电路中的代价

椭圆曲线标量乘法（scalar multiplication）需要大量 field 运算：

- 一次 256-bit 标量乘 ≈ 数千个约束（constraints）
- 对电路大小有显著影响

---

## Poseidon Hash Commitment

### 原理

使用 ZK 友好的哈希函数 Poseidon 构造承诺：

```
C = Poseidon(value, owner_pubkey, randomness)
```

- `value`：被承诺的值
- `owner_pubkey`：UTXO 所有者的公钥
- `randomness`：随机数，保证隐藏性

### 核心特性：ZK 原生

Poseidon 的内部运算全部基于有限域上的加法和乘法（`x^5 + 常数`），直接对应 ZK 电路的原生操作：

```
// Poseidon 一轮的 S-box 操作
x → x^5  // 仅需 3 个乘法约束

// 整个 Poseidon hash ≈ 250 个约束
```

### 不具备同态性

```
Poseidon(a, ...) + Poseidon(b, ...) ≠ Poseidon(a + b, ...)
```

因此余额守恒**必须在 ZK 电路内证明**：

```rust
// Noir 伪代码
fn transfer(input_values: [u64; 2], output_values: [u64; 2]) {
    assert(input_values[0] + input_values[1] == output_values[0] + output_values[1]);
}
```

### 安全属性

- **Hiding**：有随机数，无法从 hash 反推原像
- **Binding**：基于 Poseidon 的抗碰撞性（collision resistance）

---

## 对比总结

| 维度 | Pedersen Commitment | Poseidon Hash Commitment |
|------|-------------------|------------------------|
| 数学基础 | 椭圆曲线离散对数 | ZK 友好哈希函数 |
| 加法同态 | ✅ 支持 | ❌ 不支持 |
| 余额守恒验证 | 链上 EC 点加法 | ZK 电路内证明 |
| 电路内开销 | 高（数千 constraints） | 低（~250 constraints） |
| 设计复杂度 | 混合模型（链上 EC + 电路内 hash） | 纯 hash，架构统一 |
| Noir 生态支持 | `std::hash::pedersen_commitment` | `std::hash::poseidon`，Aztec 首选 |

---

## 对本项目的选择

本项目采用 **Poseidon Hash Commitment**，理由：

1. **电路效率**：Poseidon 在电路内极其便宜，proof 生成更快
2. **架构统一**：commitment、nullifier、Merkle tree 全部基于 Poseidon，一致性高
3. **生态契合**：Noir + Barretenberg 的 PLONKish 约束系统与 Poseidon 天然匹配
4. **设计哲学**：既然全面拥抱 ZK，不需要 Pedersen 的链上同态验证能力——一切都在电路内证明

> 注意：Pedersen commitment 的加法同态 ≠ 同态加密（FHE）。前者只是 commitment 可以相加，后者是可以在密文上做任意计算。本项目拒绝的是 FHE，Pedersen commitment 本身并不违反这一原则。
