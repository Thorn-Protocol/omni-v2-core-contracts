# OmniFarming V2 Technical Paper

_A solidity forked from [Yearn V3](https://github.com/yearn/yearn-vaults-v3/blob/master/contracts/VaultV3.vy)_

- [Overview](#overview)
- [Architecture](#architecture)
- [Mechanism](#mechanism)

  - [Strategies Mechanism](#strategies-mechanism)
  - [Auto Allocate Mechanism](#auto-allocate-mechanism)
  - [Reward Mechanism](#reward-mechanism)
  - [Fee Mechanism](#fee-mechanism)
  - [Limit Mechanism](#limit-mechanism)
  - [Unrealised Losses Mechanism](#unrealised-losses-mechanism)
  - [Buy Debt Mechanism](#buy-debt-mechanism)

- [Functions](#function)
  - [Deposit](#deposit)
  - [Withdraw](#withdraw)
  - [Report](#report)

# Overview

OmniFarming V2 is a yield optimization protocol that extends the Yearn V3 vault architecture. It allows users to deposit assets into a vault, which are then allocated to various strategies to maximize returns. Key features include automated fund allocation, performance-based fees, and mechanisms to handle unrealized losses

# Architecture

OmniFarming V2 comprises three main components:

- **Vault.sol**: Manages user deposits, withdrawals, and strategy allocations, follow ERC4626..
- **Strategies**: External contracts that invest vault assets to generate profits, follow ERC4626.
- **Accountant.sol**: Calculates and distributes performance fees and refunds.

# Mechanism

### Strategies Mechanism

- Description of strategy functions: add, remove strategy, debt management, update max debt
- Explanation of Queue functionality in this mechanism

* Requires predefined roles (e.g., ROLE_QUEUE_MANAGER, ROLE_ADD_STRATEGY_MANAGER,...) to add, remove strategies, manage debt, and update max debt.

1. Add strategy: Add a strategy to `default_queue` so the vault can allocate assets to it.
2. Remove strategy: Remove a strategy from `default_queue`, preventing it from receiving additional assets.
3. Debt management: includes `ExecuteUpdateMaxDebtForStrategy`: Update max debt for strategy, limiting the amount of assets the strategy can borrow.

- Explanation of Queue functionality in this mechanism:
- The Queue functionality in OmniFarmingV2 is quite interesting, you can think of it as a priority queue of Strategies. When users deposit/mint, the Vault automatically sends assets to the first strategy in `default_queue`, which is very convenient if users want to optimize farming profits. If the first strategy contains maximum assets (reaches maxDebt), it will gradually push to the next strategy (second), and so on.
- This helps optimize profits when farming while being flexible with different strategies.
- Each strategy is an ERC4626 vault, `QUEUE_MANAGER` can arrange, add, or remove strategies in the queue to adjust investment strategies.
  Special case: If a strategy becomes inactive, it is considered out of `default_queue` until it becomes active again.

### Auto Allocate Mechanism

- The Auto Allocate mechanism in OmniFarmingV2 is an automatic asset allocation mechanism when depositing/minting. A very convenient mechanism when the vault can automatically re-balance current_debt to target debt by sending or withdrawing assets from strategy (target_debt must be less than or equal to Strategy's max debt)
- When autoAllocate = true, the vault will automatically allocate assets sent through the deposit function to the first strategy in the queue as long as conditions regarding maxDebt, maxDeposit, minimumTotalIdle are met.
- Where:
- Assets sent to strategy will be taken from vault's "idle" assets (totalIdle). This is the amount of assets that the vault is currently holding (usually tokens like USDC) in the vault contract, not yet allocated to any strategy.
- Assets are withdrawn from strategy. Strategy returns assets to vault based on the amount that can be withdrawn (maxRedeem) and asset status (may be locked or not). (cannot withdraw if locked - special case)
- This mechanism will compare current_debt with target_debt and take funds or deposit a new amount of funds for strategy. At that time, strategy can request a maximum amount of funds it wants to receive to invest and strategy can also refuse freeing funds if those funds are locked (special case).

### Reward Mechanism

The reward mechanism in OmniFarming V2 is quite special. When a strategy is integrated, it can bring profit or loss depending on the time of reporting.

- In case of profit, profit will be gradually vested through a curve based on `profitMaxUnlockTime` in seconds

- In case of loss, the vault will immediately update losses, first by reducing the profit being vested from the previous report, then by reducing pps.

### Fee Mechanism

OmniFarming V2 charges 2 types of fees, including:

- Management Fee: 1% per year on fund
- Performance Fee: 10% on profit

**Management Fee**

Mint corresponding LP amount between 2 deposit/withdraw times based on **user's total liquidity**,
Override functions `PreviewMint` `PreviewWithdraw` `PreviewDeposit` `PreviewRedeem` with new formula that calculates management Fee into totalSupply

**Performance Fee**

Collect fees through `Accountant` in `ExecuteProcessReport` function

When a strategy is reported, `Accountant` will calculate `performanceFee` and `refund`

- `PerformanceFee`: Fee collected based on profit (business requirement 10%), this fee will be converted to liquidity amount and mint them as liquidity for `Accountant`

- `refund`: (Forked yearn v3) refund value each time a strategy is reported (can be used to make reward boost apy or compensate for losses)

### Limit Mechanism

The vault has mechanisms to limit user balances by setting limits at `deposit` and `withdraw` functions through 2 main mechanisms: `depositLimit` of Vault or more advanced `depositLimitModule` and `withdrawLimitModule`

- `DepositLimit`: Limit vault's TVL, updated through `setLimitDeposit` function
- `depositLimitModule`: Advanced checking contract with additional user address input, can be used to limit individual user assets
- `withdrawLimitModule`: Similar to depositLimitModule, with additional advanced checking conditions when withdrawing

### Unrealised Losses Mechanism

UnrealisedLosses is a mechanism to calculate temporary losses (unreported losses), which is an important part to protect users who withdraw later from having to "bear losses" for users who withdraw earlier.

### Buy Debt Mechanism

Buy Debt is a mechanism that allows authorized people to buy back Vault debt

- Only people with ROLE `ROLE_DEBT_PURCHASER` can buy debt
- ROLE is granted by GOVERNANCE

### Minimum Total Idle

Minimum total Idle is a function to keep a minimum amount of money in the vault for liquidity, they have functions

- Fast withdrawal liquidity
- Avoid force-withdraw from strategy causing losses
- Set and updated by `ROLE_MINIMUM_IDLE_MANAGER`

# Function

### Deposit

#### Functional Requirements

- Users send assets and receive corresponding LP
- If automatic allocation mechanism to first strategy is enabled, assets will be automatically allocated to the first strategy

### Withdraw

### Functional Requirements

- Users withdraw assets by burning corresponding LP amount
- Vault prioritizes idle assets in vault, if insufficient will withdraw the shortfall from strategies
- When withdrawing early from strategy there may be losses (due to strategy may lose), so users can enter a `loss` value that works similar to slippage, can accept loss amount within allowable limits
- Allow B to withdraw money from A's LP if A has approved the LP amount

### Report

Functional Requirements

- Calculate profit/loss between 2 reports
- Call `Accountant` to calculate performanceFees or refund
- Recalculate totalSupply based on profit/loss option
  - if profit: mint additional LP, lock to vault address and burn gradually through `unlockShares()` function
  - if loss: first compensate by reducing reward from previous report, if still loss after that then reduce PPS (update totalDebt)

```solidity
uint256 totalSupply = vault.totalSupply() + lockedShares;
uint256 endingSupply = totalSupply - lockedShares + sharesToLock - sharesToBurn;

if (endingSupply > totalSupply) {
      vault._mint(address(this), endingSupply - totalSupply);
  }
if (totalSupply > endingSupply) {
    uint256 toBurn = Math.min(totalSupply - endingSupply,totalLockedShares);
    vault._burn(address(this), toBurn);
}
```
