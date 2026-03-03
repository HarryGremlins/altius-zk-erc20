# 未解决问题与技术难点

## 概述

本文梳理了 Altius 隐私模块技术方案中的主要技术难点和未确定的设计决策。按严重性和依赖关系排序 — 前两项是 Phase 1 实现的直接阻塞项，其余所有组件都依赖这两项决策。

---

## 1. ~~复合 ZK Proof 设计~~（已解决）

**严重性**：~~关键~~ → 已解决

**决策**：使用 **PLONK（Universal SRS）** 作为统一证明系统。所有证明语句（wrap、保密转账、unwrap）表达为算术电路，在单一 PLONK 实例下使用 KZG 多项式承诺。这消除了组合独立 sigma 协议的需要，并在链上只需一个验证器。详细理由见 `llms/260303-proof-system-decisions.md`。

**原始问题**：方案中写的是 "Bulletproofs 或等效方案，待定"，但保密转账实际需要的**不是一个普通的 range proof**。证明者必须同时证明：

1. 知道能打开 `com_sender` 的 `(balance, r)` — commitment opening proof
2. `balance >= amount` — 差值的 range proof
3. `amount > 0` — 转账金额的 range proof
4. `com_transfer = Com(amount, r_transfer)` — 转账 commitment 的 opening proof

标准 Bulletproofs 只能证明单个已承诺值 `v ∈ [0, 2^n)`。而这里需要证明的是**两个 commitment 之间的关系**（发送方余额 commitment 和转账 commitment），并与 range proof 组合。

### 为什么这很难

有两种大方向，各有取舍：

**方案 A — 组合 sigma 协议**：将 Schnorr 类证明（用于 commitment opening）与 Bulletproofs（用于 range 约束）通过 Fiat-Shamir 启发式组合成自定义证明。这保留了 Bulletproofs 紧凑的 ~600 字节 proof 大小，但需要精心设计协议以确保组合多个证明语句时的可靠性（soundness）。组合必须共享统一的 transcript 以防止 proof-splicing 攻击。

**方案 B — 通用 ZK 电路**：将所有约束表达在单一电路中（如 Circom/Groth16 或 Noir/UltraPlonk）。实现上更简单正确，但：
- 如果用 Groth16 则失去 Bulletproofs 的无需 trusted setup 属性
- 需要在电路内做 EC 标量乘法来验证 commitment（数千个约束）
- Proof 大小从 ~600 字节变为 ~128 字节（Groth16）或 ~2-4 KB（PLONK）

方案尚未指定选择哪种路径，而该选择将显著影响 proof 大小、验证成本和实现复杂度。

### 额外复杂性：三种不同的 Proof 类型

方案实际需要三种不同的 proof：
- **Wrap proof**：证明 `commitment == Com(amount, r)` 其中 `amount` 公开 — 本质上是离散对数知识证明
- **保密转账 proof**：上述完整复合证明
- **Unwrap proof**：类似保密转账但 `amount` 公开

如果它们使用不同的证明系统（如 wrap 用 Schnorr，transfer 用 Bulletproofs），合约需要多条验证路径，增加了攻击面和复杂度。

---

## 2. 椭圆曲线选择

**严重性**：关键 — 级联影响所有组件

曲线选择未确定，但影响每个组件：

| 组件 | 曲线依赖 |
|------|---------|
| Pedersen commitment | 定义 `G`、`H` 所在的群；决定安全级别 |
| Bulletproofs | 原生工作在素数阶群上；配对友好曲线（BN254）有不必要的开销 |
| ECIES | 密钥格式，与 EVM secp256k1 的兼容性 |
| 预编译 | Altius 引擎需要实现哪些 EC 操作 |
| 链上验证 | EC 点加法/标量乘法的 gas 成本 |

### 核心矛盾

- **secp256k1**（EVM 原生）：从 EVM 密钥派生自然契合，但没有通用 `ecAdd`/`ecMul` 预编译 — EIP-196 预编译只适用于 alt_bn128（BN254）
- **BN254**（alt_bn128）：有 EVM 预编译支持，但它是配对友好曲线（对 Pedersen/Bulletproofs 没有必要，且配对友好曲线的安全边际弱于预期）。此外 ECIES 密钥会与 EVM 密钥不同，增加密钥派生层级的复杂度
- **Curve25519 / Ristretto**：Bulletproofs 的最优选择（Monero 和 Penumbra 使用），但需要完全自定义的预编译，无 EVM 生态支持

由于 Altius 执行引擎可以自定义预编译，这比在以太坊主网上限制少 — 但选择仍需明确做出，且阻塞所有实现工作。

### 解决方案

**决策**：使用 **BN254（alt_bn128）**。

PLONK + KZG 多项式承诺（问题 #1 的决策）迫使我们选择配对友好曲线 — KZG 验证本质上是一个配对方程检查，而 secp256k1 没有配对。两个现实选项是 BN254 和 BLS12-381。

选择 BN254 而非 BLS12-381 基于务实考虑：
- **工具链成熟度**：整个以太坊 ZK 工具链（Barretenberg、snarkjs、Solidity verifier 生成器）都围绕 BN254 构建
- **性能**：更小的域（254-bit vs 381-bit）意味着更快的 EC 运算
- **生态对齐**：zkSync、Polygon zkEVM、Aztec、Semaphore 都使用 BN254
- **ECIES 密钥分离已处理**：密钥派生层级将 `sk_ecies = KDF("ecies" || evm_sk)` 派生在 BN254 上，EVM 签名密钥保留在 secp256k1 上

约 100-bit 安全级别（相比 BLS12-381 的约 128-bit）是已知折衷，但目前不存在实际攻击。如需更强安全性，后续可在同一 PLONK + KZG 框架内迁移至 BLS12-381。

---

## 3. 接收方状态同步

**严重性**：中等 — 工程可靠性问题

ECIES memo 机制为接收方状态创造了一条依赖链：

```
转账 1：Bob 收到 (v1, r_tx1) → balance = v1, r = r_tx1
转账 2：Bob 收到 (v2, r_tx2) → balance = v1 + v2, r = r_tx1 + r_tx2
转账 3：Bob 收到 (v3, r_tx3) → balance = v1 + v2 + v3, r = r_tx1 + r_tx2 + r_tx3
```

如果 Bob 的客户端**漏掉了转账 2**（客户端崩溃、索引器延迟、RPC 故障），他的本地状态将永久失同步 — 无法为任何后续交易构造有效证明，因为他的 `(balance, r)` 对不再能打开 `com_bob`。

### 方案中未提及的边界情况

- **损坏的 calldata**：如果 memo 密文格式错误（截断、编码错误），解密失败，整条后续致盲因子链断裂
- **链重组**：Bob 已处理的交易在重组后被移除，但替代区块包含相同转账但交易顺序不同 — Bob 的状态是否需要回滚？
- **同一区块内多笔交易**：Bob 在同一区块中收到两笔保密转账，处理顺序影响他的累积 `r` 值
- **恶意 memo**：恶意发送方可以提交链上 proof 有效但 memo 故意错误的交易 — Bob 的 commitment 在链上正确更新了，但他收到的 `(v, r_tx)` 是垃圾值，无法恢复余额

### 方案描述的恢复机制

方案声明："扫描链上所有隐私操作，按时间顺序重放以重建 nonce 序列。"理论上可行，但：

- 需要按顺序解密**每一条**收到的 memo — 复杂度 O(n)，n 为用户相关交易总数
- 如果单条解密失败（损坏的 memo、恶意发送方），整个恢复过程中断
- 没有链上检查点或本地状态承诺机制来检测或从失同步中恢复

### 可能的缓解措施（方案中未提及）

- 每次操作后在链上存储用户 `(balance, r)` 的哈希作为检查点
- 在加密 memo 中包含序列号，使接收方可以检测缺口
- 允许发送方在链上证明 memo 的正确性（开销大但可防止恶意 memo）

---

## 4. 对抗条件下的 Nonce 管理

**严重性**：中等 — 影响恢复和确定性

Pedersen 随机数的密钥派生使用客户端 nonce：

```
r_tx = KDF("pedersen" || evm_sk || nonce)
```

方案声明 nonce"由用户的链上交易历史隐式确定"，但以下场景会打破这一假设：

- **回退的交易**：消耗 nonce `n` 的交易提交后被回退（如余额不足）。nonce 是否递增了？如果用户客户端在本地递增了但链上状态未改变，nonce 序列就会分歧。
- **并发交易**：用户快速连续提交两笔隐私操作（第一笔确认前就提交第二笔）。两笔都使用相同的 nonce 值构造，导致相同的 `r_tx` — 这是致盲因子复用，可能泄露信息。
- **链重组**：用户的交易在重组后被重新排序，改变了隐式的 nonce 映射。

### 可能的缓解措施

- 使用显式的链上 nonce 计数器（简单的 `uint256`，每次成功的隐私操作递增）— 增加一次 SSTORE 但消除歧义
- 从交易特定的熵派生 `r_tx`（如 `KDF(evm_sk || tx_hash)`）而非顺序 nonce — 但恢复时需要知道所有交易哈希

---

## 5. Wrap Proof — 验证路径设计

**严重性**：中等 — 单独看简单，集成时麻烦

Wrap 操作需要证明："我知道 `r` 使得 `commitment == amount·G + r·H`"，其中 `amount` 是公开的。由于 `amount` 已知，验证者可以计算 `amount·G` 并检查 `commitment - amount·G` 是 `H` 的有效倍数。这归结为关于 `(commitment - amount·G)` 对 `H` 的**离散对数知识证明** — 本质上是一个 Schnorr 证明。

密码学上很简单，但集成问题仍在：如果保密转账用 Bulletproofs（或复合证明系统）而 wrap 用 Schnorr 证明，合约需要**两条独立的验证代码路径**。每个验证器都是额外的攻击面和审计目标。

理想情况下，所有 proof 类型应使用统一的证明系统，但这可能迫使次优折衷（比如为简单的 Schnorr 证明使用完整电路）。

---

## 6. Phase 2 承诺桥接

**严重性**：高 — 但推迟到 Phase 2

双轨架构要求两种表示保持一致：

```
ct_balance:  FHE.Enc(balance)              // FHE 轨道
com_balance: PedersenCommit(balance, r)     // ZK 轨道
```

方案列出了三种桥接方式：

| 方式 | 保证强度 | 成本 |
|------|---------|------|
| 仅在 wrap/unwrap 时验证 | 弱（只在边界检查） | 低 |
| 执行引擎门限解密验证 | 中（需信任验证者） | 中 |
| ZK 证明 FHE 加密正确性 | 强（无需信任） | 极高 |

### 为什么强方案很难

在 ZK 电路内证明"这个 TFHE 密文加密的值与这个 Pedersen commitment 相同"需要：

- 将 LWE 加密（大整数上的模运算）编码为域算术约束
- TFHE 密文维度 `n` 通常为 500-1000 — 每个 LWE 样本涉及该维度的内积运算
- 估计电路规模：数百万约束，比 range proof 电路大数个数量级

这是一个活跃的研究领域。部分项目（如 Sunscreen、PESCA）正在探索 ZK-FHE 桥接，但均未达到生产可用状态。

---

## 总结

| # | 难点 | 严重性 | 状态 | 阻塞项 |
|---|------|-------|------|-------|
| 1 | ~~复合 ZK proof 设计~~ | ~~关键~~ | **已解决：PLONK（Universal SRS）** | — |
| 2 | ~~椭圆曲线选择~~ | ~~关键~~ | **已解决：BN254** | — |
| 3 | 接收方状态同步 | 中等 | 部分提及 | 生产可用性 |
| 4 | 重组/回退下的 nonce 管理 | 中等 | 未提及 | 恢复机制 |
| 5 | Wrap proof 验证路径 | 中等 | 未详细说明 | 合约设计 |
| 6 | Phase 2 承诺桥接 | 高 | 已推迟 | Phase 2 |
