# Ethereum 2.0 Phase 0 -- The Beacon Chain

**注意**:对研究人员和开发人员而言，本文档处在变化中。它反映了最近的规范变化，并重要于Python的概念验证实现[[python-poc]()]。

## 引言

本文档代表以太坊2.0的阶段0——信标链——规范。

以太坊2.0的核心是一个称为“信标链”的系统链。新标链存储和管理所有[验证者]()的注册。在以太坊2.0的初始部署阶段，成为一名[验证者]()的唯一机制是向以太坊1.0的充值合约做一个单向的ETH交易。当以太坊1.0的充值收据被信标链处理，激活成为一名[验证者]，获取到激活余额，并经过一排队过程。退出要么是自愿的，要么是作为对其不规范行为的惩罚而强制完成的。

信标链上的主要负载源是"attestation"。Attestation是对分片block的(数据)可用性投票，同时对信标链block的权益证明投票。对同一个分片block而言，足够多的attestation会创建一个“crosslink”，确认直到该分片block的分片segment进入信标链。Crosslinks还充当异步跨分片通信的基础设施。

## 记号

除非另有说明，以该风格显示的代码被理解为Python中定义的算法。只要行为和提供的算法一致，可以使用任何代码和编程语言实现这些算法。

## 术语

- Validator——Casper/Sharding共识系统的一名参与者(participant)。您可以通过充值32ETH到Casper机制成为一名validotor。
- Active validator——当前参与协议的一名validator，Casper机制期待活跃的验证者创建block和证明(attest)区块，crosslink以及其他的共识对象。
- Committee——[活跃验证者]()中(伪)随机抽样的一个子集。当一个委员会(committe)被集体提及时，如“该委员会证明X”，这被假定为“该委员会的某个子集包含足够的[验证者]()，协议认可其代表委员会”。
- Proposer——创建一个信标链block的[验证者]()。
- Attester——一名验证者，委员会的一员，需要对信标链的block上进行签名，同时创建到特定分片链(shard chain)上的最近的分片block的一个link(crosslink)。
- Beacon chain——核心的PoS链，分片系统的基础(base)。
- Shard chain——用户交易发生和存储账户数据的许多链的一个。
- Block root——信标链block或分片block中32字节的默克尔根。以前称为“block hash”。
- Crosslink——来自委员会的一组签名，证明分片链中的一个block可以包含在信标链中，Crosslink是信标链“了解”分片链更新状态的主要手段。
- Slot——一段SLOT_DURATIONS秒，在此期间，一个提案者能够创建信标链block以及一些attesters有能力进行证明(attestation)。
- Epoch——
- Finalized，justified——参见Casper FFG终局化[casper-ffg]。
- Withdrawal period——一名验证者退出和该验证者余额可被提现的时隙(slot)数。
- Genesis time——在时隙0，创世信标链区块的Unix时间。

## 常数

### Misc

| 名称(Name)                   | 值(Value)     | 单位(Unit)   |
| ---------------------------- | ------------- | ------------ |
| SHARD_COUNT                  | 2**10(=1,024) | shards       |
| TARGET_COMMITTEE_SIZE        | 3**7(=128)    | validators   |
| EJECTION_BALANCE             | 2**4(=16)     | ETH          |
| MAX_BALANCE_CHURN_QUOTIENT   | 2**5(=32)     | -            |
| GWEI_PER_ETH                 | 10**9         | Gwei/ETH     |
| BEACON_CHAIN_SHARD_NUMBER    | 2**64-1       | -            |
| MAX_CAPSER_VOTES             | 2**10(=1,024) | votes        |
| LATEST_BLOCK_ROOTS_LENGTH    | 2**13(=8,192) | block roots  |
| LATEST_RANDAO_MIXES_LENGTH   | 2**13(=8192)  | randao mixes |
| LATEST_PENALIZED_EXIT_LENGTH | 2**13(=8,192) | epochs       |

为了crosslinks的安全性，TARGET_COMMITTEE_SIZE超过了推荐的最先委员会大小111。有了充足的活跃验证者(至少EPOCH_LENGTH*TARGET_COMMITTEE_SIZE)，混洗算法保证委员会大小至少TARGET_COMMITTEE_SIZE。（无偏的随机性将改进委员会的鲁棒性并降低最小安全委员会大小）。

### 充值合约(Deposit contract)

| 名称(Name)                  | 值(Value) | 单位(Unit) |
| --------------------------- | --------- | ---------- |
| DEPOSIT_CONTRACT_ADDRESS    | TBD       |            |
| DEPOSIT_CONTRACT_TREE_DEPTH | 2**5(=32) | -          |
| MIN_DEPOSIT                 | 2**0(=1)  | ETH        |
| MAX_DEPOSIT                 | 2**5(=32) | ETH        |

### 初始值(Initial values)

| 名称(Name)                 | 值(Value)               |
| -------------------------- | ----------------------- |
| GENESIS_FORK_VERSION       | 0                       |
| GENESIS_SLOT               | 0                       |
| GENESIS_START_SHARD        | 0                       |
| FAR_FURURE_SLOT            | 2**64-1                 |
| ZERO_HASH                  | bytes32(0)              |
| EMPTY_SIGNATURE            | [bytes48(0),bytes48(0)] |
| BLS_WITHDRAWAL_PREFIX_BYTE | bytes1(0)               |

### 时间参数

| 名称(Name)                       | 值(Value)      | 单位(Unit) | 持续时间(Duration) |
| -------------------------------- | -------------- | ---------- | ------------------ |
| SLOT_DURATION                    | 6              | seconds    | 6 seconds          |
| MIN_ATTESTATION_INCLUSTION_DELAY | 2**2(=4)       | slots      | 24 seconds         |
| EPOCH_LENGTH                     | 2**6(=64)      | slots      | 6.4 minutes        |
| SEED_LOOKHEAD                    | 2**6(=64)      | slots      | 6.4 minutes        |
| ENTRY_EXIT_DELAY                 | 2**8(=256)     | slots      | 25.6 minutes       |
| DEPOSIT_ROOT_VOTING_PERIOD       | 2**10(=1,024)  | slots      | ~ 1.7 hours        |
| MIN_VALIDATOR_WITHDRAWL_TIME     | 2**14(=16,384) | slots      | ~ 27 hours         |

### 奖惩指数

| 名称(Name)                    | 值(Value)           |
| ----------------------------- | ------------------- |
| BASE_REWARD_QUOTIENT          | 2**10(=1,024)       |
| WHISTLEBLOWER_REWARD_QUOTIENT | 2**9 (= 512)        |
| INCLUDER_REWARD_QUOTIENT      | 2**3(= 8)           |
| INACTIVITY_PENALTY_QUOTIENT   | 2**64(= 16,777,216) |

> 上述这些参数可以在shared/params/config.go文件中找到。