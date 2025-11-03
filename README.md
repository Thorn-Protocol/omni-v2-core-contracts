# OmniFarming Vault

OmniFarming is designed with two main parts: Vault and Strategies. They are designed so that users can deposit money into the vault, then the vault manager will allocate assets to the strategies. The special feature is that the vault will distribute profits in a curve, helping to stabilize profits. Strategy sockets are designed according to the ERC4626 standard to help directly integrate available strategies and shorten development time.

# Contracts

### Vault

This contract is a rewritten version of YearnV3, edited from Vyper language to solidity, in addition they have been edited to fit the fee calculation formula.

- The [Yearn V3 repo](https://github.com/yearn/yearn-vaults-v3), design source description.
- The [Technical paper](https://github.com/Thorn-Protocol/omni-v2-core-contracts/tree/main/docs), describes some mechanisms inherited from yearnV3 and some changed mechanisms such as calculating fees.

### Strategy

Strategies designed according to ERC4626 standard can be integrated directly into Vault, strategies are compiled in the repo: https://github.com/Thorn-Protocol/omni-V2-strategy-contracts

# Deployment

| Contract              | Strategy          | Address                                    |
| --------------------- | ----------------- | ------------------------------------------ |
| USDC V2 On Base Vault |                   | 0x2669DfA1D91c1dF9fe51DEAC6E5369C7D43242a8 |
|                       | Wasabi Strategy   | 0x1C4a802FD6B591BB71dAA01D8335e43719048B24 |
|                       | OffChain Strategy | 0x00aa576bfa5f75BC6C651e8Cb587dD78b287040A |
