# [H-01] Incorrect hardcoded addresses in the oracles constructors

## Summary

There are incorrect hardcoded addresses in the oracles [StableOracleDAI](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleDAI.sol#L23-L31), [StableOracleWBGL](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleWBGL.sol#L17-L22) and [StableOracleWBTC](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleWBTC.sol#L17) constructors.

## Vulnerability Detail

Incorrect addresses in [StableOracleDAI](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleDAI.sol#LL28C18-L28C18) and [StableOracleWBGL](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleWBGL.sol#L19) will always revert because the contracts on these addresses don't have `quoteSpecificPoolsWithTimePeriod` function.

If the zero address is used in [StableOracleDAI](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleDAI.sol#LL30C33-L30C33), the `getPriceUSD` function will always revert.

The incorrect address in [StableOracleWBTC](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleWBTC.sol#L17) will result in an incorrect price because the given address corresponds to the ETH/USD price feed.

## Impact

Incorrect addresses in the oracles constructors will require redeployment of the oracles contracts or break the entire price calculation logic.

## Code Snippet

https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleDAI.sol#L23-L31
https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleWBGL.sol#L17-L22
https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleWBTC.sol#L15-L19

## Tool used

Manual Review

## Recommendation

For the [DAIEthOracle](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleDAI.sol#LL28C25-L28C25), you can use `0xB210CE856631EeEB767eFa666EC7C1C57738d438`. Alternatively, you can deploy your static oracle and use it.
For the [ethOracle](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleDAI.sol#L30), replace the `0x0` address with the already deployed StableOracleWETH.

```diff
File: DAIEthOracle.sol
-constructor() {
+constructor(address _ethOracle, address _staticOracleUniV3) {
        priceFeedDAIETH = AggregatorV3Interface(
            0x773616E4d11A78F511299002da57A0a94577F1f4
        );
        DAIEthOracle = IStaticOracle(
-            0x982152A6C7f732Ec7C9EA998dDD9Ebde00Dfa16e
+            0xB210CE856631EeEB767eFa666EC7C1C57738d438 // or use _staticOracleUniV3
        );
-        ethOracle = IStableOracle(0x0000000000000000000000000000000000000000); // TODO: WETH oracle price
+       ethOracle = IStableOracle(_ethOracle); // TODO: WETH oracle price
}
```

For the [StableOracleWBGL](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleWBGL.sol#LL19C13-L19C55), you can use `0xB210CE856631EeEB767eFa666EC7C1C57738d438`. Alternatively, you can deploy your static oracle and use it.

```diff
File: StableOracleWBGL.sol
constructor(address _WETHoracle) {
+constructor(address _WETHoracle, address _staticOracleUniV3) {
        staticOracleUniV3 = IStaticOracle(
-            0x982152A6C7f732Ec7C9EA998dDD9Ebde00Dfa16e
+            0xB210CE856631EeEB767eFa666EC7C1C57738d438 // or use _staticOracleUniV3
        );
        ethOracle = IStableOracle(_WETHoracle);
}
```

For the [StableOracleWBTC](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleWBTC.sol#LL17C40-L17C40) replace `0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419` (ETH/USD) with `0xF4030086522a5bEEa4988F8cA5B36dbC97BeE88c` (BTC/USD) price feed

```diff
File: StableOracleWBTC.sol
constructor() {
        priceFeed = AggregatorV3Interface(
-           0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419
+           0xF4030086522a5bEEa4988F8cA5B36dbC97BeE88c
        );
}
```

# [M-01] Chainlink's latestRoundData return stale or incorrect result

## Summary

Insufficient round completeness checking could result in stale prices, incorrect price return values, or outdated prices. Functions dependent on precise price feeds may not perform as expected, potentially leading to financial losses.

## Vulnerability Detail

The `getPriceUSD()` function in the oracle wrappers relies on the `latestRoundData()` function from the oracle to fetch the token's price. However, there is a lack of checks in place to ensure data integrity and accuracy.

As per Chainlink's documentation, this function does not throw an error if no answer has been reached but instead returns `0` or outdated round data. The external Chainlink oracle, which supplies index price information to the system, introduces inherent risk due to reliance on third-party data sources. For instance, the oracle may experience delays or fail to be properly maintained, resulting in the usage of outdated data for index price calculations. Dependence on oracles has historically led to impaired on-chain systems, and complications that contribute to these outcomes can arise from factors as simple as network congestion.

## Impact

In the event of a problem with Chainlink initiating a new round and reaching a consensus on the new value for the oracle (such as Chainlink nodes abandoning the oracle, chain congestion, or vulnerabilities/attacks on the Chainlink system), consumers of this contract may persistently rely on outdated and stale data if no new round is initiated.

This situation can result in incorrect price return values, stale prices, and outdated price information, ultimately disrupting the entire price calculation logic.

## Code Snippet

https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleDAI.sol#L48
https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleWBTC.sol#L23
https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleWETH.sol#L23

## Tool used

Manual Review

## Recommendation

Please ensure that all functions utilizing Chainlink price feeds include appropriate checks.

```diff
function getPriceUSD() {
...
-  (, int256 price, , , ) = priceFeed.latestRoundData();
+  (uint80 roundID, int signedPrice, , uint updatedAt, uint80 answeredInRound) = _priceFeed.latestRoundData();

+  require(signedPrice > 0, "Negative Oracle Price");
+  require(updatedAt > 0, "Round is not complete");
+  require(answeredInRound >= roundID, "round not complete");
...
}
```
