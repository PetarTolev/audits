# The Standard - Findings Report

# Table of contents

- ## High Risk Findings
  - ### [H-01. Possible `LiquidithPool.pendingStakes` DOS, due to unbounded array length](#H-01)
  - ### [H-02. Missing proper access control in the `LiquidationPool.distributeAssets` function](#H-02)
- ## Medium Risk Findings
  - ### [M-01. The `SmartVaultV3.swap` function accepts any slippage, leading to the potential theft of all collateral tokens from the contract.](#M-01)
  - ### [M-02. Fixed Fee Level Usage During Token Swaps on Uniswap](#M-02)
- ## Low Risk Findings
  - ### [L-01. Calculating asset distribution is susceptible to precision loss due to division before multiplication](#L-01)
  - ### [L-02. Anyone can frontrun `LiquidationPoolManager.distributeFees()` and steal profit](#L-02)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: The Standard

### Dates: Dec 27th, 2023 - Jan 10th, 2024

[See more contest details here](https://www.codehawks.com/contests/clql6lvyu0001mnje1xpqcuvl)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 2
- Medium: 2
- Low: 2

# High Risk Findings

## <a id='H-01'></a>H-01. Possible `LiquidithPool.pendingStakes` DOS, due to unbounded array length

### Relevant GitHub Links

https://github.com/Cyfrin/2023-12-the-standard/blob/41147c652c41489bd30953a4122a1e9d836799b1/contracts/LiquidationPool.sol#L119-L132

https://github.com/Cyfrin/2023-12-the-standard/blob/41147c652c41489bd30953a4122a1e9d836799b1/contracts/LiquidationPool.sol#L136

https://github.com/Cyfrin/2023-12-the-standard/blob/41147c652c41489bd30953a4122a1e9d836799b1/contracts/LiquidationPool.sol#L150

https://github.com/Cyfrin/2023-12-the-standard/blob/41147c652c41489bd30953a4122a1e9d836799b1/contracts/LiquidationPool.sol#L206

## Summary

A vulnerability exists in the `LiquidithPool.increasePosition` function, which permits a malicious user to repeatedly create positions with minimal value. This leads to a bloated `LiquidithPool.pendingStakes` array, potentially causing a Denial of Service on the `LiquidithPool` contract and impairing its core functions.

## Vulnerability Details

The `increasePosition` function in the `liquidationPool` contract allows for the deposit of EUROs or TST tokens. Users must wait one day for their stake to be consolidated. The functions `increasePosition` and `decreasePosition` invoke `consolidatePendingStakes`, an expensive operation due to multiple storage read/write processes.

The `increasePosition` process does not check or limit the amount of the new position. This oversight allows a malicious user to indefinitely create pending stakes with trivial amounts (as little as 1 wei), potentially causing an out-of-gas exception for any function interacting with the `pendingStakes` array.

A simple Proof of Concept demonstrates how a malicious user can flood the `pendingStakes` array with numerous low-valued stakes.

```javascript
it("DOS of LiquidationPool", async () => {
  await TST.mint(attacker.address, 1000000);
  await TST.connect(attacker).approve(LiquidationPool.address, 1000000);

  for (let i = 0; i < 1000; i++) {
    await LiquidationPool.connect(attacker).increasePosition(1, 0);
  }
});
```

## Impact

The `LiquidithPool` contract is integral to the protocol, especially in the liquidation process. The DOS vulnerability has several significant impacts:

1. `consolidatePendingStakes` becomes inoperable, affecting `increasePosition`, `decreasePosition`, and `distributeAssets`.
2. `decreasePosition`, `distributeAssets`, and `distributeFees` also become inoperable, trapping tokens in the `liquidationPool`.
3. The inoperability of `distributeFees` and `distributeAssets` hinders the liquidation process.

The severity of this attack is high due to the potential for funds to be trapped in the contract. The likelihood of exploitation is also high, given the low cost of executing the attack.

## Tools Used

Manual Review

## Recommendations

Several mitigations can be implemented:

1. Enforce a minimum amount requirement for each deposit to prevent positions with very low values.
2. Aggregate positions from the same `msg.sender` to reduce the number of entries in the `pendingStakes` array.

## <a id='H-02'></a>H-02. Missing proper access control in the `LiquidationPool.distributeAssets` function

### Relevant GitHub Links

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/LiquidationPool.sol#L205

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/LiquidationPoolManager.sol#L80

## Summary

Missing proper access control in the `LiquidationPool.distributeAssets` function.

## Vulnerability Details

It appears that the `LiquidationPool.distributeAssets` function is intended to be exclusively callable by the `LiquidationPoolManager` as part of the liquidation process initiated by `LiquidationPoolManager.runLiquidation`. However, currently, the `LiquidationPool.distributeAssets` function lacks access control. This oversight allows any user to call it with arbitrary parameters. These parameters include:

1. `_assets` array, which specifies the tokens and amounts to be distributed along with the price feed for price determination.
2. `_collateralRate`, utilized in calculating the `costInEuros`.
3. `_hundredPC`, a factor in the `costInEuros` calculation, which must be a constant (1e5)

## Impact

The `LiquidationPool.distributeAssets` function requires token approval by the `LiquidationPoolManager` before transferring tokens from it. Because of this, no immediate vulnerabilities compromising the protocol are currently evident. However, future upgrades may introduce significant issues if this function remains unregulated.

## Tools Used

Manual Review

## Recommendations

To mitigate this issue, it is recommended to implement the `onlyManager` modifier in the `distributeAssets` function.

```diff
-   function distributeAssets(ILiquidationPoolManager.Asset[] memory _assets, uint256 _collateralRate, uint256 _hundredPC) external payable {
+   function distributeAssets(ILiquidationPoolManager.Asset[] memory _assets, uint256 _collateralRate, uint256 _hundredPC) external payable onlyManager {
        ...
}
```

# Medium Risk Findings

## <a id='M-01'></a>M-01. The `SmartVaultV3.swap` function accepts any slippage, leading to the potential theft of all collateral tokens from the contract.

### Relevant GitHub Links

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/SmartVaultV3.sol#L214-L231

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/SmartVaultV3.sol#L199

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/SmartVaultV3.sol#L210-L211

## Summary

The `SmartVaultV3.swap` function lacks slippage protection, making it vulnerable to sandwich attacks, potentially leading to the loss of swapped locked collateral tokens.

## Vulnerability Details

In some cases the `SmartVaultV3.swap()` function uses `amountOutMinimum` and `sqrtPriceLimitX96` set to 0, enabling MEV or sandwich attacks.

From [Uniswap's documentation](https://docs.uniswap.org/contracts/v3/guides/swaps/single-swaps#swap-input-parameters):

- `amountOutMinimum` is advised to be non-zero in production to safeguard against price manipulation.
- `sqrtPriceLimitX96` being zero deactivates this parameter. In production, it should be set to manage price impact and facilitate other price-related mechanisms.

The `SmartVaultV3.swap()` function, callable by the vault owner, internally calls [calculateMinimumAmountOut](https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/SmartVaultV3.sol#L206-L212) to calculate `amountOutMinimum` based on collateral value and swap token value. If [the locked collateral tokens' value - the swapped token value](https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/SmartVaultV3.sol#L209) exceeds [requiredCollateralValue](https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/SmartVaultV3.sol#L208), the function returns 0. For example, this can happen when the ETH price bumps. This lack of slippage protection is critical, particularly when the value of tokens like ETH rises sharply.

A potential attack involves using a flash loan to manipulate the WETH/USDC pool on Uniswap, allowing the attacker to drain all USDC from the SmartVaultV3 contract. This method can also target other ERC-20 tokens in the contract.

For instance, to drain all USDC, consider the following proof-of-concept:

- The attacker borrows a flash loan in USDC and buys WETH from Uniswap's WETH/USDC pool.
- The attacker executes the `SmartVaultV3.swap` function with USDC.
- The `SmartVaultV3.swap` will use all the locked USDC to buy WETH at a very high price.
- The attacker then sells the previously obtained WETH for USDC in the same pool and repays the flash loan.
- The attacker takes all the locked USDC as profit.

### Coded Proof of Concept

Place this test in the `smartVault.js` file and execute it by `POC: slippage, amountOutMinimum = 0`

```javascript
it("POC: slippage, amountOutMinimum = 0", async () => {
  // user vault has 1 ETH collateral
  await user.sendTransaction({ to: Vault.address, value: ethers.utils.parseEther("1") });
  // user borrows 1200 EUROs
  const borrowValue = ethers.utils.parseEther("1200");
  await Vault.connect(user).mint(user.address, borrowValue);

  // price of eth increased
  await ClEthUsd.setPrice(BigNumber.from(400000000000));

  const inToken = ethers.utils.formatBytes32String("ETH");
  const outToken = ethers.utils.formatBytes32String("sUSD");

  // user is swapping .5 ETH
  const swapValue = ethers.utils.parseEther("0.5");
  await Vault.connect(user).swap(inToken, outToken, swapValue);

  const { amountOutMinimum } = await SwapRouterMock.receivedSwap();

  expect(amountOutMinimum).to.equal(0);
});
```

Furthermore, the `deadline` parameter is set to `block.timestamp`, allowing token swaps at any block number without an expiration deadline.
Another concern is the constant pool `fee` of 3000 (0.3%). This inflexibility could lead to liquidity shifts and higher MEV attack risks due to lower liquidity tiers.

## Impact

This vulnerability, particularly during a significant price increase of locked collateral tokens, allows an attacker to deplete the `SmartVaultV3` of various cryptocurrencies. It's considered a high-risk issue due to specific required conditions but can lead to substantial financial losses.

Additionally, without an expiration deadline, miners or validators could manipulate transactions for their benefit, increasing the risk of fund loss due to slippage.

## Tools Used

Manual Review

## Recommendations

To mitigate this vulnerability, it's advised not to use 0 as `amountOutMinimum` during swaps.
Given that the `swap` function is restricted to the vault owner, they should be able to set parameters like `amountOutMinimum`, `sqrtPriceLimitX96`, `deadline`, and `fee` to enhance security and control slippage.

```diff
-   function swap(bytes32 _inToken, bytes32 _outToken, uint256 _amount) external onlyOwner {
+   function swap(bytes32 _inToken, bytes32 _outToken, uint256 _amount, uint256 _amountOutMinimum, uint256 _sqrtPriceLimitX96, uint256 _deadline, uint256 _fee) external onlyOwner {
        uint256 swapFee = _amount * ISmartVaultManagerV3(manager).swapFeeRate() / ISmartVaultManagerV3(manager).HUNDRED_PC();
        address inToken = getSwapAddressFor(_inToken);
        uint256 minimumAmountOut = calculateMinimumAmountOut(_inToken, _outToken, _amount);
        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter.ExactInputSingleParams({
                tokenIn: inToken,
                tokenOut: getSwapAddressFor(_outToken),
-               fee: 3000,
+               fee: _fee,
                recipient: address(this),
-               deadline: block.timestamp,
+               deadline: _deadline,
                amountIn: _amount - swapFee,
-               amountOutMinimum: minimumAmountOut,
+               amountOutMinimum: _minimumAmountOut,
-               sqrtPriceLimitX96: 0
+               sqrtPriceLimitX96: _sqrtPriceLimitX96
            });
        inToken == ISmartVaultManagerV3(manager).weth() ?
            executeNativeSwapAndFee(params, swapFee) :
            executeERC20SwapAndFee(params, swapFee);
    }
```

## <a id='M-02'></a>M-02. Fixed Fee Level Usage During Token Swaps on Uniswap

### Relevant GitHub Links

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/SmartVaultV3.sol#L221

## Summary

Fixed fee level is used when swap tokens on Uniswap.

## Vulnerability Details

The `SmartVaultV3.swap` function is designed for swapping collateral tokens. It constructs the `ISwapRouter.ExactInputSingleParams` with a constant fee of 3000 (0.3%). However, not all Uniswap pools operate at this fee level. For instance, the XMON/ETH pool on Mainnet (0x59b4bb1f5d943cf71a10df63f6b743ee4a4489ee) has a 1% fee, and the WETH/BOB pool on Optimism (0x1a54ae9f662b463f8d432482975c17e51518b50d) charges 0.05%.

```javascript
ISwapRouter.ExactInputSingleParams memory params = ISwapRouter
    .ExactInputSingleParams({
        tokenIn: inToken,
        tokenOut: getSwapAddressFor(_outToken),
        fee: 3000,
        recipient: address(this),
        deadline: block.timestamp,
        amountIn: _amount - swapFee,
        amountOutMinimum: minimumAmountOut,
        sqrtPriceLimitX96: 0
    });
```

## Impact

This fixed fee approach poses several issues:

1. It restricts swaps to pools with a 0.3% fee, potentially leading to suboptimal liquidity utilization. For example, the USDC/ETH pool with 0.05% fees has \$161.32 million in TVL, compared to \$71.89 million in the 0.3% fee pool. Higher liquidity pools could reduce slippage, offering better rates for users. A fixed fee can lead to missed opportunities for optimal swaps.

2. The approach may cause transactions to revert when attempting to swap in non-existent pools with the fixed fee level.

## Tools Used

Manual Review

## Recommendations

Modify the `SmartVaultV3.swap` function to accept a variable fee parameter, enhancing flexibility and addressing the outlined issues. This change involves passing the fee as a parameter to the function, allowing for dynamic fee adjustments based on pool characteristics.

```diff
-   function swap(bytes32 _inToken, bytes32 _outToken, uint256 _amount) external onlyOwner {
+   function swap(bytes32 _inToken, bytes32 _outToken, uint256 _amount, uint256 _fee) external onlyOwner {
        uint256 swapFee = _amount * ISmartVaultManagerV3(manager).swapFeeRate() / ISmartVaultManagerV3(manager).HUNDRED_PC();
        address inToken = getSwapAddressFor(_inToken);
        uint256 minimumAmountOut = calculateMinimumAmountOut(_inToken, _outToken, _amount);
        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter.ExactInputSingleParams({
                tokenIn: inToken,
                tokenOut: getSwapAddressFor(_outToken),
-               fee: 3000,
+               fee: _fee,
                recipient: address(this),
                deadline: block.timestamp,
                amountIn: _amount - swapFee,
                amountOutMinimum: minimumAmountOut,
                sqrtPriceLimitX96: 0
            });
        inToken == ISmartVaultManagerV3(manager).weth() ?
            executeNativeSwapAndFee(params, swapFee) :
            executeERC20SwapAndFee(params, swapFee);
    }
```

By adopting this recommendation, the SmartVaultV3 will be able to interact more effectively with a broader range of liquidity pools, improving swap efficiency and potentially reducing transaction failures.

# Low Risk Findings

## <a id='L-01'></a>L-01. Calculating asset distribution is susceptible to precision loss due to division before multiplication

### Relevant GitHub Links

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/LiquidationPool.sol#L219-L221

## Summary

In the `LiquidationPool.distributeAssets` function, there is a potential for rewards loss caused by precision loss in the `costInEuros` calculation.

## Vulnerability Details

The `LiquidationPool.distributeAssets` function calculates `_portion` by using a combination of multiplication and division operations. The initial computation of `_portion` is based on the formula: `assetAmount * positionStaked / stakeTotal`. This value is subsequently utilized in the `costInEuros` equation, which expands to:
`assetAmount * positionStaked / stakeTotal * decimals * assetPriceUsd / priceEurUsd * _hundredPC / _collateralRate`.

The issue arises due to several divisions occurring before multiplications in this formula, potentially leading to significant precision loss, and in some cases, resulting in a calculation that equals 0.

## Impact

The sequence of division before multiplication can result in the loss of token rewards, impacting the accuracy and fairness of the distribution process.

## Tools Used

Manual Review

## Recommendations

It is advisable to restructure the calculation process by performing all multiplications first, followed by the divisions. This approach can minimize precision loss, ensuring more accurate and equitable asset distribution.

## <a id='L-02'></a>L-02. Anyone can frontrun `LiquidationPoolManager.distributeFees()` and steal profit

### Relevant GitHub Links

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/LiquidationPool.sol#L191

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/LiquidationPool.sol#L149-L161

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/LiquidationPool.sol#L137

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/LiquidationPoolManager.sol#L38

## Summary

Anyone can frontrun `LiquidationPoolManager.distributeFees()` and steal profit from the other users which have positions.

## Vulnerability Details

The `LiquidationPoolManager.distributeFees()` function gives approval to the ``LiquidationPool to access all EUROs tokens in the contract. It then executes `LiquidationPool.distributeFees`and subsequently transfers the remaining EUROs to the protocol's treasury account. The process involves`LiquidationPool.distributeFees`transferring EUROs from`LiquidationPoolManager`. It then iterates through all holders, dividing the EUROs proportionally based on each holder's position in TST, using the formula: `(EURO amount \* holder's TST position) / tstTotal`. Similarly, EUROs are allocated to `pendingStakes`using a related formula. However, the function does not account for the duration for which these pending stakes have existed. This oversight potentially allows anyone to exploit the system by front-running the`LiquidationPoolManager.distributeFees()`function. They could receive a portion of the EUROs and then use`decreasePosition` to withdraw their deposited TST amount plus the received EUROs.

### Coded POC:

Please follow the `@audit` tags for an explanation of the attack

```javascript
it("POC frontrun distributeFees", async () => {
  // #region @audit users' deposits

  const tstPosition1Value = ethers.utils.parseEther("1250");
  await TST.mint(holder1.address, tstPosition1Value);
  await TST.connect(holder1).approve(LiquidationPool.address, tstPosition1Value);
  await LiquidationPool.connect(holder1).increasePosition(tstPosition1Value, 0);

  const tstPosition2Value = ethers.utils.parseEther("7000");
  await TST.mint(holder2.address, tstPosition2Value);
  await TST.connect(holder2).approve(LiquidationPool.address, tstPosition2Value);
  await LiquidationPool.connect(holder2).increasePosition(tstPosition2Value, 0);

  const tstPosition3Value = ethers.utils.parseEther("85000");
  await TST.mint(holder3.address, tstPosition3Value);
  await TST.connect(holder3).approve(LiquidationPool.address, tstPosition3Value);
  await LiquidationPool.connect(holder3).increasePosition(tstPosition3Value, 0);

  const tstPosition4Value = ethers.utils.parseEther("800");
  await TST.mint(holder4.address, tstPosition4Value);
  await TST.connect(holder4).approve(LiquidationPool.address, tstPosition4Value);
  await LiquidationPool.connect(holder4).increasePosition(tstPosition4Value, 0);

  const tstPosition5Value = ethers.utils.parseEther("600000");
  await TST.mint(holder5.address, tstPosition5Value);
  await TST.connect(holder5).approve(LiquidationPool.address, tstPosition5Value);
  await LiquidationPool.connect(holder5).increasePosition(tstPosition5Value, 0);

  // #endregion @audit users' deposits

  // @audit Fast forward 1 day, to consolidate all users' pending stakes
  await fastForward(DAY);

  // #region @audit FRONTRUN: just before the `distributeFees` call, a malicious user deposits more TST than all other pendingStakes
  const tstAttackValue = ethers.utils.parseEther("700000");
  await TST.mint(attacker.address, tstAttackValue);
  await TST.connect(attacker).approve(LiquidationPool.address, tstAttackValue);
  await LiquidationPool.connect(attacker).increasePosition(tstAttackValue, 0);
  // #region @audit Just before the `distributeFees` call, a malicious user deposits more TST than all other pendingStakes

  // @audit mint EUROs which will be distributed
  const feeBalance = ethers.utils.parseEther("1000");
  await EUROs.mint(LiquidationPoolManager.address, feeBalance);

  // @audit `LiquidationPoolManager.distributeFees` calls `LiquidationPool.distributeFees` which distributes all the EUROs among the existing positions and pending stakes (at that moment only the attacker's pending stake)
  await LiquidationPoolManager.distributeFees();

  // @audit Fast forward 1 day, to consolidate all users' pending stakes (at that moment only the attacker's pending stake)
  await fastForward(DAY);

  // @audit attacker can decrease his position to withdraw all the TST + the profit in the form of EUROs
  await LiquidationPool.connect(attacker).decreasePosition(
    ethers.utils.parseUnits("700000000000000000000000", "wei"), // @audit withdraw all deposited TST
    ethers.utils.parseUnits("251067034898317850866", "wei") // @audit-issue reward
  );
});
```

## Impact

Loss of funds for the all other users which have already consolidated stakes.

## Tools Used

Manual Review

## Recommendations

When `distributeFees` split the reward only among already consolidated stakes which are created as `positions`.

```diff
    function distributeFees(uint256 _amount) external onlyManager {
        uint256 tstTotal = getTstTotal();
        if (tstTotal > 0) {
            IERC20(EUROs).safeTransferFrom(msg.sender, address(this), _amount);
            for (uint256 i = 0; i < holders.length; i++) {
                address _holder = holders[i];
                positions[_holder].EUROs += _amount * positions[_holder].TST / tstTotal;
            }
-           for (uint256 i = 0; i < pendingStakes.length; i++) {
-               pendingStakes[i].EUROs += _amount * pendingStakes[i].TST / tstTotal;
-           }
        }
    }
```
