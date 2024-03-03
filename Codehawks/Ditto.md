# DittoETH - Findings Report

# Table of contents

- ## Medium Risk Findings
  - ### [M-01. Insufficient staleness validation in LibOracle's oracleCircuitBreaker function](#M-01)
  - ### [M-02. Market Order ID Overflow and Potential Disruption to Other Market IDs](#M-02)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: Ditto

### Dates: Sep 8th, 2023 - Oct 9th, 2023

[See more contest details here](https://www.codehawks.com/contests/clm871gl00001mp081mzjdlwc)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 0
- Medium: 2
- Low: 0

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

## <a id='M-02'></a>M-02. Market Order ID Overflow and Potential Disruption to Other Market IDs

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
