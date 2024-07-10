# Index DIMv3 Audit Report

### Reviewed by: 0x52 ([@IAm0x52](https://twitter.com/IAm0x52))

### Review Date(s): 5/8/24 - 5/10/24

### Fix Review Date(s): m/d/yy

# 0x52 Background

As an independent smart contract auditor I have completed over 100 separate reviews. I primarily compete in public contests as well as conducting private reviews (like this one here). I have more than 30 1st place finishes (and counting) in public contests on [Code4rena](https://code4rena.com/@0x52) and [Sherlock](https://audits.sherlock.xyz/watson/0x52). I have also partnered with [SpearbitDAO](https://cantina.xyz/u/iam0x52) as a Lead Security researcher. My work has helped to secure over $1 billion in TVL across 100+ protocols.

# Scope

The [index-protocol](https://github.com/IndexCoop/index-protocol/) repo was reviewed at commit hash [a43caa2](https://github.com/IndexCoop/index-protocol/tree/a43caa279eacd77b7db7e2014cdf4eb2a1dde4a6/)

In-Scope Contracts
- contracts/protocol/modules/v1/DebtIssuanceModuleV3.sol

Deployment Chain(s)
- Ethereum Mainnet
- Arbitrum Mainnet

# Summary of Findings

|  Identifier  | Title                        | Severity      | Mitigated |
| ------ | ---------------------------- | ------------- | ----- |
| [L-01] | [Using a tokenTransferBuffer of 1 will allow DOS to occur again sometime in the future](#l-01-using-a-tokentransferbuffer-of-1-will-allow-dos-to-occur-again-sometime-in-the-future) | Low | ✔️ |

# Detailed Findings

## [L-01] Using a tokenTransferBuffer of 1 will allow DOS to occur again sometime in the future

### Details 

The motivation for the DebtIssuanceModule upgrade was to address a balance anomaly present in AAVE v3 in which the final balance did not match the prior balance incremented by the transfer amount:

    componentToken.balanceOf(setToken)[after transfer] != componentToken.balanceOf(setToken)[before transfer] + transferredAmount

After a deep inspection and extensive fuzzing of the AAVE v3 token contracts, it was concluded that the following line was causing this unusual behavior:

[ScaledBalanceTokenBase.sol#L142](https://github.com/aave/aave-v3-core/blob/724a9ef43adf139437ba87dcbab63462394d4601/contracts/protocol/tokenization/base/ScaledBalanceTokenBase.sol#L142)

    super._transfer(sender, recipient, amount.rayDiv(index).toUint128());

The amount.rayDiv creates single wei rounding errors in certain circumstances which are dependent on the current liquidity index. 

A short explanation on how the liquidity index functions inside the AAVE v3 protocol. It is the conversion ratio (stored as a ray) from aToken to underlying. When it is 2e27, 1 wei of aToken == 2 wei of underlying token. It increases as interest is accrued. As an example, after 1 year of 5% interest, the liquidity index will increase from 1e27 to 1.05e27.

The above rounding leads to single wei rounding errors in UNSCALED balances. The results is that the balance can vary by as much as the liquidity index / 2e27 rounded up. Variance scales with 2e27 rather than 1e27 since the [math library](https://github.com/aave/aave-v3-core/blob/724a9ef43adf139437ba87dcbab63462394d4601/contracts/protocol/libraries/math/WadRayMath.sol#L65-L74) used is rounded up or down around 0.5 which effectively increase the dp by 0.5. Through fuzzing it was confirmed that the variance is 1 up to a liquidity index of 2e27, 2 up to 4e27, 3 up to 6e27.

At 5% APR it would take a token ~14 years to reach a liquidity index of 2e27 and ~28 years to reach 4e27.

### Lines of Code

[DebtIssuanceModuleV3.sol#L45](https://github.com/IndexCoop/index-protocol/blob/a43caa279eacd77b7db7e2014cdf4eb2a1dde4a6/contracts/protocol/modules/v1/DebtIssuanceModuleV3.sol#L45)

### Recommendation

Depending on the expected lifespan of the product, it may be desireable to use a dynamic buffer that can be increased as the liquidity index increases. If gas efficiency is a higher priority, setting the immutable buffer to 10 would cover 5% APY for up to ~61 years.

### Remediation

