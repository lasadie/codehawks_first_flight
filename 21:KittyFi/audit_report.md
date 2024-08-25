# KittyFi Audit
- Scope
    - src/KittyCoin.sol
    - src/KittyPool.sol
    - src/KittyVault.sol
- Findings
    - High: 1  
    [H-01: Lack of access control in `burnKittyCoin()` function](#high-01)  
    - Medium: 1  
    [M-01: `meownufactureKittyVault()` function missing vault limit (20) constraint check](#med-01)
    - Low: 1  
    [L-01: Various core functions missing emit event](#Low-01)
    - Gas: 1  
    [G-01: `whiskdrawMeowllateral()` transfer before checking, resulting in gas wastage when revert](#gas-01)
     
- Tools
    - [Foundry](https://github.com/foundry-rs/foundry)

# Findings

## <a id="high-01"></a>H-01: Lack of access control in `burnKittyCoin()` function

### Summary
The `burnKittyCoin()` function in the KittyPool contract lacks proper access control and allows anyone to burn KittyCoin on behalf of any user. This can result in inaccurate collateral management issues, which can cause disruptions to the protocol's operations.

### Vulnerability Details
The function allows User A to burn the tokens of User B without authorization.
```
function burnKittyCoin(address _onBehalfOf, uint256 _ameownt) external {
        kittyCoinMeownted[_onBehalfOf] -= _ameownt;
        i_kittyCoin.burn(msg.sender, _ameownt);
    }
```

### Impact/Proof of Concept
In this PoC, we can use doggo to burn the tokens of catto without it's authorization.
```
function test_AnyoneCanBurnKittyCoin() public {
        address doggo = address(888);
        address catto = address(777);
        uint256 toDeposit = 5 ether;

        // Deal 10 ether to Doggo and Catto then deposit collaterals and mint tokens
        deal(weth, catto, 10 ether);
        deal(weth, doggo, 10 ether);
        vm.startPrank(catto);
        IERC20(weth).approve(address(wethVault), toDeposit);
        kittyPool.depawsitMeowllateral(weth, toDeposit);
        kittyPool.meowintKittyCoin(20e18);
        vm.stopPrank();

        vm.startPrank(doggo);
        IERC20(weth).approve(address(wethVault), toDeposit);
        kittyPool.depawsitMeowllateral(weth, toDeposit);
        kittyPool.meowintKittyCoin(20e18);
        vm.stopPrank();

        // Doggo burns the 20 tokens from Catto
        // Print Catto's before and after token balance
        console.log("before tokens: ",kittyPool.getKittyCoinMeownted(address(catto)));
        vm.prank(doggo);
        kittyPool.burnKittyCoin(catto, 20e18);
        console.log("after tokens: ",kittyPool.getKittyCoinMeownted(address(catto)));
    }
```
Result
```
Ran 1 test for test/KittyFiTest.t.sol:KittyFiTest
[PASS] test_AnyoneCanBurnKittyCoin() (gas: 682149)
Logs:
  before tokens:  20000000000000000000
  after tokens:  0
```

### Recommendations
1. Add a logic to check if the msg.sender is the user or the maintainer of the protocol.
```diff
function burnKittyCoin(address _onBehalfOf, uint256 _ameownt) external {
+       require(msg.sender == _onBehalfOf || msg.sender == meowntainer, "Not authorized to burn tokens");
        kittyCoinMeownted[_onBehalfOf] -= _ameownt;
        i_kittyCoin.burn(msg.sender, _ameownt);
    }
```
2. Implement Approval from ERC20 to authorize another user to burn.


## <a id="med-01"></a>M-01: `meownufactureKittyVault()` function missing vault limit (20) constraint check

### Summary
The `meownufactureKittyVault()` function in the KittyPool contract does not check if the vault length has exceeded the limit of 20, which is stated in the contract assumptions.

### Vulnerability Details
This function does not check if the vault length has exceeded the limit of 20.
```
function meownufactureKittyVault(address _token, address _priceFeed) external onlyMeowntainer {
        require(tokenToVault[_token] == address(0), KittyPool__TokenAlreadyExistsMeeoooww());

        address _kittyVault = address(new KittyVault{ salt: bytes32(abi.encodePacked(ERC20(_token).symbol())) }(_token, address(this), _priceFeed, i_euroPriceFeed, meowntainer, i_aavePool));

        tokenToVault[_token] = _kittyVault;
        vaults.push(_kittyVault);
    }
```

### Impact/Proof of Concept
In this PoC, we simply add more than 20 tokens types into the vault
```
// Add this to KittyPool.sol to get the lenght of tokenToVault
function vaultsLength() public view returns (uint256) {
        return vaults.length;
    }

function test_VaultExceeds20Length() public {
        // Mock ERC20 tokens
        for (uint256 i = 0; i < 21; i++) {
            vm.startPrank(meowntainer);
            ERC20Mock token = new ERC20Mock();
            MockV3Aggregator priceFeed = new MockV3Aggregator(8, 1e8);

            // Call meownufactureKittyVault to add the token
            kittyPool.meownufactureKittyVault(address(token), address(priceFeed));
        }
        
        uint256 length = kittyPool.vaultsLength();
        console.log("length: ", length);
    }
```
Result
```
Ran 1 test for test/KittyFiTest.t.sol:KittyFiTest
[PASS] test_VaultExceeds20Length() (gas: 36596661)
Logs:
  length:  22

Traces:
  ........

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 54.03s (46.38s CPU time)

Ran 1 test suite in 55.47s (54.03s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### Recommendations
Add a constraint check to ensure that the no more than 20 collateral tokens will be used in the protocol.
```diff
function meownufactureKittyVault(address _token, address _priceFeed) external onlyMeowntainer {
        require(tokenToVault[_token] == address(0), KittyPool__TokenAlreadyExistsMeeoooww());
+       require(vaults.length <= 20, "Maximum number of vaults reached");
        address _kittyVault = address(new KittyVault{ salt: bytes32(abi.encodePacked(ERC20(_token).symbol())) }(_token, address(this), _priceFeed, i_euroPriceFeed, meowntainer, i_aavePool));

        tokenToVault[_token] = _kittyVault;
        vaults.push(_kittyVault);
    }
```

## <a id="low-02"></a>L-01: Various core functions missing emit event

### Summary
Various core functions such as `depawsitMeowllateral()`, `whiskdrawMeowllateral()`, `meowintKittyCoin()`, `burnKittyCoin()`, `purrgeBadPawsition()` and `meownufactureKittyVault()` is missing emit event. This does not provide enough transparency and there will be difficulties during security debugging and auditing.

### Vulnerability Details
These functions are missing emit event
```
function depawsitMeowllateral(address _token, uint256 _ameownt) external tokenExists(_token) {
    IKittyVault(tokenToVault[_token]).executeDepawsit(msg.sender, _ameownt);
} 

function whiskdrawMeowllateral(address _token, uint256 _ameownt) external tokenExists(_token) {
    IKittyVault(tokenToVault[_token]).executeWhiskdrawal(msg.sender, _ameownt);
    require(_hasEnoughMeowllateral(msg.sender), KittyPool__NotEnoughMeowllateralPurrrr());
}

function meowintKittyCoin(uint256 _ameownt) external {
    kittyCoinMeownted[msg.sender] += _ameownt;
    i_kittyCoin.mint(msg.sender, _ameownt);
    require(_hasEnoughMeowllateral(msg.sender), KittyPool__NotEnoughMeowllateralPurrrr());
}

function burnKittyCoin(address _onBehalfOf, uint256 _ameownt) external {
    kittyCoinMeownted[_onBehalfOf] -= _ameownt;
    i_kittyCoin.burn(msg.sender, _ameownt);
}

function purrgeBadPawsition(address _user) external returns (uint256 _totalAmountReceived) {
    require(!(_hasEnoughMeowllateral(_user)), KittyPool__UserIsPurrfect());
    uint256 totalDebt = kittyCoinMeownted[_user];

    kittyCoinMeownted[_user] = 0;
    i_kittyCoin.burn(msg.sender, totalDebt);

    uint256 userMeowllateralInEuros = getUserMeowllateralInEuros(_user);

    uint256 redeemPercent;

    if (totalDebt >= userMeowllateralInEuros) {
        redeemPercent = PRECISION;
    }
    else {
        redeemPercent = totalDebt.mulDiv(PRECISION, userMeowllateralInEuros);
    }

    uint256 vaults_length = vaults.length;

    for (uint256 i; i < vaults_length; ) {
        IKittyVault _vault = IKittyVault(vaults[i]);
        uint256 vaultCollateral = _vault.getUserVaultMeowllateralInEuros(_user);
        uint256 toDistribute = vaultCollateral.mulDiv(redeemPercent, PRECISION);
        uint256 extraCollateral = vaultCollateral - toDistribute;

        uint256 extraReward = toDistribute.mulDiv(REWARD_PERCENT, PRECISION);
        extraReward = Math.min(extraReward, extraCollateral);
        _totalAmountReceived += (toDistribute + extraReward);

        _vault.executeWhiskdrawal(msg.sender, toDistribute + extraReward);

        unchecked {
            ++i;
        }
    }
}

function meownufactureKittyVault(address _token, address _priceFeed) external onlyMeowntainer {
    require(tokenToVault[_token] == address(0), KittyPool__TokenAlreadyExistsMeeoooww());

    address _kittyVault = address(new KittyVault{ salt: bytes32(abi.encodePacked(ERC20(_token).symbol())) }(_token, address(this), _priceFeed, i_euroPriceFeed, meowntainer, i_aavePool));

    tokenToVault[_token] = _kittyVault;
    vaults.push(_kittyVault);
} 
```

### Impact/Proof of Concept
The absence of events in key functions of the KittyPool contract undermines the ability to track and audit critical operations, such as minting, burning, depositing, and withdrawing collateral. This lack of visibility can hinder effective monitoring, make debugging challenging, and reduce transparency, potentially leading to difficulties in verifying contract activity and ensuring accountability.


### Recommendations
Define the events and emit those events when the core functions are performed.
```diff
// Define events
+ event MeowllateralDeposited(address indexed user, address indexed token, uint256 amount);
+ event MeowllateralWithdrawn(address indexed user, address indexed token, uint256 amount);
+ event KittyCoinMinted(address indexed user, uint256 amount);
+ event KittyCoinBurned(address indexed user, uint256 amount);
+ event BadDebtLiquidated(address indexed user, uint256 totalAmountReceived);
+ event VaultCreated(address indexed token, address indexed vault);

// Emit events in the relevant functions
function depawsitMeowllateral(address _token, uint256 _ameownt) external tokenExists(_token) {
    // existing code...
+    emit MeowllateralDeposited(msg.sender, _token, _ameownt);
}

function whiskdrawMeowllateral(address _token, uint256 _ameownt) external tokenExists(_token) {
    // existing code...
+   emit MeowllateralWithdrawn(msg.sender, _token, _ameownt);
}

function meowintKittyCoin(uint256 _ameownt) external {
    // existing code...
+    emit KittyCoinMinted(msg.sender, _ameownt);
}

function burnKittyCoin(address _onBehalfOf, uint256 _ameownt) external {
    // existing code...
+    emit KittyCoinBurned(_onBehalfOf, _ameownt);
}

function purrgeBadPawsition(address _user) external returns (uint256 _totalAmountReceived) {
    // existing codes...
+    emit BadDebtLiquidated(_user, _totalAmountReceived);
}

function meownufactureKittyVault(address _token, address _priceFeed) external onlyMeowntainer {
    // existing codes...
+    emit VaultCreated(_token, _kittyVault);
}
```

## <a id="gas-01"></a>G-01: `whiskdrawMeowllateral()` transfer before checking, resulting in gas wastage when revert

### Summary
The `whiskdrawMeowllateral()` function performs the transfer of tokens first before checking if there is enough collateral. In the event if there is insufficient colleteral, the transaction will revert and causing the gas for transferring the tokens to be wasted.

### Vulnerability Details
The function transfer tokens first before checking if there is enough collateral.
```
function whiskdrawMeowllateral(address _token, uint256 _ameownt) external tokenExists(_token) {
        IKittyVault(tokenToVault[_token]).executeWhiskdrawal(msg.sender, _ameownt);
        require(_hasEnoughMeowllateral(msg.sender), KittyPool__NotEnoughMeowllateralPurrrr());
    }
```

### Impact/Proof of Concept
This PoC will simulate a user depositing collateral and minting, thereafter to withdraw when there is not enough collateral and resulting in revert. This results in wasting **3297** gas.
```
function test_WhiskdrawMeowllateralRevertWasteGas() public {
        address doggo = address(888);
        uint256 initialDeposit = 5 ether;

        // Setup: Fund doggo and deposit collateral
        deal(weth, doggo, 10 ether);
        vm.startPrank(doggo);
        IERC20(weth).approve(address(wethVault), initialDeposit);
        kittyPool.depawsitMeowllateral(weth, initialDeposit);
        kittyPool.meowintKittyCoin(20e18);
        vm.stopPrank();

        // Attempt to withdraw and expecting a revert
        vm.startPrank(doggo);
        uint256 withdrawAmount = 5 ether;
        try kittyPool.whiskdrawMeowllateral(weth, withdrawAmount) {
            console.log("Withdrawal succeeded unexpectedly");
        } catch {
            console.log("Withdrawal reverted due to insufficient collateral");
        }
        vm.stopPrank();
    }
```
Result
```diff
Ran 1 test for test/KittyFiTest.t.sol:KittyFiTest
[PASS] test_WhiskdrawMeowllateralRevertWasteGas() (gas: 454421)
Logs:
  Withdrawal reverted due to insufficient collateral

Traces:
  [454421] KittyFiTest::test_WhiskdrawMeowllateralRevertWasteGas()
  ...
  ...
  ...
  [0] VM::startPrank(0x0000000000000000000000000000000000000378)
    │   └─ ← [Return] 
    ├─ [36516] KittyPool::whiskdrawMeowllateral(0xC558DBdd856501FCd9aaF1E62eae57A9F0629a3c, 5000000000000000000 [5e18])
    │   ├─ [15619] KittyVault::executeWhiskdrawal(0x0000000000000000000000000000000000000378, 5000000000000000000 [5e18])
    │   │   ├─ [4859] 0x6Ae43d3271ff6888e7Fc43Fd7321a503ff738951::getUserAccountData(KittyVault: [0x4a89a6cFA7bB08AeAee2B1A63cDA0CdD5720Bc91]) [staticcall]
    │   │   │   ├─ [4283] 0x0562453c3DAFBB5e625483af58f4E6D668c44e19::getUserAccountData(KittyVault: [0x4a89a6cFA7bB08AeAee2B1A63cDA0CdD5720Bc91]) [delegatecall]
    │   │   │   │   ├─ [402] 0x012bAC54348C0E635dCAc9D5FB99f06F24136C9A::getPriceOracle() [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x0000000000000000000000002da88497588bf89281816106c7259e31af45a663
    │   │   │   │   ├─ [1521] 0x1ce1bA9946C30b4C505631AD9E3E0342877FdE02::26ec273f(000000000000000000000000000000000000000000000000000000000000003400000000000000000000000000000000000000000000000000000000000000360000000000000000000000000000000000000000000000000000000000000037000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000090000000000000000000000004a89a6cfa7bb08aeaee2b1a63cda0cdd5720bc910000000000000000000000002da88497588bf89281816106c7259e31af45a6630000000000000000000000000000000000000000000000000000000000000000) [delegatecall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
    │   │   │   │   └─ ← [Return] 0, 0, 0, 0, 0, 115792089237316195423570985008687907853269984665640564039457584007913129639935 [1.157e77]
    │   │   │   └─ ← [Return] 0, 0, 0, 0, 0, 115792089237316195423570985008687907853269984665640564039457584007913129639935 [1.157e77]
    │   │   ├─ [3543] 0x694AA1769357215DE4FAC081bf1f309aDC325306::latestRoundData() [staticcall]
    │   │   │   ├─ [1612] 0x719E22E3D4b690E5d96cCb40619180B5427F14AE::latestRoundData() [staticcall]
    │   │   │   │   └─ ← [Return] 14925 [1.492e4], 251894720686 [2.518e11], 1722957240 [1.722e9], 1722957240 [1.722e9], 14925 [1.492e4]
    │   │   │   └─ ← [Return] 18446744073709566541 [1.844e19], 251894720686 [2.518e11], 1722957240 [1.722e9], 1722957240 [1.722e9], 18446744073709566541 [1.844e19]
+     │   │   ├─ [3297] 0xC558DBdd856501FCd9aaF1E62eae57A9F0629a3c::transfer(0x0000000000000000000000000000000000000378, 5000000000000000000 [5e18])
    │   │   │   ├─ emit Transfer(from: KittyVault: [0x4a89a6cFA7bB08AeAee2B1A63cDA0CdD5720Bc91], to: 0x0000000000000000000000000000000000000378, value: 5000000000000000000 [5e18])
    │   │   │   └─ ← [Return] true
    │   │   └─ ← [Return] 
    │   ├─ [18901] KittyVault::getUserVaultMeowllateralInEuros(0x0000000000000000000000000000000000000378) [staticcall]
    │   │   ├─ [3543] 0x694AA1769357215DE4FAC081bf1f309aDC325306::latestRoundData() [staticcall]
    │   │   │   ├─ [1612] 0x719E22E3D4b690E5d96cCb40619180B5427F14AE::latestRoundData() [staticcall]
    │   │   │   │   └─ ← [Return] 14925 [1.492e4], 251894720686 [2.518e11], 1722957240 [1.722e9], 1722957240 [1.722e9], 14925 [1.492e4]
    │   │   │   └─ ← [Return] 18446744073709566541 [1.844e19], 251894720686 [2.518e11], 1722957240 [1.722e9], 1722957240 [1.722e9], 18446744073709566541 [1.844e19]
    │   │   ├─ [3543] 0x1a81afB8146aeFfCFc5E50e8479e826E7D55b910::latestRoundData() [staticcall]
    │   │   │   ├─ [1612] 0xD404D68e5616e9C7045be2Dc1C5865EE328B6638::latestRoundData() [staticcall]
    │   │   │   │   └─ ← [Return] 1666, 109265000 [1.092e8], 1722931836 [1.722e9], 1722931836 [1.722e9], 1666
    │   │   │   └─ ← [Return] 18446744073709553282 [1.844e19], 109265000 [1.092e8], 1722931836 [1.722e9], 1722931836 [1.722e9], 18446744073709553282 [1.844e19]
    │   │   ├─ [4859] 0x6Ae43d3271ff6888e7Fc43Fd7321a503ff738951::getUserAccountData(KittyVault: [0x4a89a6cFA7bB08AeAee2B1A63cDA0CdD5720Bc91]) [staticcall]
    │   │   │   ├─ [4283] 0x0562453c3DAFBB5e625483af58f4E6D668c44e19::getUserAccountData(KittyVault: [0x4a89a6cFA7bB08AeAee2B1A63cDA0CdD5720Bc91]) [delegatecall]
    │   │   │   │   ├─ [402] 0x012bAC54348C0E635dCAc9D5FB99f06F24136C9A::getPriceOracle() [staticcall]
    │   │   │   │   │   └─ ← [Return] 0x0000000000000000000000002da88497588bf89281816106c7259e31af45a663
    │   │   │   │   ├─ [1521] 0x1ce1bA9946C30b4C505631AD9E3E0342877FdE02::26ec273f(000000000000000000000000000000000000000000000000000000000000003400000000000000000000000000000000000000000000000000000000000000360000000000000000000000000000000000000000000000000000000000000037000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000090000000000000000000000004a89a6cfa7bb08aeaee2b1a63cda0cdd5720bc910000000000000000000000002da88497588bf89281816106c7259e31af45a6630000000000000000000000000000000000000000000000000000000000000000) [delegatecall]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
    │   │   │   │   └─ ← [Return] 0, 0, 0, 0, 0, 115792089237316195423570985008687907853269984665640564039457584007913129639935 [1.157e77]
    │   │   │   └─ ← [Return] 0, 0, 0, 0, 0, 115792089237316195423570985008687907853269984665640564039457584007913129639935 [1.157e77]
    │   │   ├─ [3543] 0x694AA1769357215DE4FAC081bf1f309aDC325306::latestRoundData() [staticcall]
    │   │   │   ├─ [1612] 0x719E22E3D4b690E5d96cCb40619180B5427F14AE::latestRoundData() [staticcall]
    │   │   │   │   └─ ← [Return] 14925 [1.492e4], 251894720686 [2.518e11], 1722957240 [1.722e9], 1722957240 [1.722e9], 14925 [1.492e4]
    │   │   │   └─ ← [Return] 18446744073709566541 [1.844e19], 251894720686 [2.518e11], 1722957240 [1.722e9], 1722957240 [1.722e9], 18446744073709566541 [1.844e19]
    │   │   └─ ← [Revert] panic: division or modulo by zero (0x12)
    │   └─ ← [Revert] panic: division or modulo by zero (0x12)
    ├─ [0] console::log("Withdrawal reverted due to insufficient collateral") [staticcall]
    │   └─ ← [Stop] 
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    └─ ← [Return] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 19.58s (14.01s CPU time)

Ran 1 test suite in 21.79s (19.58s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### Recommendations
Perform the check first to ensure there is sufficient collateral before withdrawing.
```diff
function whiskdrawMeowllateral(address _token, uint256 _ameownt) external tokenExists(_token) {
    require(_hasEnoughMeowllateral(msg.sender), KittyPool__NotEnoughMeowllateralPurrrr());
    IKittyVault(tokenToVault[_token]).executeWhiskdrawal(msg.sender, _ameownt);
    
}
```