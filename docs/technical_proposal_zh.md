# ZK ERC20 技术方案

## 1. 项目目标

实现一个基于零知识证明的 ERC20 代币协议，支持：

- **可选隐私**：发送者可自由选择隐藏或公开转账金额与收款人
- **纯 ZK 方案**：所有隐私功能基于零知识证明实现，不依赖同态加密（FHE）
- **Parallel EVM 友好**：架构层面为未来并行 EVM 执行器优化留出空间

## 2. 核心设计决策

### 2.1 UTXO 模型（非账户模型）

本项目采用 UTXO（未花费交易输出）模型而非账户模型，核心理由：

**并行性**：账户模型下，多笔指向同一收款人的交易会竞争同一 state slot，导致 ZK proof 基于过期状态生成而失效。UTXO 模型中每个 UTXO 是独立的 state object，交易间无共享状态，天然可并行。

```
账户模型：A → C, B → C 竞争 C.balance → proof 冲突
UTXO 模型：A 创建 UTXO_c1, B 创建 UTXO_c2 → 互不干扰
```

**隐私性**：UTXO + Nullifier 模式天然适配隐私场景——消费 UTXO 时通过 nullifier 标记已花费，无需暴露具体是哪个 UTXO。

### 2.2 ZK 技术栈：Noir + Barretenberg

| 选型 | 选择 | 理由 |
|------|------|------|
| 电路语言 | **Noir** | Rust-like 语法，开发效率高，编译器自动生成约束 |
| 约束系统 | **PLONKish (ACIR)** | 支持 lookup table，表达能力强 |
| 证明后端 | **Barretenberg (UltraPlonk/UltraHonk)** | 无 per-circuit trusted setup，使用 universal SRS |

## 3. 系统架构

### 3.1 双层余额模型

本合约**本身就是一个 ERC20 代币**，同时内置隐私层。每个用户拥有两种余额：

- **公开余额**：标准 ERC20 `balanceOf`，支持普通 `transfer`/`approve`/`transferFrom`
- **隐私余额**：以 UTXO 形式存在于 Merkle tree 中，仅所有者可见

两种余额可以互相转化：

```
公开余额 ←→ 隐私余额（UTXO）

shield:   balanceOf[msg.sender] -= amount → 生成新 UTXO commitment
unshield: 消费 UTXO（nullifier + proof）→ balanceOf[recipient] += amount
```

### 3.2 整体结构

```
┌─────────────────────────────────────────────────────┐
│                    用户 / 前端                        │
│           (构造交易, 生成 ZK proof)                   │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│            ZK ERC20 合约 (Solidity)                  │
│            — 本身就是 ERC20 代币 —                    │
│                                                     │
│  ┌───────────────┐  ┌──────────┐  ┌──────────────┐  │
│  │ ERC20 标准功能 │  │ UTXO 管理 │  │ ZK 验证器     │  │
│  │ (balanceOf,   │  │(Merkle   │  │(Noir         │  │
│  │  transfer,    │  │ Tree +   │  │ Verifier)    │  │
│  │  approve)     │  │Nullifier)│  │              │  │
│  └───────────────┘  └──────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────┘
```

### 3.3 核心组件

#### UTXO（Note）

每个 UTXO 包含：

```
Note {
    value: u64,           // 金额
    owner: Field,         // 所有者公钥（或其哈希）
    randomness: Field,    // 随机致盲因子
}
```

链上存储的是 Note 的承诺（commitment），原始数据仅所有者知晓。承诺方案待定（Poseidon Hash 或 Pedersen Commitment）。

#### Nullifier（作废标记）

消费 UTXO 时公开 nullifier，合约检查其未被使用过。nullifier 无法被反向关联到具体 commitment，实现了花费隐私。

#### Merkle Tree（承诺树）

- 链上维护一棵固定深度的 append-only Merkle tree（如深度 20，支持 ~100 万个 UTXO）
- 仅支持追加，不支持删除（已花费的 UTXO 通过 nullifier set 标记）

## 4. 交易类型

### 4.1 标准 ERC20 操作

合约本身是完整的 ERC20，支持所有标准操作：`transfer`、`approve`、`transferFrom` 等。这些操作仅涉及公开余额，无需 ZK proof。

### 4.2 Shield（公开余额 → 隐私 UTXO）

用户将自己的公开余额转入隐私层。

```
balanceOf[msg.sender] -= amount
→ 新 commitment 插入 Merkle tree
无需 ZK proof（输入来自公开余额）
```

### 4.3 Private Transfer（隐私转账）

消费一个或多个已有 UTXO，生成新的 UTXO 给收款人和自己（找零）。

```
输入：nullifier(s) + ZK proof
输出：新 commitment(s) 插入 Merkle tree
```

**隐私维度独立可选**：

| 金额 | 收款人 | 描述 |
|------|--------|------|
| 隐藏 | 隐藏 | 全隐私模式，链上只有 commitment + nullifier |
| 公开 | 隐藏 | 链上附带明文 value，收款人不可见 |
| 隐藏 | 公开 | 链上附带明文 recipient，金额不可见 |
| 公开 | 公开 | 全透明，但仍经过 UTXO 流程 |

两个隐私维度（金额、收款人）**相互独立**，发送者可自由组合。实现方式：在交易 calldata 中附带可选的明文字段，ZK 电路内增加一致性约束（若选择公开，则证明公开值与隐私值一致）。

### 4.4 Unshield（隐私 UTXO → 公开余额）

消费隐私 UTXO，转回公开余额。

```
消费 UTXO（nullifier + ZK proof）
→ balanceOf[recipient] += amount
```

## 5. 合约设计（Solidity）

### 5.1 核心接口

```solidity
contract ZkERC20 is ERC20 {

    // --- 标准 ERC20 ---
    // transfer(), approve(), transferFrom(), balanceOf() 等继承自 ERC20

    // --- 公开余额 → 隐私 UTXO ---
    function shield(uint256 amount, bytes32 commitment) external;

    // --- 隐私转账（UTXO → UTXO） ---
    function privateTransfer(
        bytes32[] calldata nullifiers,
        bytes32[] calldata newCommitments,
        bytes calldata proof
    ) external;

    // --- 隐私 UTXO → 公开余额 ---
    function unshield(
        uint256 amount,
        address recipient,
        bytes32 nullifier,
        bytes calldata proof
    ) external;
}
```

### 5.2 合约存储

```solidity
contract ZkERC20 is ERC20 {
    // Merkle tree for commitments
    mapping(uint256 => bytes32) public filledSubtrees;
    mapping(uint256 => bytes32) public roots;
    uint32 public nextLeafIndex;

    // Nullifier set
    mapping(bytes32 => bool) public nullifierUsed;

    // Noir proof verifier (auto-generated)
    IVerifier public verifier;
}
```

## 6. 未来优化方向

### 6.1 Parallel EVM 优化

UTXO 模型的天然优势：

- 每笔交易读写的 state 不重叠（独立的 nullifier + 新 commitment）
- Merkle tree 的 append 操作可以批量化
- 适合 access-list 预声明，便于并行调度

后续可能的优化：
- Batch proof verification（多笔 proof 批量验证）
- Merkle tree 分片（多棵子树并行追加）
- Nullifier set 分桶（减少写冲突）

### 6.2 其他

- 递归证明（Recursive proofs）：将多笔交易的 proof 聚合为一个
- 合规接口：可选的 view key 机制，允许审计方查看特定交易
- 跨链桥接：隐私 UTXO 的跨链转移

## 7. 技术栈总结

| 层级 | 技术 |
|------|------|
| ZK 电路 | Noir |
| 证明系统 | Barretenberg (UltraPlonk / UltraHonk) |
| 承诺方案 | 待定（Poseidon Hash 或 Pedersen Commitment） |
| 数据模型 | UTXO (Note) + Nullifier |
| 智能合约 | Solidity (ERC20) |
| Merkle Tree | 链上 append-only, 深度待定 |
