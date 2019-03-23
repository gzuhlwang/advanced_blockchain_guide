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
- Active validator——当前参与协议的一名validator，Casper机制期待活跃的验证者创建block和证明(attest)区块，对crosslink以及其他的共识对象进行投票。
- Committee——[活跃验证者]()中(伪)随机抽样的一个子集。当一个委员会(committe)被集体提及时，如“该委员会证明X”，这被假定为“该委员会的某个子集包含足够的[验证者]()，协议认可其代表委员会”。【笔者注：超过2/3的voting power的验证者们】
- Proposer——创建一个信标链block的[验证者]()。
- Attester——一名验证者，委员会的一员，需要对信标链的block上进行签名，同时创建到特定分片链(shard chain)上的最近的分片block的一个link(crosslink)。
- Beacon chain——核心的PoS链，分片系统的基础(base)。
- Shard chain——用户交易发生和存储账户数据的许多链的一个。
- Block root——信标链block或分片block中32字节的默克尔根。以前称为“block hash”。
- Crosslink——来自委员会的一组签名，证明分片链中的一个block可以包含在信标链中，Crosslink是信标链“了解”分片链更新状态的主要手段。
- Slot——一段SLOT_DURATIONS秒，在此期间，一个提案者能够创建信标链block以及一些attesters有能力进行证明(attestation)。【笔者注：类似eth1.0中的block number】
- Epoch——对齐slot的一段时间，在这期间所有的验证人只有一次机会做出attestation。【笔者注：attestation可以视作投票(vote)】
- Finalized，justified——参见Casper FFG终局化[casper-ffg]。
- Withdrawal period——一名验证者退出和该验证者余额可被提现的时隙(slot)数。
- Genesis time——在时隙0，创世信标链区块的Unix时间戳。

## 常数

### Misc

| 名称(Name)                   | 值(Value)      | 单位(Unit)   |
| ---------------------------- | -------------- | ------------ |
| SHARD_COUNT                  | 2**10(= 1,024) | shards       |
| TARGET_COMMITTEE_SIZE        | 3**7(= 128)    | validators   |
| EJECTION_BALANCE             | 2**4(=16)      | ETH          |
| MAX_BALANCE_CHURN_QUOTIENT   | 2**5(= 32)     | -            |
| GWEI_PER_ETH                 | 10**9          | Gwei/ETH     |
| BEACON_CHAIN_SHARD_NUMBER    | 2**64-1        | -            |
| MAX_CAPSER_VOTES             | 2**10(= 1,024) | votes        |
| LATEST_BLOCK_ROOTS_LENGTH    | 2**13(= 8,192) | block roots  |
| LATEST_RANDAO_MIXES_LENGTH   | 2**13(= 8192)  | randao mixes |
| LATEST_PENALIZED_EXIT_LENGTH | 2**13(= 8,192) | epochs       |

为了crosslinks的安全性，`TARGET_COMMITTEE_SIZE`超过了[建议的最小委员会大小111](https://vitalik.ca/files/Ithaca201807_Sharding.pdf)。有了足够的活跃验证者(至少`EPOCH_LENGTH * TARGET_COMMITTEE_SIZE`)，混洗算法(shuffling algorithm)保证委员会规模至少为`TARGET_COMMITTEE_SIZE`。（可验证延迟函数(VDF)具有的无偏随机性将改进委员会的鲁棒性并降低安全的最小安全委员会规模）。

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

| 名称(Name)                       | 值(Value)       | 单位(Unit) | 持续时间(Duration) |
| -------------------------------- | --------------- | ---------- | ------------------ |
| SLOT_DURATION                    | 6               | seconds    | 6 seconds          |
| MIN_ATTESTATION_INCLUSTION_DELAY | 2**2(= 4)       | slots      | 24 seconds         |
| EPOCH_LENGTH                     | 2**6(= 64)      | slots      | 6.4 minutes        |
| SEED_LOOKAHEAD                   | 2**6(= 64)      | slots      | 6.4 minutes        |
| ENTRY_EXIT_DELAY                 | 2**8(= 256)     | slots      | 25.6 minutes       |
| DEPOSIT_ROOT_VOTING_PERIOD       | 2**10(= 1,024)  | slots      | ~ 1.7 hours        |
| MIN_VALIDATOR_WITHDRAWL_TIME     | 2**14(= 16,384) | slots      | ~ 27 hours         |

### 奖惩指数

| 名称(Name)                    | 值(Value)           |
| ----------------------------- | ------------------- |
| BASE_REWARD_QUOTIENT          | 2**5(= 32)          |
| WHISTLEBLOWER_REWARD_QUOTIENT | 2**9 (= 512)        |
| INCLUDER_REWARD_QUOTIENT      | 2**3(= 8)           |
| INACTIVITY_PENALTY_QUOTIENT   | 2**64(= 16,777,216) |

### 状态标记

| 名称(Name)     | 值(Value)    |
| -------------- | ------------ |
| INITIATED_EXIT | 2**0 （=1）  |
| WITHDRAWABLE   | 2**1    (=2) |

### 每块最大运算(Max operations per block)

| 名称(Name)             | 值(Value)    |
| ---------------------- | ------------ |
| MAX_PROPOSER_SLASHINGS | 2**4 (= 16)  |
| MAX_ATTESTER_SLASHINGS | 2**0 (= 1)   |
| MAX_ATTESTATIONS       | 2**7 (= 128) |
| MAX_DEPOSITS           | 2**4 (= 16)  |
| MAX_EXITS              | 2**4 (= 16)  |

### 签名域

| 名称(Name)         | 值(Value) |
| ------------------ | --------- |
| DOMAIN_DEPOSIT     | 0         |
| DOMAIN_ATTESTATION | 1         |
| DOMAIN_PROPOSAL    | 2         |
| DOMAIN_EXIT        | 3         |
| DOMAIN_RANDAO      | 4         |

> 上述这些参数可以在shared/params/config.go文件中找到。

## 数据结构

下列的数据结构被定义为[SimpleSerialize(SSZ)](https://github.com/ethereum/eth2.0-specs/blob/master/specs/simple-serialize.md)对象。

### Beacon chain操作

Proposer slashings(不妨翻译为罚没提案者,这是beacon chain的一个操作)

`ProposerSlashing`(这是数据结构)

```
{
    # Proposer index  
    'proposer_index': 'uint64',
    # First proposal data 
    'proposal_data_1': ProposalSignedData,
    # First proposal signature
    'proposal_signature_1': 'bytes96',
    # Second proposal data
    'proposal_data_2': ProposalSignedData,
    # Second proposal signature
    'proposal_signature_2': 'bytes96',
}
```

[笔者注]：该结构有两个相同类型的变量proposal_data_1和proposal_data_2，它们都属于ProposalSignedData类型。

Attester slashings(不妨翻译为罚没证明人，这是beacon chain的一个操作)

`AttesterSlashings`(这是数据结构)

```
{
    # First slashable attestation
    'slashable_attestation_1': SlashableAttestation,
    # Second slashable attestation
    'slashable_attestation_2': SlashableAttestation,
}
```

[笔者注]：该结构中有两个相同类型的变量slashable_attestation_1和slashable_attestation_2，它们都属于SlashableAttestation类型。

`SlashableAttestation`(这是数据结构，可罚没的证明，即double vote或surround vote)

```
{
    # Validator indices  验证者序号
    'validator_indices': ['uint64'],
    # Attestation data   证明数据
    'data': AttestationData,
    # Custody bitfield
    'custody_bitfield': 'bytes',
    # Aggregate signature
    'aggregate_signature': 'bytes96',
}
```

Attestations(这是一个操作)

`Attestation`(这是一个数据结构)

```
{
    # Attester aggregation bitfield
    'aggregation_bitfield': 'bytes',
    # Attestation data
    'data': AttestationData,
    # Custody bitfield
    'custody_bitfield': 'bytes',
    # BLS aggregate signature
    'aggregate_signature': 'bytes96',
}
```

`AttestationData`(这是数据结构，就是casper ffg论文中的vote消息)

```
{
    # Slot number   信标链区块号
    'slot': 'uint64', 
    # Shard number  分片序号
    'shard': 'uint64',
    # Hash of root of the signed beacon block  
    'beacon_block_root': 'bytes32',
    # Hash of root of the ancestor at the epoch boundary
    'epoch_boundary_root': 'bytes32',
    # Shard block's hash of root
    'shard_block_root': 'bytes32',
    # Last crosslink's hash of root
    'latest_crosslink_root': 'bytes32',
    # Last justified epoch in the beacon state  信标链中上一个justified的epoch
    'justified_epoch': 'uint64',
    # Hash of the last justified beacon block
    'justified_block_root': 'bytes32',
}
```

##### `AttestationDataAndCustodyBit`

```
{
    # Attestation data
    'data': AttestationData,
    # Custody bit
    'custody_bit': 'bool',
}
```

Deposit(这是一个操作)

`Deposit`

```
{
    # Branch in the deposit tree  分支
    'branch': ['bytes32'],
    # Index in the deposit tree   树中的位置
    'index': 'uint64',
    # Data                        充值数组
    'deposit_data': DepositData,
}
```

`DepositData`

```
{
    # Amount in Gwei
    'amount': 'uint64',
    # Timestamp from deposit contract
    'timestamp': 'uint64',
    # Deposit input
    'deposit_input': DepositInput,
}
```

`DepositInput`

```
{
    # BLS pubkey
    'pubkey': 'bytes48',
    # Withdrawal credentials    提现凭证
    'withdrawal_credentials': 'bytes32',
    # A BLS signature of this `DepositInput`
    'proof_of_possession': 'bytes96',
}
```

Exits(这是一个操作)

`Exit`

```
{
    # Minimum epoch for processing exit
    'epoch': 'uint64',
    # Index of the exiting validator
    'validator_index': 'uint64',
    # Validator signature
    'signature': 'bytes96',
}
```

`Transfer`

```
{
    # Sender index
    'sender': 'uint64',
    # Recipient index
    'recipient': 'uint64',
    # Amount in Gwei
    'amount': 'uint64',
    # Fee in Gwei for block proposer
    'fee': 'uint64',
    # Inclusion slot
    'slot': 'uint64',
    # Sender withdrawal pubkey
    'pubkey': 'bytes48',
    # Sender signature
    'signature': 'bytes96',
}
```

### 信标链区块(Beacon chain blocks)

#### `BeaconBlock`

```
{
    ## Header ##
    'slot': 'uint64',
    'parent_root': 'bytes32',
    'state_root': 'bytes32',
    'randao_reveal': 'bytes96',
    'eth1_data': Eth1Data,
    'signature': 'bytes96',

    ## Body ##
    'body': BeaconBlockBody,
}
```

#### `BeaconBlockBody`

```
{
    'proposer_slashings': [ProposerSlashing],
    'attester_slashings': [AttesterSlashing],
    'attestations': [Attestation],
    'deposits': [Deposit],
    'exits': [Exit],
}
```

#### `ProposalSignedData`

```
{
    # Slot number
    'slot': 'uint64',
    # Shard number (`BEACON_CHAIN_SHARD_NUMBER` for beacon chain)
    'shard': 'uint64',
    # Block's hash of root
    'block_root': 'bytes32',
}
```

### Beacon chain state

#### `BeaconState`

```
{
    # Misc
    'slot': 'uint64',
    'genesis_time': 'uint64',
    'fork': Fork,  # For versioning hard forks

    # Validator registry
    'validator_registry': [Validator],
    'validator_balances': ['uint64'],
    'validator_registry_update_epoch': 'uint64',

    # Randomness and committees
    'latest_randao_mixes': ['bytes32'],
    'previous_epoch_start_shard': 'uint64',
    'current_epoch_start_shard': 'uint64',
    'previous_calculation_epoch': 'uint64',
    'current_calculation_epoch': 'uint64',
    'previous_epoch_seed': 'bytes32',
    'current_epoch_seed': 'bytes32',

    # Finality
    'previous_justified_epoch': 'uint64',
    'justified_epoch': 'uint64',
    'justification_bitfield': 'uint64',
    'finalized_epoch': 'uint64',

    # Recent state
    'latest_crosslinks': [Crosslink],
    'latest_block_roots': ['bytes32'],
    'latest_index_roots': ['bytes32'],
    'latest_penalized_balances': ['uint64'],  # Balances penalized at every withdrawal period
    'latest_attestations': [PendingAttestation],
    'batched_block_roots': ['bytes32'],

    # Ethereum 1.0 chain data
    'latest_eth1_data': Eth1Data,
    'eth1_data_votes': [Eth1DataVote],
}
```

#### `Validator`

```
{
    # BLS public key
    'pubkey': 'bytes48',
    # Withdrawal credentials
    'withdrawal_credentials': 'bytes32',
    # Epoch when validator activated
    'activation_epoch': 'uint64',
    # Epoch when validator exited
    'exit_epoch': 'uint64',
    # Epoch when validator withdrew
    'withdrawal_epoch': 'uint64',
    # Epoch when validator was penalized
    'penalized_epoch': 'uint64',
    # Status flags
    'status_flags': 'uint64',
}
```

#### `Crosslink`

```
{
    # Epoch number
    'epoch': 'uint64',
    # Shard block root
    'shard_block_root': 'bytes32',
}
```

#### `PendingAttestation`

```
{
    # Attester aggregation bitfield
    'aggregation_bitfield': 'bytes',
    # Attestation data
    'data': AttestationData,
    # Custody bitfield
    'custody_bitfield': 'bytes',
    # Inclusion slot
    'inclusion_slot': 'uint64',
}
```

#### `Fork`

```
{
    # Previous fork version
    'previous_version': 'uint64',
    # Current fork version
    'current_version': 'uint64',
    # Fork epoch number
    'epoch': 'uint64',
}
```

#### `Eth1Data`

```
{
    # Root of the deposit tree
    'deposit_root': 'bytes32',
    # Block hash  ETH1.0的块hash
    'block_hash': 'bytes32',
}
```

#### `Eth1DataVote`

```
{
    # Data being voted for
    'eth1_data': Eth1Data,
    # Vote count
    'vote_count': 'uint64',
}
```

## 自定义类型(custom types)

我们定义下列的Python版自定义类型以便类型提示和可读性。

| 名称(Name)       | SSZ equivalent | 描述(Description)                  |
| ---------------- | -------------- | ---------------------------------- |
| `SlotNumber`     | `uint64`       | a slot number                      |
| `EpochNumber`    | `uint64`       | an epoch number                    |
| `ShardNumber`    | `uint64`       | a shard number                     |
| `ValidatorIndex` | `uint64`       | an index in the validator registry |
| `Gwei`           | `uint64`       | an amount in Gwei                  |
| `Bytes32`        | `bytes32`      | 32 bytes of binary data            |
| `BLSPubkey`      | `bytes48`      | a BLS12-381 public key             |
| `BLSSignature`   | `bytes96`      | a BLS12-381 signature              |

辅助函数

### `hash`