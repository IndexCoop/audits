# Index PRT Audit Report

### Reviewed by: 0x52 ([@IAm0x52](https://twitter.com/IAm0x52))

### Review Date(s): 7/4/24 - 7/7/24

### Fix Review Date(s): 7/8/24

### Fix Review Hash: [e4769f7](https://github.com/IndexCoop/periphery/commit/e4769f70d31d4422e2a8d70f60335e56b38703fd)

# <br/> 0x52 Background

As an independent smart contract auditor I have completed over 100 separate reviews. I primarily compete in public contests as well as conducting private reviews (like this one here). I have more than 30 1st place finishes (and counting) in public contests on [Code4rena](https://code4rena.com/@0x52) and [Sherlock](https://audits.sherlock.xyz/watson/0x52). I have also partnered with [SpearbitDAO](https://cantina.xyz/u/iam0x52) as a Lead Security researcher. My work has helped to secure over $1 billion in TVL across 100+ protocols.

# <br/> Scope

The [index-coop-smart-contracts](https://github.com/IndexCoop/index-coop-smart-contracts/) repo and [periphery](https://github.com/IndexCoop/periphery/) repo were reviewed at commit hash [6477886](https://github.com/IndexCoop/index-coop-smart-contracts/tree/6477886650d82339d06594c29b40292ab77dcd5e) and commit hash [93ee6d1](https://github.com/IndexCoop/periphery/tree/93ee6d1597471aad3e65d7d30ff538f4bb2b0d8b ), respectively.

In-Scope Contracts
	periphery
		- src/staking/SnapshotStakingPool.sol
		- src/staking/SignedSnapshotStakingPool.sol
	index-coop-smart-contracts
		- contracts/adapters/PrtFeeSplitExtension.sol
		- contracts/token/Prt.sol

Deployment Chain(s)
- Ethereum Mainnet

# <br/> Summary of Findings

|  Identifier  | Title                        | Severity      | Mitigated |
| ------ | ---------------------------- | ------------- | ----- |
| [M-01] | [Allowing anyone to accrue opens significant opportunity to MEV snapshots with a flashloan](#m-01-allowing-anyone-to-accrue-opens-significant-opportunity-to-mev-snapshots-with-a-flashloan) | Med | ✔️ |
| [M-02] | [`lastSnapshotTime` isn't initialized leading to race conditions to get entire first accrue](#m-02-lastsnapshottime-isnt-initialized-leading-to-race-conditions-to-get-entire-first-accrue) | Med | ✔️ |

# <br/> Detailed Findings

## [M-01] Allowing anyone to accrue opens significant opportunity to MEV snapshots with a flashloan

### Details 

[SnapshotStakingPool.sol#L186-L190](https://github.com/IndexCoop/periphery/blob/93ee6d1597471aad3e65d7d30ff538f4bb2b0d8b/src/staking/SnapshotStakingPool.sol#L186-L190)

    function rewardOfAt(address account, uint256 snapshotId) public view virtual returns (uint256) {
        if (snapshotId == 0) revert InvalidSnapshotId();
        if (snapshotId > _getCurrentSnapshotId()) revert NonExistentSnapshotId();
        return _rewardOfAt(account, snapshotId);
    }

[SnapshotStakingPool.sol#L243-L245](https://github.com/IndexCoop/periphery/blob/93ee6d1597471aad3e65d7d30ff538f4bb2b0d8b/src/staking/SnapshotStakingPool.sol#L243-L245)

    function _rewardOfAt(address account, uint256 snapshotId) internal view returns (uint256) {
        return _rewardAt(snapshotId) * balanceOfAt(account, snapshotId) / totalSupplyAt(snapshotId);
    }

When checking the rewards for a snapshot, the snapshot balance of the user is used to determine their percentage ownership of the pool and rewards are distributed proportionally. As a function of how snapshots work, a users snapshot balance is still recorded even if that user withdraws in the same block of the snapshot as long as the withdraw takes place after the snapshot in the transaction flow.

Coupled with the ability for any user to accrue rewards leads to a significant MEV opportunities. A user can deposit, accrue, then withdraw in a single block. This makes flashloans a viable option to exploit this. Consider that AMMs are the backbone of nearly all token liquidity and would contain a sizable portion of the PRT. Additionally almost every major AMM besides curve allows utilizing it's entire balance for flashloans. A user can borrow all PRT from AMMs (i.e. Uniswap, Balancer, etc) via flashloans, deposit all the tokens, call accrue, then unstake and repay all flashloans. Doing so allows the user to MEV a significant portion of the distribution.

To test add the following to SnapshotStakingPool.t.sol

    function testSnapshotFlashLoanMEV() public {
        _stake(bob.addr, 1 ether);
        _snapshotMev(alice.addr, 1 ether);

        assertEq(snapshotStakingPool.rewardOfAt(bob.addr, 1), 0.5 ether);
        assertEq(snapshotStakingPool.rewardOfAt(alice.addr, 1), 0.5 ether);
    }

    function _snapshotMev(address staker1, uint256 amount) internal {
        vm.warp(block.timestamp + snapshotDelay + 1);
        _stake(staker1, amount);
        vm.prank(owner);
        rewardToken.transfer(distributor, amount);
        vm.prank(distributor);
        rewardToken.approve(address(snapshotStakingPool), amount);
        vm.prank(distributor);
        snapshotStakingPool.accrue(amount);
        _unstake(staker1, amount);
    }

### Lines of Code

[SnapshotStakingPool.sol#L243-L245](https://github.com/IndexCoop/periphery/blob/93ee6d1597471aad3e65d7d30ff538f4bb2b0d8b/src/staking/SnapshotStakingPool.sol#L243-L245)

### Recommendation

Generally it is recommended to implement at least some timelock for staking that offers payouts. One example is the Sushibar implemented by Sushiswap. It has no timelock on staking and large amounts of staking returns have been lost to MEV by sandwich staking.

### Remediation

Fixed in commit [45cac95](https://github.com/IndexCoop/periphery/pull/4/commits/45cac955ea22967d4c6949a4c5c65aa5eb3ddead). Adds a cutoff time before distribution in which staking is disabled. This enforces a minimum staking length and prevents MEV during the distribution.

## [M-02] `lastSnapshotTime` isn't initialized leading to race conditions to get entire first accrue

### Details 

[SnapshotStakingPool.sol#L210-L212](https://github.com/IndexCoop/periphery/blob/93ee6d1597471aad3e65d7d30ff538f4bb2b0d8b/src/staking/SnapshotStakingPool.sol#L210-L212)

    function canAccrue() public view returns (bool) {
        return block.timestamp >= lastSnapshotTime + snapshotDelay;
    }

Streaming fees can be accrued once `block.timestamp >= lastSnapshotTime + snapshotDelay`. Since `lastSnapshotTime` is not initialized, the contract will be eligible to accrue immediately after it is created. This leads to race conditions since every participant would be highly incentivized to claim their PRT, stake the accrue before any other users. This way they can get all fees that have accumulated during the post launch period prior to the PRT launch.

### Lines of Code

[SnapshotStakingPool.sol#L72-L86](https://github.com/IndexCoop/periphery/blob/93ee6d1597471aad3e65d7d30ff538f4bb2b0d8b/src/staking/SnapshotStakingPool.sol#L72-L86)

### Recommendation

`lastSnapshotTime` should be initialized to `block.timestamp` in the constructor

### Remediation

Fixed as recommended in commit [45cac95](https://github.com/IndexCoop/periphery/pull/4/commits/45cac955ea22967d4c6949a4c5c65aa5eb3ddead).