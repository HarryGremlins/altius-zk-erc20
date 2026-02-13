# R1CS vs PLONKish 约束系统

## 概述

ZK 证明系统的核心思路是：**将一段计算转化为数学约束（constraints），然后证明你知道满足所有约束的输入，但不泄露输入本身。**

不同的证明系统使用不同的约束表达形式，最主流的两种是 **R1CS** 和 **PLONKish**。

---

## 从计算到约束

无论哪种约束系统，流程都是：

```
高级代码 (Circom / Noir)
    ↓ 编译
算术约束 (R1CS / PLONKish gates)
    ↓ 证明系统 (Groth16 / PLONK)
ZK Proof
    ↓ 验证
链上验证合约
```

所有的计算——哈希、签名验证、余额检查——最终都要被拆解成有限域 (finite field) 上的加法和乘法约束。

---

## R1CS (Rank-1 Constraint System)

### 约束形式

每条约束的形式是：

```
⟨a, w⟩ × ⟨b, w⟩ = ⟨c, w⟩
```

其中 `w` 是所有变量（witness）的向量，`a`, `b`, `c` 是系数向量。

简化理解：**每条约束只能做一次乘法**。

### 示例

计算 `x³`：

```
// 不能直接写 x * x * x = y（两次乘法）
// 必须拆成单次乘法：

x * x = t       // 约束 1：引入中间变量 t
t * x = y       // 约束 2：t × x = x³
```

计算 `(a + b) * c`：

```
(a + b) * c = d  // 这是合法的，括号内是线性组合（加法免费）
```

### 特征

- **加法是免费的**：线性组合不消耗约束
- **每次乘法花费一条约束**：这是衡量电路大小的核心指标
- 使用的证明系统：**Groth16**（最常见）、Marlin 等

### 使用者

- Circom 编译到 R1CS
- snarkjs / rapidsnark 作为 prover

---

## PLONKish 约束系统

### 约束形式

每个 gate 的通用表达：

```
q_L·a + q_R·b + q_O·c + q_M·(a·b) + q_C = 0
```

- `a`, `b`, `c`：gate 的输入/输出 wire
- `q_L`, `q_R`, `q_O`, `q_M`, `q_C`：selector（选择系数），由电路编译时固定

### 通过设置 selector 表达不同操作

**加法 gate**（`a + b = c`）：

```
q_L=1, q_R=1, q_O=-1, q_M=0, q_C=0
→ 1·a + 1·b + (-1)·c + 0 + 0 = 0
→ a + b = c
```

**乘法 gate**（`a × b = c`）：

```
q_L=0, q_R=0, q_O=-1, q_M=1, q_C=0
→ 0 + 0 + (-1)·c + 1·(a·b) + 0 = 0
→ a·b = c
```

**常量 gate**（`a = 5`）：

```
q_L=1, q_R=0, q_O=0, q_M=0, q_C=-5
→ 1·a + 0 + 0 + 0 + (-5) = 0
→ a = 5
```

### 扩展能力

PLONKish 还支持：

- **Custom gates**：可以定义更复杂的门，比如一个门同时做加法和乘法
- **Lookup tables**：直接查表验证（比如范围检查 `0 ≤ x < 2^64`），而不需要拆成逐 bit 约束
- **Copy constraints**：通过排列（permutation）约束确保不同 gate 之间的 wire 值一致

### 使用者

- Noir 编译到 ACIR，后端 Barretenberg 使用 UltraPlonk/UltraHonk
- Halo2 也是 PLONKish

---

## 对比总结

| 维度 | R1CS | PLONKish |
|------|------|----------|
| 约束形式 | `a × b = c`（只能一次乘法） | `q_L·a + q_R·b + q_O·c + q_M·(a·b) + q_C = 0` |
| 表达能力 | 有限，只有乘法门 | 灵活，支持 custom gate |
| 加法操作 | 免费（线性组合） | 也可以免费，或用专门的加法 gate |
| Lookup table | ❌ 不支持 | ✅ 支持（范围检查等场景巨大优势） |
| Trusted setup | Groth16 需要 per-circuit setup | PLONK 用 universal SRS，一次搞定 |
| Proof 大小 | Groth16 极小（~128 bytes） | 较大（~2-4 KB） |
| 验证开销 | Groth16 便宜（~200k gas） | 较高（~300-500k gas） |
| 代表性系统 | Groth16, Marlin | PLONK, UltraPlonk, Halo2 |

---

## Lookup Table 的威力

PLONKish 相对 R1CS 的一个核心优势是 **lookup argument**。

以范围检查 `0 ≤ x < 2^64` 为例：

**R1CS 做法**：将 x 拆成 64 个 bit，每个 bit 约束为 0 或 1：

```
b_i * (1 - b_i) = 0    // 对每个 bit，共 64 条约束
x = Σ b_i * 2^i         // 再加上重组约束
// 总计 ~64+ 条约束
```

**PLONKish + Lookup 做法**：预计算一张表 `{0, 1, 2, ..., 2^16 - 1}`，将 x 拆成 4 个 16-bit 的 limb，每个 limb 查表验证：

```
x = limb_0 + limb_1 * 2^16 + limb_2 * 2^32 + limb_3 * 2^48
// 4 次 lookup + 1 条重组约束，远比 64 条约束高效
```

---

## 对本项目的选择

本项目选择 **Noir + PLONKish (UltraPlonk/UltraHonk)**，理由：

1. **Lookup table**：范围检查、Poseidon 优化等场景大量受益
2. **Universal SRS**：无需 per-circuit trusted setup，迭代开发更便捷
3. **Custom gates**：可以为 Poseidon 等高频操作定义专用门，进一步优化
4. **生态方向**：PLONKish 是当前 ZK 领域的主流演进方向
