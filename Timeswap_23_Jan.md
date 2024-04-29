# Title 1: The `collect()` function will always TRANSFER ZERO fees, losing `_feesPositions` without receiving fees!

## Severity

High

## Impact

The `collect()` function will always transfer ZERO fees. At the same time, non-zero `_feesPosition` will be burned.
```solidity
	_feesPositions[id][msg.sender].burn(long0Fees, long1Fees, shortFees);
```
As a result, the contracts will be left in an inconsistent state. The user will burn `_feesPositions` without receiving the fees!

## Proof of Concept

Provide direct links to all referenced code in GitHub. Add screenshots, logs, or any other relevant proof that illustrates the concept.

The `collect()` function will always transfer ZERO fees in the following line:
```solidity
	// transfer the fees amount to the recipient
    ITimeswapV2Pool(poolPair).transferFees(param.strike, param.maturity, param.to, long0Fees, long1Fees, shortFees);
```
This is because, at this moment, the values of `long0Fees`, `long1Fees`, `shortFees` have not been calculated yet, actually, they will be equal to zero. Therefore, no fees will be transferred. The values of `long0Fees`, `long1Fees`, `shortFees` are calculated afterwards by the following line:
```solidity
	(long0Fees, long1Fees, shortFees) = _feesPositions[id][msg.sender].getFees(param.long0FeesDesired, param.long1FeesDesired, param.shortFeesDesired);
```
Therefore, `ITimeswapV2Pool(poolPair).transferFees` must be called after this line to be correct.

## Tools Used

Remix

## Recommended Mitigation Steps

We moved the line `ITimeswapV2Pool(poolPair).transferFees` after `long0Fees`, `long1Fees`, `shortFees` have been calculated first.
```solidity
	function collect(TimeswapV2LiquidityTokenCollectParam calldata param) external returns (uint256 long0Fees, uint256 long1Fees, uint256 shortFees, bytes memory data) {
        ParamLibrary.check(param);
        bytes32 key = TimeswapV2LiquidityTokenPosition({token0: param.token0, token1: param.token1, strike: param.strike, maturity: param.maturity}).toKey();
        // start the reentrancy guard
        raiseGuard(key);
        (, address poolPair) = PoolFactoryLibrary.getWithCheck(optionFactory, poolFactory, param.token0, param.token1);
        uint256 id = _timeswapV2LiquidityTokenPositionIds[key];
        _updateFeesPositions(msg.sender, address(0), id);
        (long0Fees, long1Fees, shortFees) = _feesPositions[id][msg.sender].getFees(param.long0FeesDesired, param.long1FeesDesired, param.shortFeesDesired);
        if (param.data.length != 0)
            data = ITimeswapV2LiquidityTokenCollectCallback(msg.sender).timeswapV2LiquidityTokenCollectCallback(
                TimeswapV2LiquidityTokenCollectCallbackParam({
                    token0: param.token0,
                    token1: param.token1,
                    strike: param.strike,
                    maturity: param.maturity,
                    long0Fees: long0Fees,
                    long1Fees: long1Fees,
                    shortFees: shortFees,
                    data: param.data
                })
            );
                // transfer the fees amount to the recipient
        ITimeswapV2Pool(poolPair).transferFees(param.strike, param.maturity, param.to, long0Fees, long1Fees, shortFees);
        // burn the desired fees from the fees position
        _feesPositions[id][msg.sender].burn(long0Fees, long1Fees, shortFees);
        if (long0Fees != 0 || long1Fees != 0 || shortFees != 0) _removeTokenEnumeration(msg.sender, address(0), id, 0);
        // stop the reentrancy guard
        lowerGuard(key);
    }
```

# Title 2: Burning a `ERC1155Enumerable` token doesn’t remove it from the enumeration

## Severity

Medium

## Summary

The `ERC1155Enumerable` base contract used in the `TimeswapV2Token` and `TimeswapV2LiquidityToken` tokens provides a functionality to enumerate all token ids that have been minted in the contract.

The logic to remove the token from the enumeration if the last token is burned is implemented in the `_afterTokenTransfer` hook:
```solidity
function _afterTokenTransfer(address, address from, address to, uint256[] memory ids, uint256[] memory amounts, bytes memory) internal virtual override {
    for (uint256 i; i < ids.length; ) {
        if (amounts[i] != 0) _removeTokenEnumeration(from, to, ids[i], amounts[i]);
        unchecked {
            ++i;
        }
    }
}
/// @dev Remove token enumeration list if necessary.
function _removeTokenEnumeration(address from, address to, uint256 id, uint256 amount) internal {
    if (to == address(0)) {
        if (_idTotalSupply[id] == 0 && _additionalConditionRemoveTokenFromAllTokensEnumeration(id)) _removeTokenFromAllTokensEnumeration(id);
        _idTotalSupply[id] -= amount;
    }
    if (from != address(0) && from != to) {
        if (balanceOf(from, id) == 0 && _additionalConditionRemoveTokenFromOwnerEnumeration(from, id)) _removeTokenFromOwnerEnumeration(from, id);
    }
}
```
The `_removeTokenEnumeration` condition to check if the supply is 0 happens before the function decreases the burned amount. This will `_removeTokenFromAllTokensEnumeration` from being called when the last token(s) is(are) burned.

## Impact

The token isn’t removed from the enumeration since `_removeTokenFromAllTokensEnumeration` will never be called. This will cause the enumeration to always contain a minted token even though it is burned afterwards. The function `totalSupply` and `tokenByIndex` will report wrong values.

This will also cause the enumeration to contain duplicate values or multiple copies of the same token. If the token is minted again after all tokens were previously burned, the token will be re added to the enumeration.

## Proof of Concept

The following test demonstrates the issue. Alice is minted a token and that token is then burned, the token is still present in the enumeration. The token is minted again, causing the enumeration to contain the token by duplicate.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity =0.8.8;
import "forge-std/Test.sol";
import "../src/base/ERC1155Enumerable.sol";
contract TestERC1155Enumerable is ERC1155Enumerable {
    constructor() ERC1155("") {
    }
    function mint(address to, uint256 id, uint256 amount) external {
        _mint(to, id, amount, "");
    }
    function burn(address from, uint256 id, uint256 amount) external {
        _burn(from, id, amount);
    }
}
contract AuditTest is Test {
    function test_ERC1155Enumerable_BadRemoveFromEnumeration() public {
        TestERC1155Enumerable token = new TestERC1155Enumerable();
        address alice = makeAddr("alice");
        uint256 tokenId = 0;
        uint256 amount = 1;
        token.mint(alice, tokenId, amount);
        // tokenByIndex and totalSupply are ok
        assertEq(token.tokenByIndex(0), tokenId);
        assertEq(token.totalSupply(), 1);
        // now we burn the token
        token.burn(alice, tokenId, amount);
        // tokenByIndex and totalSupply still report previous values
        // tokenByIndex should throw index out of bounds, and supply should return 0
        assertEq(token.tokenByIndex(0), tokenId);
        assertEq(token.totalSupply(), 1);
        // Now we mint it again, this will re-add the token to the enumeration, duplicating it
        token.mint(alice, tokenId, amount);
        assertEq(token.totalSupply(), 2);
        assertEq(token.tokenByIndex(0), tokenId);
        assertEq(token.tokenByIndex(1), tokenId);
    }
}
```

## Recommendation

Decrease the amount before checking if the supply is 0.

```solidity
function _removeTokenEnumeration(address from, address to, uint256 id, uint256 amount) internal {
    if (to == address(0)) {
        _idTotalSupply[id] -= amount;
        if (_idTotalSupply[id] == 0 && _additionalConditionRemoveTokenFromAllTokensEnumeration(id)) _removeTokenFromAllTokensEnumeration(id);
    }
    if (from != address(0) && from != to) {
        if (balanceOf(from, id) == 0 && _additionalConditionRemoveTokenFromOwnerEnumeration(from, id)) _removeTokenFromOwnerEnumeration(from, id);
    }
}
```

