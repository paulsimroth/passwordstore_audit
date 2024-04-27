### [High-1] Variables stored in Storage are visible to anyone; password is not a private password

**Description:** All data stored on chain is visible to anyone and can be seen on the blockchain. declaring a variable or function as private does not mean it can´t be seen by other people. Therefore the `PasswordStore::s_password` is exposed and not secure. This variable is intended to be only accessed with `PasswordStore::getPassword()`, which is intended to be only called by the contract owner.

**Impact:** Anyone can read the private password, severely breaking the intended functionality of the protocol.

**Proof of Concept:**
This case shows how anyone can read the password directly from the chain.

1. Run local chain
   ```bash
   make anvil
   ```
2. Deploy the contract
   ```bash
   make deploy
   ```
3. Run cast
   `PasswordStore::s_password` is at storage slot `1`

   ```bash
   cast storage <CONTRACT_ADDRESS> 1 --rpc-url http://127.0.0.1:8545
   ```
   The output should look like this:
   `0x6d7950617373776f726400000000000000000000000000000000000000000014`

    You can convert it from hex to atring like this:
    ```bash
    cast parse-bytes32-string 0x6d79 50617373776f726400000000000000000000000000000000000000000014
    ```
    The output is: 
    `myPassword`

**Recommended Mitigation:** Due to this vulnerability, the architecture of this contract needs to be rethought. For example the password could bes encrypted off-chain. This in turn would require the user to keep another password for decryption off-chain.


### [High-2] `PasswordStore::setPassword()` has no access control; anyone could change the password

**Description:** `PasswordStore::setPassword()` is extern al and according to the comments `This function allows only the owner to set a new password.` 

```javascript
    /*
     * @notice This function allows only the owner to set a new password.
     * @param newPassword The new password to set.
     */
    function setPassword(string memory newPassword) external {
@>      // @audit any user can send password
        s_password = newPassword;
        emit SetNetPassword();
    }
```

**Impact:** Anyone can change the password, therefire severly breaking the intended functionality

**Proof of Concept:** add the following test to `PasswordStore.t.sol`.

<details>
<summary>Test Code</summary>

```javascript
   function test_anyone_can_set_password(address randomAddress) public {
      vm.assume(randomAddress != owner);
      vm.prank(randomAddress);
      string memory expectedPassword = "myNewPassword";
      passwordStore.setPassword(expectedPassword);

      vm.prank(owner);
      string memory actualPassword = passwordStore.getPassword();
      assertEq(actualPassword, expectedPassword);
   }
```

</details>

**Recommended Mitigation:** Add Access control conditional or modifier to `PasswordStore::setPassword()****`.

```javascript
   if (msg.sender != s_owner) {
      revert PasswordStore__NotOwner();
   }
```


### [Informational-1] `PasswordStore::getPassword()` has no input parameter; the password can´t be changed

**Description:** `PasswordStore::getPassword()` is meant to allow the owner to retrieve the password. According to the comments this function is also meant to make setting a new password possible; `newPassword The new password to set.`. There is no input parameter, making this function not work like this comment suggests. This function only checks if `msg.sender == owner` and returns `PasswordStore:s_password`.

**Impact:** This function does not work as the comments suggest. It does not take an input parameter and only returns `PasswordStore:s_password`. Therefore this function does not work as the comments suggest.

<details>
<summary>Code</summary>

```javascript
    /*
     * @notice This allows only the owner to retrieve the password.
     * @audit there is no newPassword param; function only returns @param s_password
     * @param newPassword The new password to set.
     */
    function getPassword() external view returns (string memory) {
        if (msg.sender != s_owner) {
            revert PasswordStore__NotOwner();
        }
        return s_password;
    }
```

</details>

**Recommended Mitigation:** Leave the function as is but remove the misleading comment.

```diff
-   * @param newPassword The new password to set.
```