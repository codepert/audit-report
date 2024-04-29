# Title 1: Interest rates are incorrect on Liquidation

## Severity

High

## Impact

The debt tokens are being transferred before calculating the interest rates. But the interest rate calculation function assumes that debt token has not yet been sent thus the outcome `currentLiquidityRate` will be incorrect.

## Proof of Concept

1. Liquidator L1 calls executeLiquidateERC20 for a position whose health factor <1.

```solidity
function executeLiquidateERC20(
        mapping(address => DataTypes.ReserveData) storage reservesData,
        mapping(uint256 => address) storage reservesList,
        mapping(address => DataTypes.UserConfigurationMap) storage usersConfig,
        DataTypes.ExecuteLiquidateParams memory params
    ) external returns (uint256) {
...
 _burnDebtTokens(liquidationAssetReserve, params, vars);
...
}
```

2. This internally calls `_burnDebtTokens`

```solidity
    function _burnDebtTokens(
        DataTypes.ReserveData storage liquidationAssetReserve,
        DataTypes.ExecuteLiquidateParams memory params,
        ExecuteLiquidateLocalVars memory vars
    ) internal {
       ...
        // Transfers the debt asset being repaid to the xToken, where the liquidity is kept
        IERC20(params.liquidationAsset).safeTransferFrom(
            vars.payer,
            vars.liquidationAssetReserveCache.xTokenAddress,
            vars.actualLiquidationAmount
        );
...
        // Update borrow & supply rate
        liquidationAssetReserve.updateInterestRates(
            vars.liquidationAssetReserveCache,
            params.liquidationAsset,
            vars.actualLiquidationAmount,
            0
        );
    }
```

3. Basically first it transfers the debt asset to `xToken` using below. This increases the balance of `xTokenAddress` for `liquidationAsset`

```
IERC20(params.liquidationAsset).safeTransferFrom(
            vars.payer,
            vars.liquidationAssetReserveCache.xTokenAddress,
            vars.actualLiquidationAmount
        );
```

4. Now `updateInterestRates` function is called on ReserveLogic.sol#L169

```
function updateInterestRates(
        DataTypes.ReserveData storage reserve,
        DataTypes.ReserveCache memory reserveCache,
        address reserveAddress,
        uint256 liquidityAdded,
        uint256 liquidityTaken
    ) internal {
...
(
            vars.nextLiquidityRate,
            vars.nextVariableRate
        ) = IReserveInterestRateStrategy(reserve.interestRateStrategyAddress)
            .calculateInterestRates(
                DataTypes.CalculateInterestRatesParams({
                    liquidityAdded: liquidityAdded,
                    liquidityTaken: liquidityTaken,
                    totalVariableDebt: vars.totalVariableDebt,
                    reserveFactor: reserveCache.reserveFactor,
                    reserve: reserveAddress,
                    xToken: reserveCache.xTokenAddress
                })
            );
...
}
```

5. Finally call to `calculateInterestRates` function on DefaultReserveInterestRateStrategy#L127 contract is made which calculates the interest rate.

```
function calculateInterestRates(
        DataTypes.CalculateInterestRatesParams calldata params
    ) external view override returns (uint256, uint256) {
...
if (vars.totalDebt != 0) {
            vars.availableLiquidity =
                IToken(params.reserve).balanceOf(params.xToken) +
                params.liquidityAdded -
                params.liquidityTaken;
            vars.availableLiquidityPlusDebt =
                vars.availableLiquidity +
                vars.totalDebt;
            vars.borrowUsageRatio = vars.totalDebt.rayDiv(
                vars.availableLiquidityPlusDebt
            );
            vars.supplyUsageRatio = vars.totalDebt.rayDiv(
                vars.availableLiquidityPlusDebt
            );
        }
...
vars.currentLiquidityRate = vars
            .currentVariableBorrowRate
            .rayMul(vars.supplyUsageRatio)
            .percentMul(
                PercentageMath.PERCENTAGE_FACTOR - params.reserveFactor
            );
        return (vars.currentLiquidityRate, vars.currentVariableBorrowRate);
}
```

6. As we can see in above code, `vars.availableLiquidity` is calculated as `IToken(params.reserve).balanceOf(params.xToken) +params.liquidityAdded - params.liquidityTaken`.

7. But the problem is that debt token is already transferred to `xToken` which means `xToken` already consist of `params.liquidityAdded`. Hence the calculation ultimately becomes `(xTokenBeforeBalance+params.liquidityAdded) +params.liquidityAdded - params.liquidityTaken`.

8. This is incorrect and would lead to higher `vars.availableLiquidity` which ultimately impacts the `currentLiquidityRate`.


## Recommended Mitigation Steps

Transfer the debt asset post interest calculation
```solidity
function _burnDebtTokens(
        DataTypes.ReserveData storage liquidationAssetReserve,
        DataTypes.ExecuteLiquidateParams memory params,
        ExecuteLiquidateLocalVars memory vars
    ) internal {
IPToken(vars.liquidationAssetReserveCache.xTokenAddress)
            .handleRepayment(params.liquidator, vars.actualLiquidationAmount);
        // Burn borrower's debt token
        vars
            .liquidationAssetReserveCache
            .nextScaledVariableDebt = IVariableDebtToken(
            vars.liquidationAssetReserveCache.variableDebtTokenAddress
        ).burn(
                params.borrower,
                vars.actualLiquidationAmount,
                vars.liquidationAssetReserveCache.nextVariableBorrowIndex
            );
liquidationAssetReserve.updateInterestRates(
            vars.liquidationAssetReserveCache,
            params.liquidationAsset,
            vars.actualLiquidationAmount,
            0
        );
IERC20(params.liquidationAsset).safeTransferFrom(
            vars.payer,
            vars.liquidationAssetReserveCache.xTokenAddress,
            vars.actualLiquidationAmount
        );
...
...
}
```

# Title 2: Discrepency in the Uniswap V3 position price calculation because of decimals

## Severity

High

## Impact

When the squared root of the Uniswap V3 position is calculated from the `_getOracleData()` function, the price may return a very high number (in the case that the token1 decimals are strictly superior to the token0 decimals).

The reason is that at the denominator, the `1E9` (`10**9`) value is hard-coded, but should take into account the delta between both decimals.

As a result, in the case of` token1Decimal > token0Decimal`, the `getAmountsForLiquidity()` is going to return a huge value for the amount of token0 and token1 as the user position liquidity.

The `getTokenPrice()`, using this amount of liquidity to calculate the token price is as its turn going to return a huge value.

## Proof of Concept

This POC demonstrates in which case the returned squared root price of the position is over inflated

```solidity
// SPDX-License-Identifier: UNLISENCED
pragma solidity 0.8.10;
import {SqrtLib} from "../contracts/dependencies/math/SqrtLib.sol";
import "forge-std/Test.sol";
contract Audit is Test {
    function testSqrtPriceX96() public {
        // ok
        uint160 price1 = getSqrtPriceX96(1e18, 5 * 1e18, 18, 18);
        // ok
        uint160 price2 = getSqrtPriceX96(1e18, 5 * 1e18, 18, 9);
        // Has an over-inflated squared root price by 9 magnitudes as token0Decimal < token1Decimal
        uint160 price3 = getSqrtPriceX96(1e18, 5 * 1e18, 9, 18);
    }
    function getSqrtPriceX96(
        uint256 token0Price,
        uint256 token1Price,
        uint256 token0Decimal,
        uint256 token1Decimal
    ) private view returns (uint160 sqrtPriceX96) {
        if (oracleData.token1Decimal == oracleData.token0Decimal) {
            // multiply by 10^18 then divide by 10^9 to preserve price in wei
            sqrtPriceX96 = uint160(
                (SqrtLib.sqrt(((token0Price * (10**18)) / (token1Price))) *
                    2**96) / 1E9
            );
        } else if (token1Decimal > token0Decimal) {
            // multiple by 10^(decimalB - decimalA) to preserve price in wei
            sqrtPriceX96 = uint160(
                (SqrtLib.sqrt(
                    (token0Price * (10**(18 + token1Decimal - token0Decimal))) /
                        (token1Price)
                ) * 2**96) / 1E9
            );
        } else {
            // multiple by 10^(decimalA - decimalB) to preserve price in wei then divide by the same number
            sqrtPriceX96 = uint160(
                (SqrtLib.sqrt(
                    (token0Price * (10**(18 + token0Decimal - token1Decimal))) /
                        (token1Price)
                ) * 2**96) / 10**(9 + token0Decimal - token1Decimal)
            );
        }
    }
}
```

## Recommended Mitigation Steps

```solidity
    if (oracleData.token1Decimal == oracleData.token0Decimal) {
        // multiply by 10^18 then divide by 10^9 to preserve price in wei
        oracleData.sqrtPriceX96 = uint160(
            (SqrtLib.sqrt(
                ((oracleData.token0Price * (10**18)) /
                    (oracleData.token1Price))
            ) * 2**96) / 1E9
        );
    } else if (oracleData.token1Decimal > oracleData.token0Decimal) {
        // multiple by 10^(decimalB - decimalA) to preserve price in wei
        oracleData.sqrtPriceX96 = uint160(
            (SqrtLib.sqrt(
                (oracleData.token0Price *
                    (10 **
                        (18 +
                            oracleData.token1Decimal -
                            oracleData.token0Decimal))) /
                    (oracleData.token1Price)
            ) * 2**96) /
                10 **
                    (9 +
                        oracleData.token1Decimal -
                        oracleData.token0Decimal)
        );
    } else {
        // multiple by 10^(decimalA - decimalB) to preserve price in wei then divide by the same number
        oracleData.sqrtPriceX96 = uint160(
            (SqrtLib.sqrt(
                (oracleData.token0Price *
                    (10 **
                        (18 +
                            oracleData.token0Decimal -
                            oracleData.token1Decimal))) /
                    (oracleData.token1Price)
            ) * 2**96) /
                10 **
                    (9 +
                        oracleData.token0Decimal -
                        oracleData.token1Decimal)
        );
    }
```

# Title 3: Fallback oracle is using spot price in Uniswap liquidity pool, which is very vulnerable to flashloan price manipulation

## Severity

Medium

## Impact

Fallback oracle is using spot price in Uniswap liquidity pool, which is very vulnerable to flashloan price manipulation. Hacker can use flashloan to distort the price and overborrow or perform malicious liqudiation.

## Proof of Concept

In the current implementation of the paraspace oracle, if the paraspace oracle has issue, the fallback oracle is used for ERC20 token.

```solidity
/// @inheritdoc IPriceOracleGetter
function getAssetPrice(address asset)
	public
	view
	override
	returns (uint256)
{
	if (asset == BASE_CURRENCY) {
		return BASE_CURRENCY_UNIT;
	}
	uint256 price = 0;
	IEACAggregatorProxy source = IEACAggregatorProxy(assetsSources[asset]);
	if (address(source) != address(0)) {
		price = uint256(source.latestAnswer());
	}
	if (price == 0 && address(_fallbackOracle) != address(0)) {
		price = _fallbackOracle.getAssetPrice(asset);
	}
	require(price != 0, Errors.ORACLE_PRICE_NOT_READY);
	return price;
}
```

which calls:

```solidity
	price = _fallbackOracle.getAssetPrice(asset);
```
which use the spot price from Uniswap V2.

```solidity
	address pairAddress = IUniswapV2Factory(UNISWAP_FACTORY).getPair(
		WETH,
		asset
	);
	require(pairAddress != address(0x00), "pair not found");
	IUniswapV2Pair pair = IUniswapV2Pair(pairAddress);
	(uint256 left, uint256 right, ) = pair.getReserves();
	(uint256 tokenReserves, uint256 ethReserves) = (asset < WETH)
		? (left, right)
		: (right, left);
	uint8 decimals = ERC20(asset).decimals();
	//returns price in 18 decimals
	return
		IUniswapV2Router01(UNISWAP_ROUTER).getAmountOut(
			10**decimals,
			tokenReserves,
			ethReserves
		);
```
and
```solidity
function getEthUsdPrice() public view returns (uint256) {
	address pairAddress = IUniswapV2Factory(UNISWAP_FACTORY).getPair(
		USDC,
		WETH
	);
	require(pairAddress != address(0x00), "pair not found");
	IUniswapV2Pair pair = IUniswapV2Pair(pairAddress);
	(uint256 left, uint256 right, ) = pair.getReserves();
	(uint256 usdcReserves, uint256 ethReserves) = (USDC < WETH)
		? (left, right)
		: (right, left);
	uint8 ethDecimals = ERC20(WETH).decimals();
	//uint8 usdcDecimals = ERC20(USDC).decimals();
	//returns price in 6 decimals
	return
		IUniswapV2Router01(UNISWAP_ROUTER).getAmountOut(
			10**ethDecimals,
			ethReserves,
			usdcReserves
		);
}
```
Using flashloan to distort and manipulate the price is very damaging technique.

Consider the POC below.

1. the User uses 10000 amount of tokenA as collateral, each token A worth 1 USD according to the paraspace oracle. the user borrow 3 ETH, the price of ETH is 1200 USD.
2. the paraspace oracle went down, the fallback price oracle is used, the user use borrows flashloan to distort the price of the tokenA in Uniswap pool from 1 USD to 10000 USD.
3. the user’s collateral position worth 1000 token X 10000 USD, and borrow 1000 ETH.
4. User repay the flashloan using the overborrowed amount and recover the price of the tokenA in Uniswap liqudity pool to 1 USD, leaving bad debt and insolvent position in Paraspace.

## Recommended Mitigation Steps

We recommend the project does not use the spot price in Uniswap V2, if the paraspace is down, it is safe to just revert the transaction.



