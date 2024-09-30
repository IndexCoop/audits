# Index Coop icUSD Audit Report

### Reviewed by: 0x52 ([@IAm0x52](https://twitter.com/IAm0x52))

### Review Date(s): 9/13/24 - 9/17/24

### Fix Review Date(s): 9/28/24

### Fix Review Hash: [713aee7](https://github.com/IndexCoop/index-protocol/tree/713aee761db8e9638df0786e63ec2a6a23be393e)

# <br/> 0x52 Background

As an independent smart contract auditor I have completed over 100 separate reviews. I primarily compete in public contests as well as conducting private reviews (like this one here). I have more than 30 1st place finishes (and counting) in public contests on [Code4rena](https://code4rena.com/@0x52) and [Sherlock](https://audits.sherlock.xyz/watson/0x52). I have also partnered with [SpearbitDAO](https://cantina.xyz/u/iam0x52) as a Lead Security researcher. My work has helped to secure over $1 billion in TVL across 100+ protocols.

# <br/> Scope

The [index-coop-smart-contracts](https://github.com/IndexCoop/index-coop-smart-contracts/) repo and [Index-protocol](https://github.com/IndexCoop/index-protocol/) repo were reviewed at commit hash [1ebd3db](https://github.com/IndexCoop/index-coop-smart-contracts/tree/1ebd3db2ddc5fc1a6959cd7b0472b775d06348fd) and commit hash [1b638bf](https://github.com/IndexCoop/index-protocol/tree/1b638bfc9eefdb20cbf5e407b9eafc7985ea65e6), respectively.

In-Scope Contracts:

    index-coop-smart-contracts
        -contract/adapters/TargetWeightWrapExtension.sol
    
    Index-protocol
        -contracts/protocol/v1/CustomOracleNAVIssuanceModule.sol
        -contracts/protocol/SetValuer.sol
        -contracts/protocol/PriceOracle.sol
        -contracts/protocol/integration/oracles/PreciseUnitOracle.sol
        -contracts/protocol/integration/oracles/ERC4626Oracle.sol
        -contracts/protocol/modules/v1/RebasingComponentModule.sol
        -contracts/protocol/modules/v1/WrapModuleV2.sol
        -contracts/protocol/integration/warp-v2/AaveV2WrapV2Adapter.sol
        -contracts/protocol/integration/warp-v2/AaveV3WrapV2Adapter.sol
        -contracts/protocol/integration/warp-v2/CompoundV3WrapV2Adapter.sol
        -contracts/protocol/integration/warp-v2/ERC4626WrapV2Adapter.sol

Deployment Chain(s)
- Ethereum Mainnet

# <br/> Summary of Findings

|  Identifier  | Title                        | Severity      | Mitigated |
| ------ | ---------------------------- | ------------- | ----- |
| [H-01] | [Alternating DIMv3 deposits and NAV redemptions can be used to inflate position multiplier and steal funds via rounding loss](#h-01-alternating-dimv3-deposits-and-nav-redemptions-can-be-used-to-inflate-position-multiplier-and-steal-funds-via-rounding-loss) | High | ✔️ |
| [M-01] | [ERC4626Oracle#read may return incorrect value for future added ERC4626 vaults](#m-01-erc4626oracleread-may-return-incorrect-value-for-future-added-erc4626-vaults) | Medium | ✔️ |
| [M-02] | [WrapModuleV2 fails to consider slippage when depositing into ERC4626 vaults](#m-02-wrapmodulev2-fails-to-consider-slippage-when-depositing-into-erc4626-vaults) | Medium | ✔️ |

# <br/> Detailed Findings

## [H-01] Alternating DIMv3 deposits and NAV redemptions can be used to inflate position multiplier and steal funds via rounding loss

### Details 

[CustomOracleNAVIssuanceModule.sol#L1024-L1039](https://github.com/IndexCoop/index-protocol/blob/1b638bfc9eefdb20cbf5e407b9eafc7985ea65e6/contracts/protocol/modules/v1/CustomOracleNAVIssuanceModule.sol#L1024-L1039)

    function _getRedeemPositionMultiplier(
        ISetToken _setToken,
        uint256 _setTokenQuantity,
        ActionInfo memory _redeemInfo
    )
        internal
        view
        returns (uint256, int256)
    {
        uint256 newTotalSupply = _redeemInfo.previousSetTokenSupply.sub(_setTokenQuantity);
        int256 newPositionMultiplier = _setToken.positionMultiplier()
            .mul(_redeemInfo.previousSetTokenSupply.toInt256())
            .div(newTotalSupply.toInt256());

        return (newTotalSupply, newPositionMultiplier);
    }

When redeeming from the NAVIM the position multiplier for the set token is increased. This scales the internal balances of the other components to account for the USDC being removed to pay for the redemption.

If mint and redeems are approximately even this is never an issue but I can be if there is large imbalance. This leads to the following attack which can inflate the position multiplier:

1. Mint via DIMv3
2. Unwrap yb-tokens to USDC (permissionlessly)
3. Redeem via NAVIM
4. Repeat

With each iteration the position multiplier is pushed higher. The opposite order can be done to deflate the position multiplier.

[SetToken.sol#L560-L574](https://github.com/IndexCoop/index-protocol/blob/1b638bfc9eefdb20cbf5e407b9eafc7985ea65e6/contracts/protocol/SetToken.sol#L560-L574)

    function _convertRealToVirtualUnit(int256 _realUnit) internal view returns(int256) {
        int256 virtualUnit = _realUnit.conservativePreciseDiv(positionMultiplier);

        // This check ensures that the virtual unit does not return a result that has rounded down to 0
        if (_realUnit > 0 && virtualUnit == 0) {
            revert("Real to Virtual unit conversion invalid");
        }

        // This check ensures that when converting back to realUnits the unit won't be rounded down to 0
        if (_realUnit > 0 && _convertVirtualToRealUnit(virtualUnit) == 0) {
            revert("Virtual to Real unit conversion invalid");
        }

        return virtualUnit;
    }

When converting from real to virtual unit during updates the contract uses the following math:

    realUnit * 1e18 / positionMultiplier

When converting back:

    virtualUnit * positionMultiplier / 1e18

For very high values of positionMultiplier (i.e. 1e35) we start getting significant precision loss. Lets walk through an example with a real unit of 0.25e18:

    virtualUnit = 0.25e18 * 1e18 / 1e35 = 2

    realUnit = 2 * 1e35 / 1e18 = 0.2e18

We can see our 0.25e18 real unit ends up truncated as 0.2e18.

With these two concepts we can now put together an attack to drain the entire vault:

1. Inflate positionMultiplier to cause precision loss
2. Deposit on DIMv3. This only requires 0.2 to mint but grants ~0.25 in assets
3. Deflate positionMultiplier to reverse precision loss
4. Withdraw more assets than deposited
5. Repeat

This can be repeated as many times as necessary to drain the vault of all funds. Due to the multiplicative nature of the positionMultiplier it take only ~50 inflation loops of 20% to increase the position multiplier to 1e22 which is what is necessary for to cause inflation loss to a 6 dp token. At with withdraw fee of 0.2% the net cost of the attack is:

    0.2 * 50 * 20% = 2%

Even with fees the total cost of the attack is ~2% NAV, making it highly profitable to execute assuming reasonable gas costs.

### Lines of Code

[SetToken.sol#L560-L574](https://github.com/IndexCoop/index-protocol/blob/1b638bfc9eefdb20cbf5e407b9eafc7985ea65e6/contracts/protocol/SetToken.sol#L560-L574)

### Recommendation

Strict positionMultiplier bounds (i.e. max 1e20) should be placed to prevent large amounts of precision loss in the underlying units.

### Remediation

Fixed in commit [7052f2c3](https://github.com/IndexCoop/index-protocol/pull/49/commits/7052f2c39aa6702eb4335e752c3838497b1a43e7). When redeeming from the NAV issuance module, the position multiplier is limited to 1e20, preventing large amount of precision loss that can be abused.

## <br/> [M-01] ERC4626Oracle#read may return incorrect value for future added ERC4626 vaults

### Details 

[ERC4626Oracle.sol#L55-L58](https://github.com/IndexCoop/index-protocol/blob/1b638bfc9eefdb20cbf5e407b9eafc7985ea65e6/contracts/protocol/integration/oracles/ERC4626Oracle.sol#L55-L58)

    function read() external view returns (uint256) {
        uint256 assetsPerShare = vault.convertToAssets(vaultFullUnit);
        return assetsPerShare * vaultFullUnit / underlyingFullUnit;
    }

PriceOracle.sol assumes that all prices returned by integrated sub-oracles will return the price to 18 dp. When returned price, ERC4626Oracle.sol assumes that vaultFullUnit == 1e18. The standard practice for ERC4626 vaults is for the share precision to match that of the underlying token. 

Although Gauntlet Morpho USDC is an exception with an 18 dp share, this contract is a generic ERC4626 adapter and would present a large security risk if other USDC vaults such as Yearn's [yvUSDC](https://yearn.fi/vaults/1/0xa354F35829Ae975e850e23e9615b11Da1B3dC4DE?tab=info) are added to the vault at a later date. In this scenario the shares would be highly undervalued, allowing users to deposit via the NAV issuance module and withdraw from the debt issuance module to drain the vault.

### Lines of Code

[ERC4626Oracle.sol#L55-L58](https://github.com/IndexCoop/index-protocol/blob/1b638bfc9eefdb20cbf5e407b9eafc7985ea65e6/contracts/protocol/integration/oracles/ERC4626Oracle.sol#L55-L58)

### Recommendation

vaultFullUnit should not be assumed to be 1e18. Instead a separate conversion factor should be used to ensure the price is returned to 18 dp.

### Remediation

Fixed in commit [cf32c30](https://github.com/IndexCoop/index-protocol/pull/49/commits/6f32c300d88501aa3163e7a8eaf34fad20a3c643) as recommended.

## <br/> [M-02] WrapModuleV2 fails to consider slippage when depositing into ERC4626 vaults

### Details 

[WrapModuleV2.sol#L450-L471](https://github.com/IndexCoop/index-protocol/blob/1b638bfc9eefdb20cbf5e407b9eafc7985ea65e6/contracts/protocol/modules/v1/WrapModuleV2.sol#L450-L471)

    function _createWrapDataAndInvoke(
        ISetToken _setToken,
        IWrapV2Adapter _wrapAdapter,
        address _underlyingToken,
        address _wrappedToken,
        uint256 _notionalUnderlying,
        bytes memory _wrapData
    ) internal {
        (
            address callTarget,
            uint256 callValue,
            bytes memory callByteData
        ) = _wrapAdapter.getWrapCallData(
            _underlyingToken,
            _wrappedToken,
            _notionalUnderlying,
            address(_setToken),
            _wrapData
        );

        _setToken.invoke(callTarget, callValue, callByteData);
    }

When depositing into a majority of ERC4626 vaults, it is possible to encounter slippage. When making deposit calls WrapModuleV2 does so directly without considering this.

Similar to M-01, Gauntlet Morpho USDC is an exception and operates in a no-loss manner which is atypical behavior for ERC4626 vaults. This would present a large security risk if other USDC vaults such as Yearn's [yvUSDC](https://yearn.fi/vaults/1/0xa354F35829Ae975e850e23e9615b11Da1B3dC4DE?tab=info) are added to the vault at a later date. In this scenario wrap/unwrap calls to the ERC4626 vaults would be MEV'd and large portions of funds would be lost.

### Lines of Code

[WrapModuleV2.sol#L450-L471](https://github.com/IndexCoop/index-protocol/blob/1b638bfc9eefdb20cbf5e407b9eafc7985ea65e6/contracts/protocol/modules/v1/WrapModuleV2.sol#L450-L471)

### Recommendation

Typically a [router](https://github.com/yearn/Yearn-ERC4626-Router/blob/master/src/Yearn4626Router.sol) is used to ensure slippage is accounted for. Integrating these slippage protections would require large changes to the WrapModuleV2 which is not meant to interact with "wrappers" that can lose value on deposit or withdrawal. This leads me to my two recommendations:

1. Future vaults should either be screened to ensure only no-loss operation vaults
2. Or these modules should be entered/exited from a newly created/modified contract that manages slippage

### Remediation

Fixed in commit [98c2c9c](https://github.com/IndexCoop/index-protocol/pull/49/commits/98c2c9c831cc1b01d737fe81cec0b9f26bd61cbf) as recommended. Warning added to ERC4626WrapV2Adapter to only use no-loss vaults with wrapper. ERC4626ExchangeAdapter has been added to allow for vaults that may incur value loss due to fees, slippage or the like
