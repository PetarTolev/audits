# DittoETH - Findings Report

# Table of contents

- ## Medium Risk Findings
  - ### [M-01. Insufficient staleness validation in LibOracle's oracleCircuitBreaker function](#M-01)
  - ### [M-02. Unwithdrawable ETH in BridgeReth Contract](#M-02)
  - ### [M-03. Market Order ID Overflow and Potential Disruption to Other Market IDs](#M-03)
- ## Low Risk Findings
  - ### [L-01. Insufficient Validation of Heartbeat for Oracle Price Feeds](#L-01)
  - ### [L-02. Decimal price discrepancies in Chainlink oracles pose a potential risk for users](#L-02)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: Ditto

### Dates: Sep 8th, 2023 - Oct 9th, 2023

[See more contest details here](https://www.codehawks.com/contests/clm871gl00001mp081mzjdlwc)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 0
- Medium: 5
- Low: 2

# Medium Risk Findings

## <a id='M-01'></a>M-01. Insufficient staleness validation in LibOracle's oracleCircuitBreaker function

### Relevant GitHub Links

https://github.com/Cyfrin/2023-09-ditto/blob/main/contracts/libraries/LibOracle.sol#L112-L126

## Summary

The `LibOracle.oracleCircuitBreaker` function lacks validation for the heartbeat of both oracles.

## Vulnerability Detail

The `LibOracle.oracleCircuitBreaker` function is responsible for validating both the `getLatestData` of the base oracle and the asset oracle. However, it currently lacks sufficient validation.

## Impact

Loss of funds due to inadequate staleness checks for the asset's price.

## Tool Used

Manual Review

## Recommendation

Add heartbeat validation similar to that in the `baseOracleCircuitBreaker` function. This modification will help ensure proper validation of both oracles' heartbeats and enhance the security of the system.

```diff
bool invalidFetchData = roundId == 0 || timeStamp == 0
+ || block.timestamp > 2 hours + timeStamp
+ || block.timestamp > 2 hours + baseTimeStamp
  || timeStamp > block.timestamp || chainlinkPrice <= 0 || baseRoundId == 0
  || baseTimeStamp == 0 || baseTimeStamp > block.timestamp
  || baseChainlinkPrice <= 0;
```

## <a id='M-02'></a>M-02. Unwithdrawable ETH in BridgeReth Contract

### Relevant GitHub Links

https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/bridges/BridgeReth.sol#L37

## Summary

The contract can receive ETH but lacks a withdraw function, posing a potential issue.

## Vulnerability Detail

The `BridgeReth` contract contains an empty `receive` function that is marked as `payable`. This allows someone to send a transaction with `msg.value > 0`, but there is no mechanism to withdraw these funds from the contract, resulting in them being permanently trapped.

Source: [BridgeReth.sol](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/bridges/BridgeReth.sol#L37)

```solidity
  receive() external payable {}
```

## Impact

- **Impact:** High, as the funds cannot be withdrawn from the contract.
- **Likelihood:** Low, as users would need to send funds by mistake.

Affected users, funds at risk, severe disruption, unavailability, incorrect function, and state not handled appropriately are potential consequences.

## Tool Used

Manual Review

## Recommendation

To address this issue, consider either removing the `receive` function altogether or adding a `revert` statement similar to the `Diamond` contract as shown below:

```solidity
receive() external payable {
    revert("Diamond: Does not accept ether");
}
```

This will prevent unintended ETH deposits from becoming permanently trapped in the contract.

## <a id='M-03'></a>M-03. Market Order ID Overflow and Potential Disruption to Other Market IDs

### Relevant GitHub Links

https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/OwnerFacet.sol#L55

https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/ERC721Facet.sol#L159

https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/ERC721Facet.sol#L276

## Summary

The `OwnerFacet.createMarket` function does not prevent the overflow of the `assetId`, leading to assets with non-unique IDs.

## Vulnerability Detail

The `OwnerFacet.createMarket` function is responsible for creating new markets in the Ditto protocol. When a market is created, several storage variables are set for the market's asset. These variables include the `assetId`, the asset's address in the `assetMapping`, and the asset's address in the assets array. The length of the assets array is used to determine the new `assetId`. However, it's important to note that the `Asset.assetId` is of type `uint8`, which can only hold values from 0 to 255. As a result, `s.assets.length` is cast to `uint8`, which effectively limits the number of markets to a maximum of 255.

The only requirement for this function is that the asset should not already be contained in `s.asset`, but there is no requirement to check if the limit of markets has been reached.

If an attempt is made to create a new market when there are already 255 markets added, the 256th `assetId` will be 0, as `uint8(s.assets.length)` will overflow to 0. This could result in multiple assets having the same `assetId`, potentially causing serious issues in the contract's functionality.

```solidity
function createMarket(address asset, STypes.Asset memory a) external onlyDAO {
    STypes.Asset storage Asset = s.asset[asset];
    // can check non-zero ORDER_ID to prevent creating same asset
    if (Asset.orderId != 0) revert Errors.MarketAlreadyCreated();

    ...

    Asset.assetId = uint8(s.assets.length);
    s.assetMapping[s.assets.length] = asset;
    s.assets.push(asset);

    ...
}
```

## Proof Of Concept

Place the following test case in [test/CreateMarket.t.sol](https://github.com/Cyfrin/2023-09-ditto/tree/a93b4276420a092913f43169a353a6198d3c21b9/test) and run it with the command `forge test --mt testCreateMarketOverflow -vv`.

```solidity
// SPDX-License-Identifier: GPL-3.0-only
pragma solidity 0.8.21;

import {OBFixture} from "test/utils/OBFixture.sol";
import {MTypes, STypes, F, O} from "contracts/libraries/DataTypes.sol";
import {Asset} from "contracts/tokens/Asset.sol";
import {Vault} from "contracts/libraries/Constants.sol";
import "@openzeppelin/contracts/utils/Strings.sol";
import {TestTypes} from "test/utils/TestTypes.sol";
import {console} from "contracts/libraries/console.sol";
import {IERC20Metadata} from "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";

contract CreateMarket is OBFixture {
    function assertIndexAndAssetId(uint index) internal {
        address[] memory assetsArray = diamond.getAssets();

        address assetAddressFromMapping = diamond.getAssetsMapping(index);
        address assetAddressFromArray = assetsArray[index];

        console.log(assetAddressFromMapping);
        console.log(assetAddressFromArray);

        uint assetIdFromArray = diamond
            .getAssetNormalizedStruct(assetsArray[index])
            .assetId;

        console.log(assetIdFromArray);

        assertTrue(
            (assetAddressFromMapping == assetAddressFromArray) &&
                (assetIdFromArray == index)
        );
    }

    function testCreateMarketOverflow() public {
        STypes.Asset memory asset;
        asset.vault = uint8(Vault.CARBON);
        asset.oracle = _ethAggregator;
        asset.initialMargin = 400;
        asset.primaryLiquidationCR = 300;
        asset.secondaryLiquidationCR = 200;
        asset.forcedBidPriceBuffer = 120;
        asset.resetLiquidationTime = 1400;
        asset.secondLiquidationTime = 1000;
        asset.firstLiquidationTime = 800;
        asset.minimumCR = 110;
        asset.tappFeePct = 25;
        asset.callerFeePct = 5;
        asset.minBidEth = 1;
        asset.minAskEth = 1;
        asset.minShortErc = 2000;

        vm.startPrank(owner);

        address lastAdded;

        for (uint i = 1; i <= 256; i++) {
            string memory assetName = string.concat(
                "temp",
                Strings.toString(i)
            );
            Asset temp = new Asset(_diamond, assetName, assetName);

            diamond.createMarket({asset: address(temp), a: asset});
            lastAdded = address(temp);
        }

        address[] memory assetsArray = diamond.getAssets();

        address asset0 = assetsArray[0];
        address asset256 = assetsArray[256];

        uint8 assetId0 = diamond.getAssetNormalizedStruct(asset0).assetId;
        uint8 assetId256 = diamond.getAssetNormalizedStruct(asset256).assetId;

        assertTrue(asset256 == lastAdded);
        assertEq(IERC20Metadata(asset256).name(), "temp256");

        assertTrue(assetId256 == assetId0);

        console.log("firstMarketId:", assetId0);
        console.log("lastMarketId: ", assetId256);

        // The error - assetMapping[Id] == Asset.assetId
        assertIndexAndAssetId(0);
        console.log("-----------");
        assertIndexAndAssetId(256);
    }
}
```

Result:

```
firstMarketId: 0
lastMarketId:  0
0xb27766C6725c2fa64D12DbFdBFB037D5D7C5fDb2
0xb27766C6725c2fa64D12DbFdBFB037D5D7C5fDb2
0
-----------
0x214eD9Da11D2fbe465a6fc601a91E62EbEc1a0D6
0x214eD9Da11D2fbe465a6fc601a91E62EbEc1a0D6
0
Error: Assertion Failed
```

## Impact

The Asset's `assetId` will not be unique, potentially causing serious issues in the protocol's functionality.

The `assetId` is used in the `ERC721Facet` for `ShortRecord` transferring, which will not be possible if there are more than one assets with the same IDs.

## Tool used

Manual Review

## Recommendation

Ensure that when the maximum market number is reached, it will not be possible to create new markets. Add a require statement at the beginning of the market creation function as follows:

```diff
function createMarket(
    address asset,
    STypes.Asset memory a
) external onlyDAO {
+   require(s.assets.length != 255, "Maximum markets number has been reached!");

    STypes.Asset storage Asset = s.asset[asset];
    ...
}
```

## <a id='M-04'></a>M-04. Unfair distribution of fees on withdraw

## Summary

Users are not obligated to withdraw their funds from the same bridge where they made their deposit. This can result in some users encountering higher withdrawal fees, which may seem unfair.

## Vulnerability Detail

The `BridgeRouterFacet` is responsible for routing users' funds between different bridges, currently including `BridgeReth` and `BridgeSteth`. Users have the option to [deposit](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/BridgeRouterFacet.sol#L46-L65) LSTs, which can be in the form of `rETH` or `stETH`, or they can choose to [depositETH](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/BridgeRouterFacet.sol#L67-L86), which will then be converted into LST. In return, users will receive `zETH`. Subsequently, users can [withdraw](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/BridgeRouterFacet.sol#L88-L114) their funds, a process that will [remove](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/BridgeRouterFacet.sol#L111) their `zETH` from the corresponding vault and return the LST to them. Alternatively, they can choose to [unstakeEth](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/BridgeRouterFacet.sol#L116-L140), which will [unstake](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/BridgeRouterFacet.sol#L138) the `zETH` and send them the native ETH.

Both [withdraw](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/BridgeRouterFacet.sol#L96) and [unstakeETH](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/BridgeRouterFacet.sol#L123) may have different fees, depending on the bridge fees, and these fees will be charged to the user for these two operations.

As can be seen in the [OwnerFacet.createBridge](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/OwnerFacet.sol#L282-L283), bridges can be created with or without fees for `withdraw`/`unstakeEth`, with the only restrictions being an upper bound of 15% for withdraw and 2.5% for unstake. This means that one bridge can have fees while the other may not, or one bridge may have lower fees than the other.

Every user can deposit through the bridge of their choice, but they are not obliged to `withdraw`/`unstakeEth` from the same bridge.

Taking this into account, it may not be reasonable to `withdraw`/`unstakeEth` from the same bridge if the fees are higher. Consequently, this situation can lead to the funds from the bridge with lower or no fees being depleted first, forcing subsequent users to `withdraw`/`unstakeEth` from the other bridge with higher fees.

## Proof Of Concept

In the provided test case, Alice and Bob deposited an equal amount of `stETH` and `rETH` in both of the bridges. However, only the `BridgeReth` imposes a withdrawal fee. As a result, when Alice decided to withdraw all of her tokens through the `BridgeSteth` (the bridge without a fee), Bob was compelled to use the other bridge, which incurs significant fees. This distribution of fees is unfair because Alice and Bob deposited an equal amount in both bridges, and therefore, the fees should also be shared during the withdrawal process.

_Place the following test case in [test/Bridges.t.sol](https://github.com/Cyfrin/2023-09-ditto/tree/a93b4276420a092913f43169a353a6198d3c21b9/test) and run it with the command ` forge test --mt test_unfair_fee_distribution -vvv`._

```solidity
pragma solidity 0.8.21;

import {OBFixture} from "test/utils/OBFixture.sol";
import {Vault, Constants} from "contracts/libraries/Constants.sol";

import {console} from "contracts/libraries/console.sol";

contract Bridges is OBFixture {
    address alice = address(0x5000);
    address bob = address(0x5001);

    function setUp() public virtual override {
        super.setUp();

        vm.label(alice, "alice");
        vm.label(bob, "bob");
    }

    function test_unfair_fee_distribution_simplified() external {
        // set fees of the bridges
        vm.startPrank(owner, owner);
        diamond.setWithdrawalFee(_bridgeSteth, 0);
        diamond.setUnstakeFee(_bridgeSteth, 0);

        diamond.setWithdrawalFee(_bridgeReth, 1500);
        diamond.setUnstakeFee(_bridgeReth, 250);
        vm.stopPrank();

        uint88 amount = Constants.MIN_DEPOSIT;

        deal(_steth, alice, amount * 2);
        deal(_reth, alice, amount * 2);
        deal(_steth, bob, amount * 2);
        deal(_reth, bob, amount * 2);

        uint aliceTotalBalance = steth.balanceOf(alice) + reth.balanceOf(alice);
        uint bobTotalBalance = steth.balanceOf(bob) + reth.balanceOf(bob);
        console.log("Alice's balance before:", aliceTotalBalance);
        console.log("Bob's balance before:  ", bobTotalBalance);

        assertTrue(aliceTotalBalance == amount * 4);
        assertTrue(bobTotalBalance == amount * 4);

        uint256 balanceBefore = diamond.getZethTotal(Vault.CARBON);

        // Alice deposits 2e14 tokens in each of the bridges
        vm.startPrank(alice);
        steth.approve(_bridgeSteth, amount * 2);
        diamond.deposit(_bridgeSteth, amount * 2);
        reth.approve(_bridgeReth, amount * 2);
        diamond.deposit(_bridgeReth, amount * 2);
        vm.stopPrank();

        // Bob deposits 2e14 tokens in each of the bridges
        vm.startPrank(bob);
        steth.approve(_bridgeSteth, amount * 2);
        diamond.deposit(_bridgeSteth, amount * 2);
        reth.approve(_bridgeReth, amount * 2);
        diamond.deposit(_bridgeReth, amount * 2);
        vm.stopPrank();

        uint256 balanceAfter = diamond.getZethTotal(Vault.CARBON);

        assertTrue(balanceBefore + amount * 8 == balanceAfter);

        // Alice withdraws all of her amount from the bridgeSteth which has a 0 fee
        vm.prank(alice);
        diamond.withdraw(_bridgeSteth, amount * 4);

        // Assert that the tokens left are only in the bridgeReth, which has a withdrawal fee
        assertTrue(bridgeReth.getZethValue() == amount * 4);
        assertTrue(bridgeSteth.getZethValue() == 0);

        vm.expectRevert();
        vm.prank(bob);
        diamond.withdraw(_bridgeSteth, amount * 2);

        // Bob is forced to withdraw from the bridge with a fee, although he deposited in both
        vm.prank(bob);
        diamond.withdraw(_bridgeReth, amount * 4);

        aliceTotalBalance = steth.balanceOf(alice) + reth.balanceOf(alice);
        bobTotalBalance = steth.balanceOf(bob) + reth.balanceOf(bob);
        console.log("");
        console.log("Alice's balance after: ", aliceTotalBalance);
        console.log("Bob's balance after:   ", bobTotalBalance);
    }
}
```

Result:

```
  Alice's balance before: 400000000000000
  Bob's balance before:   400000000000000

  Alice's balance after:  400000000000000
  Bob's balance after:    340000000000000
```

## Impact

Some users may be subject to higher fees.

## Tool used

Manual Review

## Recommendation

Ensure that the user can `withdraw`/`unstakeEth` only from the bridge in which it was deposited.
Alternatively, require that all of the bridges has equal fees.

## <a id='M-05'></a>M-05. The absence of an expiration deadline option can result in the loss of funds when placing an order on the order book.

## Summary

Functions, such as createAsk, createBid, and createLimitShort, within a smart contract that lacks a crucial deadline parameter. This parameter is commonly used to protect users from unexpected delays in transaction execution and sudden price fluctuations.

## Vulnerability Details

Protocols like Uniswap offer users the ability to set a deadline parameter when swapping tokens. This feature safeguards users against their transactions lingering in the mempool and executing at an unforeseen time, which could result in financial losses.

> Deadline: The Unix time after which a swap will fail, serving as protection against prolonged transaction delays and sudden price fluctuations.

If a user submits a transaction but the gas price increases due to high demand, their transaction may remain in the mempool for an unexpected duration. Over time, the gas price could decrease, making the user's transaction eligible for validators to include in the next block. However, the asset price may have already changed, resulting in the user's order being executed at a less favorable price.

However, the `createAsk`, `createBid`, and `createLimitShort` functions lack this parameter. This omission could potentially lead to the unintended execution of transactions in the future, exposing users to the risk of losing their tokens.

## Impact

Users may encounter unexpected execution of their transactions at an inopportune moment, potentially leading to the loss of their funds.

## Tool Used

Manual Review

## Recommendation

Provide users with the option to specify a deadline parameter when utilizing the `createAsk`, `createBid`, or `createLimitShort` functions.

# Low Risk Findings

## <a id='L-01'></a>L-01. Insufficient Validation of Heartbeat for Oracle Price Feeds

### Relevant GitHub Links

https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibOracle.sol#L71-L73

## Summary

Hardcoded heartbeat value in the `LibOracle.baseOracleCircuitBreaker` function.

## Vulnerability Details

The `LibOracle.baseOracleCircuitBreaker` function is responsible for validating the `getLatestData` function of the base oracle. While the ETH/USD price feed should ideally be used for the base oracle, it lacks sufficient validation for the heartbeat. Currently, a hardcoded heartbeat value of 2 hours (7200 seconds) is set for the ETH/USD price feed, which is not recommended. The recommended heartbeat value is 3600 seconds. Due to this discrepancy, there is a possibility that the `getOraclePrice` function may return stale prices.

You can find the appropriate heartbeat values in [Chainlink's list of Ethereum mainnet price feeds](https://docs.chain.link/data-feeds/price-feeds/addresses/?network=ethereum&page=1&search=eth%2Fusd) by checking the "Show More Details" section.

## Impact

This vulnerability results in insufficient staleness checks for the asset's price.

## Tool Used

Manual Review

## Recommendation

To address this issue, it is recommended to implement proper validation for the price feed. The code modification can be made as follows:

```diff
bool invalidFetchData = roundId == 0 || timeStamp == 0
  || timeStamp > block.timestamp || chainlinkPrice <= 0
- || block.timestamp > timeStamp + 2 hours;
+ || block.timestamp > timeStamp + 1 hour;
```

Ensure that the heartbeat value is set to 3600 seconds for the ETH/USD price feed to avoid returning stale prices.

## <a id='L-02'></a>L-02. Decimal price discrepancies in Chainlink oracles pose a potential risk for users

### Relevant GitHub Links

https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibOracle.sol#L56

## Summary

Chainlink oracles can return prices with different decimal places, potentially resulting in incorrect price calculations.

## Vulnerability Detail

In `LibOracle.getOraclePrice`, if the asset for which the price is being retrieved uses a different decimal place than the base oracle, the price will be calculated based on the base oracle's price and the asset's oracle price. The calculation is straightforward: `oracle's price / base oracle's price`. However, there is a possibility that both oracles have different decimal places, which could lead to incorrect price calculations.

## Impact

Incorrect price calculations can result in financial losses for users.

## Tool Used

Manual Review

## Recommendation

To mitigate this issue, use `AggregatorV3Interface.decimals()` to ensure that both oracles have the same number of decimal places. This will help ensure accurate price calculations.
