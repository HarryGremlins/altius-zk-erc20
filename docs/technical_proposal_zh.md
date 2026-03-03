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
│             (构造交易, 生成 ZK range proof)                   │
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
com_sender'   = com_sender   - Com(v, r_tx)   // EC 点减法
com_receiver' = com_receiver + Com(v, r_tx)   // EC 点加法
// 执行引擎全程不接触明文
```

其中 `r_tx` 是转账 commitment `Com(v, r_tx)` 的**新鲜随机致盲因子**，由发送方的私钥确定性派生：`r_tx = KDF("pedersen" || evm_sk || nonce)`（详见下文密钥派生层级）。致盲因子对 commitment 的隐藏性至关重要 — 如果没有 `r_tx`，commitment 将退化为 `v·G`，任何人都可以通过穷举所有可能的金额值来反推转账金额。转账完成后，接收方需要 `v` 和 `r_tx` 才能计算新的明文余额并为后续交易生成证明；这些值通过加密备忘录传递给接收方（详见下文加密备忘录）。

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

保密转账发生时，接收方的 commitment 在链上被更新，但接收方并不知道转账金额 `v` 和随机数 `r_tx`。没有这些值，接收方无法计算新余额，也无法为后续交易生成证明。

**解决方案**：发送方使用 **ECIES**（椭圆曲线集成加密方案）加密 `(v, r_tx)`，作为 calldata 附在交易中：

```
发送方加密 memo 给接收方：
  1. 从链上注册表查找接收方的 ECIES 公钥（详见下文账户注册）
  2. 生成临时密钥对：(ek, ek·G)
  3. ECDH 共享密钥：secret = ek · PK_ecies_receiver
  4. 派生对称密钥：sym_key = KDF(secret)
  5. 加密 memo：    ciphertext = AES-GCM(sym_key, [v, r_tx])
  6. 附加到交易：   (ek·G, ciphertext)

接收方解密：
  1. ECDH 共享密钥：secret = sk_ecies_receiver · ek·G   // 相同的值
  2. 派生对称密钥：sym_key = KDF(secret)
  3. 解密 memo：    [v, r_tx] = AES-GCM.decrypt(sym_key, ciphertext)
  4. 更新本地状态：  balance_new = balance_old + v, r_new = r_old + r_tx
```

注意：`PK_ecies_receiver` 是接收方的 **ECIES 公钥**（由其 ECIES 私钥派生），而非 EVM 公钥。每个用户在加入隐私系统时需在链上注册其 ECIES 公钥（详见下文账户注册）。

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

**Nonce 管理**：nonce 在客户端追踪，每次隐私操作（wrap 或保密转账）递增一次。nonce 不存储在链上 — 其序列由用户的链上交易历史隐式确定。

**恢复能力**：用户只需 EVM 私钥，即可扫描链上所有隐私操作（wrap 和保密转账），按时间顺序重放以重建 nonce 序列，进而重新派生所有 ECIES 密钥和 Pedersen 随机数 — 完整恢复隐私余额状态。

#### 可审计性

为满足机构合规，用户可授权审计方查看其交易。有两种不同模式：

**选择性披露**（逐笔交易）：用户直接共享特定 commitment 的 `(value, randomness)` 对，允许审计方验证 `Com(v, r) = v·G + r·H`。这是简单的数据共享 — 无需特殊密钥或基础设施。适用于一次性或临时审计。

**Viewing Key**（全量访问）：用户将其 **ECIES 私钥**共享给审计方。该单一密钥允许审计方解密用户所有收到的转账加密 memo（过去和未来），无需用户逐笔配合即可重建完整交易历史。这是真正的"viewing key" — 适用于监管机构或机构托管方的持续合规监控。

两种模式均为**可选** — 用户完全控制披露什么以及向谁披露。选择性披露将暴露范围限定于特定交易；viewing key 提供持续监控能力，应仅与受信任的审计方共享。

#### 账户注册

用户在参与隐私系统前，需调用 `register` 函数：

1. **将用户的 ECIES 公钥存储在链上** — 使发送方在构造保密转账时可以查找接收方的加密公钥（详见上文加密备忘录）
2. **将用户的隐私余额初始化**为单位元（无穷远点），即 `Com(0, 0)` — 零余额、零致盲因子的 commitment

注册是接收保密转账的前提条件 — 如果接收方未注册（链上无 ECIES 公钥），`confidentialTransfer` 将回退。仅使用标准 ERC20 功能（公开余额）的用户无需注册。

## 4. 交易类型

### 4.1 标准 ERC20 操作

合约本身是完整的 ERC20，支持所有标准操作：`transfer`、`approve`、`transferFrom` 等。这些操作仅涉及公开余额，无需 ZK proof。

### 4.2 Wrap（公开余额 → 隐私余额）

用户将公开代币转入隐私层。

```
balanceOf[msg.sender] -= amount
→ 新 Pedersen commitment Com(amount, r) 加入发送方的隐私余额
```

发送方提供：
- `amount`：要 wrap 的公开金额
- `commitment`：链下计算的 Pedersen commitment `Com(amount, r)`
- `proof`：一个 ZK proof，证明该 commitment 打开后对应声称的金额（即证明者知道 `r` 使得 `commitment == amount·G + r·H`）

合约验证 proof 以防止通胀攻击 — 若无验证，恶意用户可以 wrap 10 个代币却承诺任意更大的值。验证通过后，合约从公开余额扣减 `amount`，并通过 EC 点加法将 commitment 加入发送方的隐私余额：`shieldedBalance[sender]' = shieldedBalance[sender] + commitment`。

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

将隐私余额转回公开代币。Unwrap 时金额公开（公开余额是透明的）。

发送方提供：
- `amount`：要 unwrap 的金额（变为公开）
- `transferCommitment`：被花费的 Pedersen commitment `Com(amount, r_unwrap)`
- `proof`：ZK proof，证明以下内容

**ZK proof 证明的内容**：
- 发送方知道能打开 `com_sender` 的 `(balance, r)`
- `balance >= amount`
- `transferCommitment == Com(amount, r_unwrap)` 对应某个证明者知道的 `r_unwrap`

**链上操作**：
```
verify(proof) == true                                         // ZK 验证
com_sender'    = com_sender - transferCommitment              // EC 减法
balanceOf[msg.sender] += amount                               // 增加公开余额
```

## 5. 合约设计

### 5.1 核心接口

```solidity
struct ECPoint {
    uint256 x;
    uint256 y;
}

contract ConfidentialERC20 is ERC20 {

    // --- 标准 ERC20 ---
    // transfer(), approve(), transferFrom(), balanceOf() 继承自 ERC20

    // --- 注册 ECIES 公钥并初始化隐私余额 ---
    function register(bytes calldata eciesPublicKey) external;

    // --- 公开余额 → 隐私余额 ---
    function wrap(
        uint256 amount,
        ECPoint calldata commitment,
        bytes calldata proof           // ZK proof：证明 commitment 打开后对应 amount
    ) external;

    // --- 保密转账（隐私 → 隐私） ---
    function confidentialTransfer(
        address recipient,
        ECPoint calldata transferCommitment,
        bytes calldata rangeProof,
        bytes calldata encryptedMemo   // ECIES 加密的 (value, randomness) 给接收方
    ) external;

    // --- 隐私余额 → 公开余额 ---
    function unwrap(
        uint256 amount,
        ECPoint calldata transferCommitment,
        bytes calldata proof
    ) external;
}
```

### 5.2 合约存储

```solidity
contract ConfidentialERC20 is ERC20 {
    // 椭圆曲线点表示 Pedersen commitment
    // ECPoint { uint256 x; uint256 y; }

    // 隐私余额：地址 → Pedersen commitment（以 EC 点形式存储）
    mapping(address => ECPoint) public shieldedBalance;

    // ECIES 公钥：地址 → 公钥（用于加密 memo 传递）
    mapping(address => bytes) public eciesPublicKeys;

    // ZK proof 验证器（预编译或部署合约）
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
| 合规 | 选择性披露 + Viewing Key | Viewing Key + 代理重加密 |

## 附录 A：符号参考表

### Pedersen Commitment

| 符号 | 类型 | 定义 |
|------|------|------|
| `v` | 标量 | 被承诺的明文值（如余额或转账金额） |
| `r` | 标量 | 随机致盲因子 — 保证 commitment 的隐藏性；仅所有者知晓 |
| `G` | EC 点 | 椭圆曲线上的第一个生成点（基点） |
| `H` | EC 点 | 第二个独立生成点；`H` 相对于 `G` 的离散对数必须未知 |
| `Com(v, r)` | EC 点 | Pedersen commitment 函数：`Com(v, r) = v·G + r·H` |
| `com_sender` | EC 点 | 发送方当前的隐私余额 commitment（交易前） |
| `com_sender'` | EC 点 | 发送方更新后的隐私余额 commitment（交易后） |
| `com_receiver` | EC 点 | 接收方当前的隐私余额 commitment（交易前） |
| `com_receiver'` | EC 点 | 接收方更新后的隐私余额 commitment（交易后） |
| `com_transfer` | EC 点 | 转账 commitment：`Com(transfer_amount, r_transfer)` — 表示被转出的金额 |
| `r_tx` | 标量 | 转账 commitment 的新鲜随机致盲因子，派生方式：`KDF("pedersen" \|\| evm_sk \|\| nonce)` |
| `r_unwrap` | 标量 | Unwrap 转账 commitment 的致盲因子 |

### 密钥派生与加密

| 符号 | 类型 | 定义 |
|------|------|------|
| `evm_sk` | 标量 | 用户的 EVM 私钥 — 作为唯一主密钥，所有派生密钥均由此生成 |
| `nonce` | 整数 | 客户端追踪的递增计数器；每次隐私操作（wrap 或保密转账）递增一次 |
| `KDF(·)` | 函数 | 密钥派生函数 — 从输入材料确定性地派生密钥 |
| `sk_ecies` | 标量 | ECIES 私钥，派生方式：`KDF("ecies" \|\| evm_sk)` — 用于解密转账 memo |
| `PK_ecies` | EC 点 | ECIES 公钥，由 `sk_ecies` 派生 — 在链上注册，供发送方加密 memo |
| `ek` | 标量 | 发送方为单次 ECIES 加密生成的临时私钥 |
| `ek·G` | EC 点 | 临时公钥，附加在交易中；允许接收方计算共享密钥 |
| `secret` | 标量 | ECDH 共享密钥：发送方计算 `ek · PK_ecies_receiver`，接收方计算 `sk_ecies_receiver · ek·G` |
| `sym_key` | 字节 | 从共享密钥派生的对称加密密钥：`sym_key = KDF(secret)` |
| `AES-GCM(key, plaintext)` | 函数 | 认证加密函数 — 对 memo `[v, r_tx]` 进行加密并提供完整性保护 |

### ZK Range Proof

| 符号 | 类型 | 定义 |
|------|------|------|
| `balance` | 标量 | 发送方的隐私余额明文（私有，仅发送方知晓） |
| `amount` | 标量 | 转账或 unwrap 金额（保密转账中为私有，unwrap 中为公开） |
| `proof` / `rangeProof` | 字节 | 提交到链上的 ZK proof；证明 `balance >= amount` 且 `amount > 0`，不泄露任何一个值 |
| `transferCommitment` | EC 点 | Unwrap 中被花费的 Pedersen commitment：`Com(amount, r_unwrap)` |
| `commitment` | EC 点 | Wrap 中创建的 Pedersen commitment：`Com(amount, r)` |

### 合约存储

| 符号 | 类型 | 定义 |
|------|------|------|
| `ECPoint` | 结构体 | `{ uint256 x; uint256 y }` — 表示一个椭圆曲线点（Pedersen commitment） |
| `shieldedBalance[addr]` | 映射 | `address → ECPoint` — 存储每个用户的隐私余额（Pedersen commitment 形式） |
| `eciesPublicKeys[addr]` | 映射 | `address → bytes` — 存储每个用户注册的 ECIES 公钥 |
| `balanceOf[addr]` | 映射 | `address → uint256` — 标准 ERC20 公开余额（继承） |
| `verifier` | 地址 | ZK proof 验证器地址（预编译或已部署合约） |

### FHE — Phase 2

| 符号 | 类型 | 定义 |
|------|------|------|
| `pk` | FHE 密钥 | 全网 FHE 公钥 — 所有用户共用，用于加密；只有同一 `pk` 加密的密文才能互相运算 |
| `evk` | FHE 密钥 | 计算密钥 — 执行引擎使用，在密文上执行同态运算，无需看到明文 |
| `sk` | FHE 密钥 | FHE 私钥 — 通过门限秘密共享分片给验证者节点（`t`-of-`n`）；任何单个验证者无法解密 |
| `t`-of-`n` | 整数 | 门限参数：需 `n` 个验证者中的 `t` 个合作才能解密 |
| `FHE.Enc(v)` | 密文 | 使用全网公钥 `pk` 对值 `v` 的 FHE 加密 |
| `ct_balance` | 密文 | 隐私账户中 FHE 加密的余额（FHE 轨道） |
| `com_balance` | EC 点 | 同一余额的 Pedersen commitment（ZK 轨道）；必须与 `ct_balance` 保持一致 |
