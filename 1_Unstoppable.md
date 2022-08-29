# Challenge #1 - Unstoppable

## Description

There's a lending pool with a million DVT tokens in balance, offering flash loans for free.

If only there was a way to attack and stop the pool from offering flash loans ...

You start with 100 DVT tokens in balance.

---

## What we have

We have 2 contracts here:

1. `UnstoppableLender`: A basic flash-loan provider which is giving DVT tokens.

2. `ReceiverUnstoppable`: Flash loan receiver.

Looking at the contracts, we find these issues:

## Issues in UnstoppableLender

### 1. Uninitialized variable `poolBalance`.

### 2. Should use `SafeERC20`'s `safeTransferFrom` instead of `transferFrom`

ERC20s vary in implementations regarding the `bool` returning functions (`transfer`, `transferFrom` and `approve`). Some may return `false` instead of reverting in case of any issue.

### 3. `assert(poolBalance == balanceBefore)` can be used to break the pool permanently

`poolBalance` is maintained using the `deposit` function, but someone can send tokens directly without that. In that case `poolBalance` will not be updated, and the assertion will fail.

### 4. Pool's current logic will not work with reflection tokens

## Issues in ReceiverUnstoppable

### 1. Use of `transfer` to send ether

`transfer` depends highly on the gas costs, as it passes a fix stipend of 2300 gas and thus carries a risk of failure with change in gas costs. Instead, `call` should be used with reeentrancy checks.

### 2. No reentrancy checks

There is a risk of `pool` doing a reentrancy attack on the receiver. Although if `pool` is trusted, this check might be omitted.

## Solution

Our goal was to stop the flashloan contract. And the 3rd issue of `UnstoppableLender` provides the way to do so.

```ts
// Don't want to waste a lot of my precious DVTs 🙃
await this.token.connect(attacker).transfer(this.pool.address, "1");
```

## Probable Fix

Just removing the assertion will fix this. The assertion serves no purpose here. Even if the pool gets some tokens from somewhere, it is beneficial for the pool anyway.
