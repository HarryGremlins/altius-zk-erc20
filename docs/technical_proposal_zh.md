# Altius 隐私模块 — 技术方案

## 1. 项目目标

为 Altius 执行引擎实现模块化隐私层，支持保密代币转账，同时满足机构合规需求：

- **保密转账**：链上隐藏转账金额，发送方/接收方地址可见
- **分阶段实施**：Phase 1 使用 Pedersen Commitment + ZK Range Proof；Phase 2 引入 FHE 实现通用密文计算
- **机构可审计**：原生支持 Viewing Key，满足合规与监管审查需求
- **可切换隐私**：用户可在公开状态和隐私状态之间无缝切换

## 2. 核心设计决策

### 2.1 账户模型 + Wrapped Account

本项目采用**账户模型**与 Wrapped Account 架构，而非 UTXO 模型。

**理由**：UTXO / 混币模型在市场上容易产生洗钱联想（如 Tornado Cash 先例）。账户模型天然适配机构合规标准和现有 EVM 账户抽象。

**Wrapped Account 流程**：

```
A  ──wrap──▶  A'  ──保密转账──▶  B'  ──unwrap──▶  B
(公开)      (隐私)            (隐私)            (公开)
```

- **Wrap**（A → A'）：将公开代币转入隐私余额 — 链上可见
- **保密转账**（A' → B'）：隐私账户间转账 — 金额隐藏
- **Unwrap**（B' → B）：将隐私余额转回公开代币 — 链上可见

**有限隐私**：由于 wrap 和 unwrap 是透明的，本方案提供的是**金额保密性**而非完全匿名。这是 Phase 1 的已接受折衷，优先保证合规兼容性而非最大化隐私。

### 2.2 Phase 1 技术栈：Pedersen Commitment + ZK Range Proof

| 组件 | 选型 | 理由 |
|------|------|------|
| 余额表示 | **Pedersen Commitment** | 加法同态性支持链上余额更新，无需解密 |
| 有效性证明 | **ZK Range Proof** | 发送方知道明文，可在链下证明余额充足和金额为正 |
| Range Proof 方案 | **待定（Bulletproofs 或等效方案）** | 无需 trusted setup，proof 紧凑（~600 bytes） |

**Phase 1 为什么不用 FHE？**

简单转账场景中，发送方知道所有明文值（自己的余额、转账金额）。发送方可以在链下生成 ZK proof；执行引擎只需验证 proof 并对 commitment 做椭圆曲线点运算。当单一方持有全部明文时，FHE 没有必要 — FHE 的真正价值在于执行引擎需要对多方加密数据进行计算，且没有任何一方能看到对方的输入（见 Phase 2）。

### 2.3 执行环境

隐私模块运行在 **Altius 执行引擎**上 — 不是原生 Ethereum。执行引擎提供：

- 椭圆曲线运算预编译（EC 点加法、标量乘法）
- ZK proof 验证预编译
- 未来：FHE 相关预编译（Phase 2）

## 3. 系统架构

### 3.1 双层余额模型

每个用户拥有两种余额：

- **公开余额**：标准 ERC20 `balanceOf`，支持普通 `transfer`/`approve`/`transferFrom`
- **隐私余额**：以 Pedersen commitment `Com(balance, r) = balance·G + r·H` 形式存储，仅所有者可见

两种余额可以互相转化：

```
公开余额 ←→ 隐私余额（Pedersen Commitment）

wrap:   balanceOf[msg.sender] -= amount → 生成隐私 commitment
unwrap: 花费隐私余额（proof）→ balanceOf[recipient] += amount
```

### 3.2 整体结构

```
┌─────────────────────────────────────────────────────────────┐
│                      用户 / 前端                              │
│             (构造交易, 生成 ZK range proof)                    │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                  Altius 执行引擎                             │
│            — 内置隐私预编译 opcodes —                         │
│                                                             │
│  ┌───────────────┐  ┌────────────────┐  ┌────────────────┐  │
│  │ ERC20 标准功能 │  │ 隐私账户管理     │  │ ZK 验证器       │  │
│  │ (balanceOf,   │  │ (Pedersen      │  │ (Range Proof   │  │
│  │  transfer,    │  │  Commitments)  │  │  验证)          │  │
│  │  approve)     │  │                │  │                │  │
│  └───────────────┘  └────────────────┘  └────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 3.3 核心组件

#### Pedersen Commitment

每个隐私余额以 Pedersen commitment 形式存储：

```
Com(v, r) = v·G + r·H
```

- `v`：余额值
- `r`：随机致盲因子（仅所有者知晓）
- `G`、`H`：两个独立的椭圆曲线生成点（离散对数关系未知）

**加法同态性**使余额更新无需解密：

```
Com(a, r1) + Com(b, r2) = Com(a + b, r1 + r2)

// 链上转账 v 的余额更新：
com_sender'   = com_sender   - Com(v, r_v)   // EC 点减法
com_receiver' = com_receiver + Com(v, r_v)   // EC 点加法
// 执行引擎全程不接触明文
```

**安全属性**：
- **隐藏性**（Hiding）：给定 `Com(v, r)`，无法推导出 `v`（因为随机致盲因子 `r`）
- **绑定性**（Binding）：无法找到 `(v', r')` 使得 `Com(v, r) = Com(v', r')`（基于离散对数假设）

#### ZK Range Proof

发送方在链下生成 ZK proof 证明交易有效性：

1. **余额充足**：`balance >= amount`（发送方有足够资金）
2. **金额为正**：`amount > 0`（防止零值或负值转账）

proof 提交到链上，由执行引擎验证。验证者不会得知实际余额或金额 — 只知道约束被满足。

**候选方案：Bulletproofs**
- 无需 trusted setup
- Proof 大小：~600 bytes（64 位 range proof）
- 验证复杂度：对数级
- 实战验证：Monero（自 2018 年起）和 Penumbra

#### 加密备忘录（ECIES）

保密转账发生时，接收方的 commitment 在链上被更新，但接收方并不知道转账金额 `v` 和随机数 `r_v`。没有这些值，接收方无法计算新余额，也无法为后续交易生成证明。

**解决方案**：发送方使用 **ECIES**（椭圆曲线集成加密方案）加密 `(v, r_v)`，作为 calldata 附在交易中：

```
发送方加密 memo 给接收方：
  1. 生成临时密钥对：(ek, ek·G)
  2. ECDH 共享密钥：secret = ek · PK_receiver
  3. 派生对称密钥：sym_key = KDF(secret)
  4. 加密 memo：    ciphertext = AES-GCM(sym_key, [v, r_v])
  5. 附加到交易：   (ek·G, ciphertext)

接收方解密：
  1. ECDH 共享密钥：secret = sk_receiver · ek·G     // 相同的值
  2. 派生对称密钥：sym_key = KDF(secret)
  3. 解密 memo：    [v, r_v] = AES-GCM.decrypt(sym_key, ciphertext)
  4. 更新本地状态：  balance_new = balance_old + v, r_new = r_old + r_v
```

链上额外开销：约 80 bytes（32 字节压缩 EC 点 + 约 48 字节 AES-GCM 密文）。

#### 密钥派生层级

系统涉及多个密钥（EVM 签名密钥、ECIES 加密密钥、Pedersen 随机数），但用户只需管理**一个私钥**。所有其他密钥均可确定性派生：

```
EVM 私钥（唯一主密钥）
  │
  ├── ECIES 私钥 = KDF("ecies" || evm_sk)
  │     用途：加密/解密转账 memo
  │     派生一次，所有交易复用
  │
  └── Pedersen 随机数 = KDF("pedersen" || evm_sk || nonce)
        用途：每个 commitment 的致盲因子 r
        每笔交易使用递增 nonce 派生
```

**恢复能力**：用户只要拥有 EVM 私钥 + 链上交易历史，即可重建所有 ECIES 密钥和 Pedersen 随机数，完整恢复隐私余额状态。

#### Viewing Key

为满足机构合规，用户可授权审计方查看其交易：

- 所有者共享特定 commitment 的 `(value, randomness)` 对，允许审计方验证 `Com(v, r) = v·G + r·H`
- 或者共享 ECIES 私钥，允许审计方解密所有转账 memo，重建完整交易历史
- Viewing key 是**可选的**且**粒度可控的** — 用户完全控制披露什么以及向谁披露

## 4. 交易类型

### 4.1 标准 ERC20 操作

合约本身是完整的 ERC20，支持所有标准操作：`transfer`、`approve`、`transferFrom` 等。这些操作仅涉及公开余额，无需 ZK proof。

### 4.2 Wrap（公开余额 → 隐私余额）

用户将公开代币转入隐私层。

```
balanceOf[msg.sender] -= amount
→ 新 Pedersen commitment Com(amount, r) 加入发送方的隐私余额
无需 ZK proof（金额来自公开输入）
```

发送方提供 commitment `Com(amount, r)`。合约验证 commitment 格式正确并记录。

### 4.3 保密转账（隐私 → 隐私）

隐私账户间转账，金额隐藏。

```
输入：发送方当前隐私 commitment + ZK range proof + 转账 commitment
输出：更新后的发送方 commitment + 更新后的接收方 commitment

链上操作：
  com_sender'   = com_sender   - com_transfer    // EC 减法
  com_receiver' = com_receiver + com_transfer    // EC 加法
  verify(range_proof) == true                    // ZK 验证
```

**ZK proof 证明的内容**：
- 发送方知道能打开 `com_sender` 的 `(balance, r)`
- `balance >= transfer_amount`
- `transfer_amount > 0`
- `com_transfer = Com(transfer_amount, r_transfer)` 对应声称的转账 commitment

**链上可见**：发送方地址、接收方地址、更新后的 commitment
**链上隐藏**：转账金额、发送方余额、接收方余额

### 4.4 Unwrap（隐私余额 → 公开余额）

将隐私余额转回公开代币。

```
花费隐私余额（commitment + ZK proof）
→ balanceOf[recipient] += amount
Unwrap 时金额公开（公开余额是透明的）
```

## 5. 合约设计

### 5.1 核心接口

```solidity
contract ConfidentialERC20 is ERC20 {

    // --- 标准 ERC20 ---
    // transfer(), approve(), transferFrom(), balanceOf() 继承自 ERC20

    // --- 公开余额 → 隐私余额 ---
    function wrap(uint256 amount, bytes32 commitment) external;

    // --- 保密转账（隐私 → 隐私） ---
    function confidentialTransfer(
        address recipient,
        bytes32 transferCommitment,
        bytes calldata rangeProof,
        bytes calldata encryptedMemo   // ECIES 加密的 (value, randomness) 给接收方
    ) external;

    // --- 隐私余额 → 公开余额 ---
    function unwrap(
        uint256 amount,
        bytes calldata proof
    ) external;
}
```

### 5.2 合约存储

```solidity
contract ConfidentialERC20 is ERC20 {
    // 隐私余额：地址 → Pedersen commitment（以 EC 点形式存储）
    mapping(address => bytes32) public shieldedBalance;

    // ZK range proof 验证器（预编译或部署合约）
    address public verifier;
}
```

## 6. Phase 2 路线图：FHE 集成

### 6.1 何时需要 FHE

当**执行引擎需要对多方加密数据进行计算**，且没有任何一方持有全部明文时，FHE 成为必需：

| 场景 | 为什么需要 FHE | 为什么 Pedersen+ZK 不够 |
|------|--------------|----------------------|
| 加密 DEX 订单撮合 | 引擎需要比较不同用户的加密报价 | 双方都不知道对方的价格，无法生成关于比较结果的 ZK proof |
| 加密清算判断 | 引擎需要对加密数据计算 `抵押品 × 价格 < 阈值` | 用户不愿公开抵押品数量，预言机提供价格 — 多方输入 |
| 保密拍卖 | 引擎需要从加密出价中确定中标者 | 任何竞标者都不应看到其他人的出价 |

### 6.2 FHE 方案候选

| 方案 | 运算类型 | 性能特点 | 适用场景 |
|------|---------|---------|---------|
| **TFHE** | 布尔/整数运算，快速 bootstrapping | 最适合任意整数计算 | 首选候选（Zama/fhEVM） |
| **BGV/BFV** | 批量整数算术 | 适合批量操作 | 可用于批量结算 |
| **CKKS** | 近似算术 | 快但不精确 | 不适合精确余额场景 |

最终 FHE 方案选型推迟到 Phase 2 规划阶段。

### 6.3 密钥管理

FHE 需要全网密钥集：

- **公钥（pk）**：所有用户共享，用于加密
- **计算密钥（evk）**：执行引擎使用，用于同态计算
- **私钥（sk）**：通过门限秘密共享分片给验证者节点（t-of-n）

任何单个验证者都无法解密。用户通过代理重加密（Proxy Re-Encryption）读取自己的余额（网络将密文重加密到用户的个人密钥）。

### 6.4 双轨架构（Phase 2）

引入 FHE 后，每个隐私账户存储两种表示：

```
Shielded Account {
    ct_balance:  FHE.Enc(balance)              // FHE 轨道：执行引擎密文计算
    com_balance: PedersenCommit(balance, r)     // ZK 轨道：range proof 和有效性验证
}
```

两条轨道之间通过"承诺桥接"（Commitment Bridge）确保一致性，具体设计将在 Phase 2 详细规划。

## 7. 技术栈总结

| 层级 | Phase 1 | Phase 2 |
|------|---------|---------|
| 余额表示 | Pedersen Commitment | Pedersen + FHE 密文 |
| 有效性证明 | ZK Range Proof（Bulletproofs 或等效方案） | ZK Range Proof + FHE 计算 |
| 承诺方案 | Pedersen（`v·G + r·H`） | Pedersen + FHE 方案待定 |
| 数据模型 | 账户模型（Wrapped Account） | 账户模型（双轨） |
| 执行层 | Altius 引擎 + EC 预编译 | Altius 引擎 + EC + FHE 预编译 |
| 合规 | Viewing Key | Viewing Key + 代理重加密 |
