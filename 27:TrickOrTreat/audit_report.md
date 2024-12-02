# Trick Or Treat Audit
- Scope
    - TrickOrTreat.sol
- Findings
    - High: 0  
    [H-01: ](#high-01)
    - Medium: 3  
    [M-01: `setTreatCost()` will always revert if a Treat was added with 0 cost](#med-01)  
    [M-02: `transfer()` is deprecated and should not be used to transfer Ether](#med-02)  
    [M-03: `random` variable in `trickOrTreat()` uses block.timestamp and block.prevrandao, which are not good source of randomness](#med-03)  
    
- Tools
		- [Foundry](https://github.com/foundry-rs/foundry)

# Findings

## <a id="med-01"></a>M-01: `setTreatCost()` will always revert if a Treat was added with 0 cost

### Summary
In `addTreat()`, a new Treat can be added without any checks and in the event where a treat was added with 0 cost (_rate), the cost cannot be updated anymore in `setTreatCost()`.

### Vulnerability Details
`setTreatCost()` will revert when the cost of the treat is 0 and there is no way to update the cost.
```solidity
function setTreatCost(string memory _treatName, uint256 _cost) public onlyOwner {
->    require(treatList[_treatName].cost > 0, "Treat must cost something."); 
    treatList[_treatName].cost = _cost;
}
```

### Impact/Proof of Concept
```
function testCannotSetTreatCostIfZeroCost() public {
    spookySwap.addTreat("Zero Cost", 0, "ipfs://zero-cost-metadata");

    vm.expectRevert("Treat must cost something.");
    spookySwap.setTreatCost("Zero Cost", 1 ether);
}
```

Results
```
[PASS] testCannotSetTreatCostIfZeroCost() (gas: 91840)
Traces:
  [91840] TrickOrTreatTest::testCannotSetTreatCostIfZeroCost()
    ├─ [81652] SpookySwap::addTreat("Zero Cost", 0, "ipfs://zero-cost-metadata")
    │   ├─ emit TreatAdded(name: "Zero Cost", cost: 0, metadataURI: "ipfs://zero-cost-metadata")
    │   └─ ← [Stop] 
    ├─ [0] VM::expectRevert(Treat must cost something.)
    │   └─ ← [Return] 
    ├─ [1393] SpookySwap::setTreatCost("Zero Cost", 1000000000000000000 [1e18])
    │   └─ ← [Revert] revert: Treat must cost something.
    └─ ← [Stop] 
```

### Recommendations
Instead of ensuring the cost of the current Treat is bigger than 0, ensure that the new `_cost` should be bigger than 0.
```diff
function setTreatCost(string memory _treatName, uint256 _cost) public onlyOwner {
-    require(treatList[_treatName].cost > 0, "Treat must cost something."); 
+    require(_cost > 0, "Treat must cost something.");
    treatList[_treatName].cost = _cost;
}
```

## <a id="med-02"></a>M-02: `transfer()` is deprecated and should not be used to transfer Ether

### Summary
In `withdrawFees()`, transfer() is used to send ether to the owner. This is not recommended as transfer() sends a fixed gas of 2300 and is deprecated, which may not be sufficient in the future if the EVM gas costs changes. This will cause the funds in the contract to be stucked.

### Vulnerability Details
```solidity
function withdrawFees() public onlyOwner {
        uint256 balance = address(this).balance;
->        payable(owner()).transfer(balance);
        emit FeeWithdrawn(owner(), balance);
    }
```

### Impact/Proof of Concept
```

```

### Recommendations
transfer() has been deprecated and call() should be used instead.
```diff
function withdrawFees() public onlyOwner {
        uint256 balance = address(this).balance;
-        payable(owner()).transfer(balance);
+        (bool success,) = payable(owner()).call{value: balance}("");
+        require(success);
        emit FeeWithdrawn(owner(), balance);
    }
```

## <a id="med-03"></a>M-03: `random` variable in `trickOrTreat()` uses block.timestamp and block.prevrandao, which are not good source of randomness

### Summary
The `random` variable in `trickOrTreat()` is not truly random as the various parameters such as block.timestamp and block.prevrandao not good source of randomness and can be manipulated by miner/validators. Where miner/validators can allow specific address address to buy NFT at half price.

### Vulnerability Details
The `random` variable in `trickOrTreat()` is not truly random as the various parameters are predictable or fixed:
1. block.timestamp: The timestamp is predictable
2. msg.sender: This value is fixed
3. nextTokenId: The value of this variable is fixed for a period of time until the next NFT mint happens
4. block.prevrandao: The value of prevrandao will remain the same until a new block is created in the blockchain
```solidity
function trickOrTreat(string memory _treatName) public payable nonReentrant {
        ...
        // Generate a pseudo-random number between 1 and 1000
->        uint256 random = uint256(keccak256(abi.encodePacked(block.timestamp, msg.sender, nextTokenId, block.prevrandao))) % 1000 + 1;
        ...
}
```

### Impact/Proof of Concept
Miner/validators can manipulate the block.prevrandao and block.timestamp to allow specific address to buy NFT at half price.

### Recommendations
Change to other randomness methods, such as using Chainlink VRF.
