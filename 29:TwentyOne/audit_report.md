# TwentyOne Audit
- Scope
    - TwentyOne.sol  
- Findings
    - High: 2  
    (Valid) [H-01: `startGame()` card value randomness can be gamed](#high-01)  
    (Valid) [H-02: `dealersHand()` and `playersHand()` has inconsistent calculation of cardValue](#high-02)  
    - Medium: 1  
    (Valid) [M-01: `call()` logic determines the player to lose even when player and dealer has same cardValue (a tie)](#med-01)
    - Low: 1  
    (Invalid) [L-01: Excess player bets can get stuck](#low-01)  
          
- Tools
        - [Foundry](https://github.com/foundry-rs/foundry)

# Findings

## <a id="high-01"></a>H-01: `startGame()` card value randomness can be gamed

### Summary
The `startGame()` function reveals the player's cardValue in the same transaction of placing the bets. This allows users to revert the transaction as long as the returned cardValue is not 20, and only play if the cardValue is 20 which increases the odds of winning to more than 60%. Hence, the card randomness can be gamed. Following are the scenarios that a player will win or lose if he plays with card value of 20.

|Dealer Points | Win/Lose |
| ---   | -------- |
|17 | win |
|18 | win |
|19 | win |
|20 | lose |
|21 | lose |
|>21 | win |


### Vulnerability Details
This function receives the bet and return the player's card value in the same transaction.
```solidity
function startGame() public payable returns (uint256) {
        address player = msg.sender;
        require(msg.value >= 1 ether, "not enough ether sent"); 
        initializeDeck(player);
        uint256 card1 = drawCard(player);
        uint256 card2 = drawCard(player);
        addCardForPlayer(player, card1);
        addCardForPlayer(player, card2);
        return playersHand(player);
    }
```

### Impact/Proof of Concept
Revert transaction if returned cardValue is not 20.
```
function test_PlayerOnlyPlayif20() public {
        vm.startPrank(player1);

        uint256 cardValue = twentyOne.startGame{value: 1 ether}();
        require(cardValue == 20, "Card value not 20");
        twentyOne.call();
    }
```

Results
```diff
Revert if not 20
[1250994] TwentyOneTest::test_PlayerOnlyPlayif20()
    ├─ [0] VM::startPrank(0x0000000000000000000000000000000000000123)
    │   └─ ← [Return] 
    ├─ [1270359] TwentyOne::startGame{value: 1000000000000000000}()
    │   └─ ← [Return] 9
    ├─ [0] console::log("balance: ", 9000000000000000000 [9e18]) [staticcall]
    │   └─ ← [Stop] 
    └─ ← [Revert] revert: Card value not 20

Play if 20
[1377791] TwentyOneTest::test_PlayerOnlyPlayif20()
    ├─ [0] VM::startPrank(0x0000000000000000000000000000000000000123)
    │   └─ ← [Return] 
    ├─ [1270379] TwentyOne::startGame{value: 1000000000000000000}()
    │   └─ ← [Return] 20
    ├─ [0] console::log("balance: ", 9000000000000000000 [9e18]) [staticcall]
    │   └─ ← [Stop] 
    ├─ [126378] TwentyOne::call()
    │   ├─ emit PlayerWonTheGame(message: "Dealer went bust, players winning hand: ", cardsTotal: 20)
```

### Recommendations
Implement Commit-Reveal scheme such as locking in the bets first then reveal the card value to players.



## <a id="high-02"></a>H-02: `dealersHand()` and `playersHand()` has inconsistent calculation of cardValue

### Summary
In `dealersHand()`, if the dealer draw 13, 26, 39, or 52, it would add 0 as the cardValue to the dealer's hand. But for `playersHand()`, those 4 cards would add 10 to the cardValue. Hence, this means that the dealer has a 7.69% chance of adding 0 instead of 10 to its cardValue.


### Vulnerability Details
```
function playersHand(address player) public view returns (uint256) {
        uint256 playerTotal = 0;
        for (uint256 i = 0; i < playersDeck[player].playersCards.length; i++) {
            uint256 cardValue = playersDeck[player].playersCards[i] % 13;

->            if (cardValue == 0 || cardValue >= 10) {
                playerTotal += 10;
            } else {
                playerTotal += cardValue;
            }
        }
        return playerTotal;
    }

    function dealersHand(address player) public view returns (uint256) {
        uint256 dealerTotal = 0;
        for (uint256 i = 0; i < dealersDeck[player].dealersCards.length; i++) {
            uint256 cardValue = dealersDeck[player].dealersCards[i] % 13;
->            if (cardValue >= 10) { 
                dealerTotal += 10;
            } else {
                dealerTotal += cardValue;
            }
        }
        return dealerTotal;
    }
```

### Impact/Proof of Concept
This inconsistency is generally favourable for the dealer, as there is a 7.69% lower chance of getting 10 points, which can cause the dealer to go bust.

### Recommendations
Apply the same calculation logic as `playersHands()`.
```diff
function dealersHand(address player) public view returns (uint256) {
        uint256 dealerTotal = 0;
        for (uint256 i = 0; i < dealersDeck[player].dealersCards.length; i++) {
            uint256 cardValue = dealersDeck[player].dealersCards[i] % 13;
-            if (cardValue >= 10) { 
+            if (cardValue == 0 || cardValue >= 10) {
                dealerTotal += 10;
            } else {
                dealerTotal += cardValue;
            }
        }
        return dealerTotal;
    }
```


## <a id="med-01"></a>M-01: `call()` logic determines the player to lose even when player and dealer has same cardValue (a tie)

### Summary
In the scenario of the player and dealer having the same cardValue(a tie), the else statement will catch this condition and assume the player to have lost the game and the dealer's hand is higher. Which in actual fact is incorrect.


### Vulnerability Details
In the scenario of the player and dealer having the same cardValue(a tie), the else statement will catch this condition and determine the player of have lost the game and the dealer's hand is higher. Which in actual fact is incorrect.
```
function call() public {
        ..........

        // Determine the winner
        if (dealerHand > 21) {
            emit PlayerWonTheGame(
                "Dealer went bust, players winning hand: ",
                playerHand
            );
            endGame(msg.sender, true);
        } else if (playerHand > dealerHand) {
            emit PlayerWonTheGame(
                "Dealer's hand is lower, players winning hand: ",
                playerHand
            );
            endGame(msg.sender, true);
        } else { 
            emit PlayerLostTheGame(
                "Dealer's hand is higher, dealers winning hand: ",
                dealerHand
            );
            endGame(msg.sender, false);
        }
    }
```

### Impact/Proof of Concept
This causes unfairness to players and reduces the integrity of the protocol.

### Recommendations
Implement another logic check to handle the scenario of dealer and player having same cardValue
```diff
else if (playerHand == dealerHand) {
        implement action here...
    }
```


## <a id="low-01"></a>L-01: Excess player bets can get stuck

### Summary
If a player transfer more than 1 ether during `startGame()`, the excess ether can get stuck in the contract because when the player wins, they only receive back 2 ether.


### Vulnerability Details
In `startGame()`, this allows the user to transfer more than 1 ether even when the bet is 1 ether. And in `endGame()`, the player receives 2 ether when he wins.
```solidity
function startGame() public payable returns (uint256) {
        address player = msg.sender;
        require(msg.value >= 1 ether, "not enough ether sent");
        initializeDeck(player);
        uint256 card1 = drawCard(player);
        uint256 card2 = drawCard(player);
        addCardForPlayer(player, card1);
        addCardForPlayer(player, card2);
        return playersHand(player);
    }

function endGame(address player, bool playerWon) internal {
    delete playersDeck[player].playersCards; // Clear the player's cards
    delete dealersDeck[player].dealersCards; // Clear the dealer's cards
    delete availableCards[player]; // Reset the deck
    if (playerWon) {
        payable(player).transfer(2 ether); // Transfer the prize to the player
        emit FeeWithdrawn(player, 2 ether); // Emit the prize withdrawal event
    }
}

```

### Impact/Proof of Concept
In the scenario where the player transfers 3 ether to `startGame()` and won the game, he would only receive 2 ether back. And have 1 ether stuck in the contract.


### Recommendations
1. Implement a system to track how much player has transferred over and return the excess ether when a player wins or loses.
2. Check if msg.value == 1 ether instead
```diff
function startGame() public payable returns (uint256) {
        address player = msg.sender;
-        require(msg.value >= 1 ether, "not enough ether sent");
+        require(msg.value == 1 ether, "not enough ether sent");
        initializeDeck(player);
        uint256 card1 = drawCard(player);
        uint256 card2 = drawCard(player);
        addCardForPlayer(player, card1);
        addCardForPlayer(player, card2);
        return playersHand(player);
    }
```



