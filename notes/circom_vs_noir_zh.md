# Circom vs Noir

## 概述

Circom 和 Noir 是目前最主流的两种 ZK 电路开发语言。Circom 是久经沙场的老兵，Noir 是快速崛起的新秀。两者的定位、设计哲学和开发体验有本质区别。

---

## Circom

### 基本信息

- 由 iden3 团队开发
- 完全独立的 DSL（Domain-Specific Language），不基于任何现有语言
- 编译目标：**R1CS**（Rank-1 Constraint System）
- 证明后端：**Groth16**（最常见）、PLONK（通过 snarkjs）

### 语法示例

```circom
pragma circom 2.0.0;

include "circomlib/poseidon.circom";
include "circomlib/comparators.circom";

template Transfer() {
    signal input balance;
    signal input amount;
    signal output newBalance;

    // Manually define the constraint
    newBalance <== balance - amount;

    // Range check: amount <= balance
    component check = LessEqThan(64);
    check.in[0] <== amount;
    check.in[1] <== balance;
    check.out === 1;
}

component main = Transfer();
```

### 核心概念

- **signal**：电路中的变量（wire），分为 `input`、`output`、中间 signal
- **template**：可复用的电路模块（类似函数/类）
- **component**：template 的实例化
- **约束操作符**：
  - `<==`：赋值 + 约束（assign and constrain）
  - `===`：纯约束（只约束，不赋值）
  - `<--`：纯赋值（⚠️ 危险！不生成约束，常见漏洞来源）

### 工具链

- `circom`：编译器，输出 R1CS + WASM/C++ witness generator
- `snarkjs`：JavaScript 实现的 prover/verifier
- `rapidsnark`：C++ 实现的高性能 prover
- `circomlib`：官方标准库（Poseidon、MerkleTree、EdDSA、comparators 等）
- Trusted setup：需要用 snarkjs 做 Powers of Tau + per-circuit Phase 2

### 生产项目

- Tornado Cash
- Semaphore（以太坊匿名信号协议）
- WorldID（Worldcoin 的身份证明）
- zkEmail

---

## Noir

### 基本信息

- 由 **Aztec Labs** 开发
- 语法高度类似 Rust，但是独立的语言和编译器
- 编译目标：**ACIR**（Abstract Circuit Intermediate Representation）
- 证明后端：**Barretenberg**（UltraPlonk / UltraHonk）

### 语法示例

```rust
use std::hash::poseidon;

fn main(
    balance: u64,
    amount: u64,
    expected_commitment: pub Field,
    secret: Field,
) {
    // Balance check — just normal code
    assert(amount <= balance);
    let new_balance = balance - amount;

    // Compute commitment
    let commitment = poseidon::bn254::hash_2([new_balance as Field, secret]);
    assert(commitment == expected_commitment);
}

#[test]
fn test_transfer() {
    // Built-in testing support
    let secret = 12345;
    let commitment = poseidon::bn254::hash_2([50, secret]);
    main(100, 50, commitment, secret);
}
```

### 核心概念

- **普通变量**：像写 Rust 一样声明变量，编译器自动生成约束
- **`pub` 关键字**：标记为公开输入（public input），其他参数默认为私有输入（private witness）
- **`assert`**：生成约束——编译器将 assert 条件转化为电路约束
- **类型系统**：支持 `u8`/`u16`/`u32`/`u64`、`Field`、`bool`、struct、array 等
- **标准库**：内置 Poseidon、Pedersen、ECDSA、Merkle tree、SHA-256 等

### 工具链

- `nargo`：一体化 CLI（编译、执行、测试、prove、verify）
- `nargo test`：内置测试框架
- `nargo prove` / `nargo verify`：生成和验证 proof
- 包管理器：支持 GitHub 依赖

### 生产项目

- Aztec Protocol（Aztec Labs 自家的 L2 隐私协议）
- 多个 hackathon 项目和快速增长的开源生态

---

## 开发体验对比

### 写一个"证明 x 的平方等于 y"的电路

**Circom**：

```circom
template Square() {
    signal input x;
    signal output y;
    y <== x * x;
}
component main {public [y]} = Square();
```

**Noir**：

```rust
fn main(x: Field, y: pub Field) {
    assert(x * x == y);
}
```

### 写一个 Merkle proof 验证

**Circom**：

```circom
include "circomlib/poseidon.circom";

template MerkleProof(depth) {
    signal input leaf;
    signal input pathElements[depth];
    signal input pathIndices[depth];
    signal output root;

    component hashers[depth];
    signal hashes[depth + 1];
    hashes[0] <== leaf;

    for (var i = 0; i < depth; i++) {
        hashers[i] = Poseidon(2);
        // Select left/right based on path index
        hashers[i].inputs[0] <== hashes[i] + pathIndices[i] * (pathElements[i] - hashes[i]);
        hashers[i].inputs[1] <== pathElements[i] + pathIndices[i] * (hashes[i] - pathElements[i]);
        hashes[i + 1] <== hashers[i].out;
    }

    root <== hashes[depth];
}
```

**Noir**：

```rust
use std::hash::poseidon;

fn compute_root(leaf: Field, path: [Field; 32], indices: [u1; 32]) -> Field {
    let mut current = leaf;
    for i in 0..32 {
        let (left, right) = if indices[i] == 0 {
            (current, path[i])
        } else {
            (path[i], current)
        };
        current = poseidon::bn254::hash_2([left, right]);
    }
    current
}
```

---

## 安全性对比

### Circom 的经典隐患：Under-constrained Bug

```circom
template Dangerous() {
    signal input a;
    signal input b;
    signal output c;

    // ⚠️ <-- 只赋值，不约束！
    // 恶意 prover 可以给 c 任意值
    c <-- a * b;

    // 正确写法应该是：
    // c <== a * b;
}
```

`<--` 只做赋值不生成约束，这是 Circom 中最危险的操作。编译器不会报错，但证明系统存在漏洞。Tornado Cash 的早期版本就曾出现过此类问题。

### Noir 的安全优势

- 没有"纯赋值不约束"的操作——编译器自动为所有计算生成约束
- 强类型系统在编译期捕获类型不匹配
- 溢出检查内置（`u64` 运算自动做范围约束）
- 但仍需开发者确保逻辑正确（ZK 电路的逻辑漏洞任何语言都无法完全避免）

---

## 对比总结

| 维度 | Circom | Noir |
|------|--------|------|
| 语言类型 | 独立 DSL | Rust-like 独立语言 |
| 抽象层级 | 低级（手写 signal 和约束） | 高级（像写普通程序） |
| 约束系统 | R1CS | PLONKish (ACIR) |
| 证明后端 | Groth16（默认） | UltraPlonk / UltraHonk |
| Trusted Setup | Per-circuit ceremony | Universal SRS |
| Proof 大小 | ~128 bytes | ~2-4 KB |
| 链上验证 Gas | ~200k | ~300-500k |
| 开发效率 | 慢，手动管理约束 | 快，接近写 Rust |
| 测试 | 需要外部工具 | 内置 `nargo test` |
| 包管理 | 手动 include | 内置包管理器 |
| 安全风险 | `<--` under-constrained bug | 编译器兜底更多 |
| 生态成熟度 | 高（5+ 年，大量生产项目） | 中（快速成长中） |
| 标准库 | circomlib（成熟） | std（持续完善） |

---

## 对本项目的选择

本项目采用 **Noir**，理由：

1. **开发效率**：Rust-like 语法，学习成本低，迭代快
2. **安全性**：编译器自动生成约束，减少 under-constrained 风险
3. **PLONKish 优势**：lookup table 对范围检查等操作有巨大优势
4. **无 per-circuit setup**：开发过程中修改电路无需重新做 trusted setup
5. **Aztec 生态**：Noir 是 Aztec 的核心语言，长期维护有保障
6. **可维护性**：项目后期需要持续迭代（Parallel EVM 优化等），Noir 代码更易维护
