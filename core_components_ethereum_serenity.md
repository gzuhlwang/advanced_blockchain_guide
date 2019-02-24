以太坊宁静采用的关键技术：

0 validator shuffling

[实现](https://github.com/protolambda/eth2-shuffle)

1 [BLS聚合签名]([<https://github.com/ethereum/eth2.0-specs/blob/master/specs/bls_signature.md>)[不足：不是量子安全的]

BLS12-381 standardisation([IETF draft](https://tools.ietf.org/id/draft-boneh-bls-signature-00.txt))

ethresear上有关[BLS聚合签名讨论](https://ethresear.ch/t/pragmatic-signature-aggregation-with-bls/2105)

替代方案：zk-stark签名【抗量子】

2 [libp2p](<https://github.com/ethresearch/p2p/issues/7>)

[libp2p example](https://github.com/libp2p/go-libp2p-examples)

3 分叉选择规则[LMD-GHOST](https://vitalik.ca/general/2018/12/05/cbc_casper.html#lmd-ghost)

不同lmd ghost实现[lmd-ghost](https://github.com/protolambda/lmd-ghost)

4 [Randao](https://github.com/randao/randao)

替代方案：verifiable delay function(vdf)

5 Casper ffg

Voting，justification，finalization，dinasty changes，slashing等等

6 序列化方法[SimpleSerialize (SSZ) ](https://github.com/ethereum/eth2.0-specs/blob/master/specs/simple-serialize.md)

7 Tree hashing

8 ETH1.0 充值合约

<https://github.com/ethereum/eth2.0-specs/blob/master/specs/core/0_beacon-chain.md#ethereum-10-deposit-contract>

9 proto buf

10 Beacon chain

https://github.com/ethereum/eth2.0-specs/blob/master/specs/core/0_beacon-chain.md

11 Validator

https://github.com/ethereum/eth2.0-specs/blob/master/specs/validator/0_beacon-chain-validator.md

 