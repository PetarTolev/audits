# Sparkn - Findings Report

# Table of contents

- ## Low Risk Findings
  - ### [L-01. Precision Loss in Token Distribution due to Integer Division](#L-01)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: CodeFox Inc.

### Dates: Aug 21st, 2023 - Aug 29th, 2023

[See more contest details here](https://www.codehawks.com/contests/cllcnja1h0001lc08z7w0orxx)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 0
- Medium: 0
- Low: 1

# Low Risk Findings

## <a id='L-01'></a>L-01. Precision Loss in Token Distribution due to Integer Division

### Relevant GitHub Links

https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/Distributor.sol#L146

## Summary

The lack of fractional number support can lead to precision loss or rounding to zero in token distribution. This occurs when the token amount each winner should receive, based on their percentage share, is calculated. If the total tokens are small and a winner's share is less than 1%, the token amount could round down to zero. Similarly, if the total tokens are large and the winner's share results in a fractional token amount, the token amount could be rounded down, leading to a loss.

## Vulnerability Details

The `_distribute` function in the `Distributor.sol` may potentially lead to precision loss or rounding to zero. This is because the calculation `totalAmount * percentages[i] / BASIS_POINTS` might not always result in a whole number. Solidity does not support floating point numbers, so any fractional part is truncated, which can lead to precision loss.

Moreover, if `totalAmount * percentages[i]` is less than `BASIS_POINTS (10_000)`, the result will be zero due to the way integer division works in Solidity. This could potentially lead to a situation where a winner is supposed to receive a small amount of tokens but receives none due to rounding to zero.

Let's consider a few examples:

1. Rounding down to zero:
   Suppose `totalAmount` is 1000 tokens, `BASIS_POINTS` is 10000 (representing 100%), and a winner's percentage `(percentages[i])` is 0.01% (which would be 1 in terms of basis points). The calculation would be: `1000 * 1 / 10000 = 0.1`.
   Since Solidity doesn't support fractional numbers, this would be rounded down to 0. So, even though the winner should receive 0.1 tokens, they would receive none due to rounding down to zero.

2. Precision loss:
   Suppose `totalAmount` is 1000 tokens, `BASIS_POINTS` is 10000, and a winner's percentage is 33.33% (which would be 3333 in terms of basis points). The calculation would be: `1000 * 3333 / 10000 = 333.3`.
   Again, since Solidity doesn't support fractional numbers, this would be rounded down to 333. So, the winner would receive 333 tokens instead of 333.3, leading to a precision loss of 0.3 tokens.

## Proof of Concept

1. Add the following test to the `FuzzTestProxyFactory.t.sol` file.
2. Run the test using the command: `forge test --match-test testDistributePrecisionLoss -vvv`.

```
    function testDistributePrecisionLoss() public {
        uint256 _CONTEST_CLOSE_TIME = block.timestamp + 1 days;
        bytes32 _CONTEST_ID = keccak256(abi.encode("Jason", "001"));
        uint256 _SPONSOR_AMOUNT = 100;

        vm.prank(factoryAdmin);
        proxyFactory.setContest(organizer, _CONTEST_ID, _CONTEST_CLOSE_TIME, address(distributor));

        bytes32 salt = keccak256(abi.encode(organizer, _CONTEST_ID, address(distributor)));
        address proxyAddress = proxyFactory.getProxyAddress(salt, address(distributor));

        vm.prank(sponsor);
        MockERC20(usdcAddress).transfer(proxyAddress, _SPONSOR_AMOUNT);
        assertEq(MockERC20(usdcAddress).balanceOf(proxyAddress), _SPONSOR_AMOUNT);

        vm.warp(_CONTEST_CLOSE_TIME + 1);

        address[] memory winners = new address[](2);
        winners[0] = user1;
        winners[1] = user2;

        uint256[] memory percentages = new uint256[](2);
        percentages[0] = 9401;
        percentages[1] = 99;

        bytes memory data = abi.encodeWithSelector(
          Distributor.distribute.selector,
          usdcAddress,
          winners,
          percentages,
          ""
        );

        vm.prank(organizer);
        proxyFactory.deployProxyAndDistribute(_CONTEST_ID, address(distributor), data);

        assert(MockERC20(usdcAddress).balanceOf(user1) == 94);
        assert(MockERC20(usdcAddress).balanceOf(user2) == 0);
        assert(MockERC20(usdcAddress).balanceOf(makeAddr("stadium")) == 6);
      }
```

## Impact

##### Impact: High, because winners will not receive their rewards.

##### Likelihood: Low, as it requires a very low percentage and a small reward amount.

The impact of this issue can be significant, especially in a scenario where a large number of tokens are being distributed among a large number of winners.

1. Loss of Funds: Winners could potentially lose funds due to the rounding down of their token amounts. In extreme cases, winners could end up receiving no tokens at all if their calculated token amount rounds down to zero.

2. Inequitable Distribution: The distribution of tokens may not accurately reflect the intended percentages due to rounding errors. This could lead to some winners receiving less than their fair share of tokens.

## Tools Used

Manual Review

## Recommendations

To mitigate this, it's important to ensure that the `totalAmount` of tokens and the percentages are set in a way that prevents such precision loss or rounding to zero.
