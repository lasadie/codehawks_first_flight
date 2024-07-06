
# Audit Details
- Scope
    - PasswordStore.sol
- Findings
    - High: 2

## H-01: Anyone can call `setPassword()` function and change the password

### Summary
The PasswordStore contract allows the owner to set the password. When the owner wants to update/change the password, it calls this function again to set it to the new password. However, access control is not in placed and anyone can update/change the password to a new one.

### Vulnerability Details
This function is missing an access control to check if the the msg.sender is `s_owner`. Hence, anyone can call this function and set new password.

*PasswordStore.sol line26*
```
function setPassword(string memory newPassword) external {
    s_password = newPassword;
    emit SetNetPassword();
}
```

### Impact/Proof of Concept
In this PoC, we first use the contract owner's address to set the password as "ownerPassword" and we call `getPassword()` to confirm that it is the same password. After which, we use another arbitrary address to call `setPassword()` to change it. Once it is done, we use the contract's owner address to call `getPassword()` again to show that the password has indeed changed.

```
function test_non_owner_can_set_password() public {
    // Use owner address to set password and then call it to confirm
    vm.startPrank(owner);
    passwordStore.setPassword("ownerPassword");
    passwordStore.getPassword();
    // Change to another arbitrary address and set password to new password
    vm.startPrank(address(1));
    passwordStore.setPassword("thisisnewpassword");
    // Switch back to owner address and call getPassword() to proof that the password has changed
    vm.startPrank(owner);
    passwordStore.getPassword();
}
```
Results:
```
Ran 1 test for test/PasswordStore.t.sol:PasswordStoreTest
[PASS] test_non_owner_set_password() (gas: 26700)
Traces:
  [26700] PasswordStoreTest::test_non_owner_set_password()
    ├─ [0] VM::startPrank(DefaultSender: [0x1804c8AB1F12E6bbf3894d4083f33e07309d1f38])
    │   └─ ← [Return] 
    ├─ [6686] PasswordStore::setPassword("ownerPassword")
    │   ├─ emit SetNetPassword()
    │   └─ ← [Stop] 
    ├─ [3320] PasswordStore::getPassword() [staticcall]
    │   └─ ← [Return] "ownerPassword"
    ├─ [0] VM::startPrank(0x0000000000000000000000000000000000000001)
    │   └─ ← [Return] 
    ├─ [1886] PasswordStore::setPassword("thisisnewpassword")
    │   ├─ emit SetNetPassword()
    │   └─ ← [Stop] 
    ├─ [0] VM::startPrank(DefaultSender: [0x1804c8AB1F12E6bbf3894d4083f33e07309d1f38])
    │   └─ ← [Return] 
    ├─ [1320] PasswordStore::getPassword() [staticcall]
    │   └─ ← [Return] "thisisnewpassword"
    └─ ← [Stop] 
```

### Recommendations
Implement access control to this function, such as providing a function modifier that checks if msg.sender is `s_owner`.
```diff
// Modifier to check if if the caller is the owner
modifier onlyOwner {
    require(msg.sender == s_owner);
    _;
}

// Add the function modifer to setPassword() function
+   function setPassword(string memory newPassword) external onlyOwner {
        s_password = newPassword;
        emit SetNetPassword();
    }

```

### Tools Used
- [Foundry](https://github.com/foundry-rs/foundry)



## H-02: `s_password` is not truly private and the data can still be seen by anyone

### Summary
In this contract, the `s_password` variable is stored as a `private` variable which only allows the variable to be accessed from within the contract. However, this assumption does not hold true as the `private` variable is still accessible via other methods.

### Vulnerability Details
*PasswordStore.sol line13*
```
address private s_owner;
string private s_password;
```

We know that smart contract storage slot is 32 bytes each and the variables in this contract are:

- address `s_owner` = 20 bytes
- string `s_password` = dynamic

Hence, we know that `s_owner` will be stored in slot 0, and `s_password` will be stored in slot 1 as string byte size is dynamic and will occupy the next slot. Hence, by calling slot 1 of the contract memory, we will be able to attain the hexadecimal representation of `s_password`.


### Impact/Proof of Concept
To prove that the `s_password` is accessible by anyone, we use Foundry's `vm.load()` method to load `s_password` from slot 1 of the contract address. Thereafter, we convert the hexadecimal value to a string and print the result.
```
function test_non_owner_can_read_password() public {
    bytes32 data = vm.load(address(passwordStore), bytes32(uint256(1)));
    string memory convertedString = string(abi.encodePacked(data));
    console.log("password:", convertedString);
}
```
Results:
```
Ran 1 test for test/PasswordStore.t.sol:PasswordStoreTest
[PASS] test_non_owner_can_read_password() (gas: 9118)
Logs:
  password: myPassword

Traces:
  [9118] PasswordStoreTest::test_non_owner_can_read_password()
    ├─ [0] VM::startPrank(0x0000000000000000000000000000000000000001)
    │   └─ ← [Return] 
    ├─ [0] VM::load(PasswordStore: [0x90193C961A926261B756D1E5bb255e67ff9498A1], 0x0000000000000000000000000000000000000000000000000000000000000001) [staticcall]
    │   └─ ← [Return] 0x6d7950617373776f726400000000000000000000000000000000000000000014
    ├─ [0] console::log("password:", "myPassword\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\u{14}") [staticcall]
    │   └─ ← [Stop] 
    └─ ← [Stop] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.95ms (912.52µs CPU time)
```

### Recommendations
Data on the blockchain are not truly secret. When storing data that is sensitive, it is recommended to encrypt the data first or to use off-chain solutions to store these sensitive data instead.

### Tools Used
- [Foundry](https://github.com/foundry-rs/foundry)
