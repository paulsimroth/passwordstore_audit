### [S-#] Variables stored in Storage are visible to anyone; password is not a private password

**Description:** All data stored on chain is visible to anyone and can be seen on the blockchain. declaring a variable or function as private does not mean it can´t be seen by other people. Therefore the `PasswordStore::s_password` is exposed and not secure. This variable is intended to be only accessed with `PasswordStore::getPassword`, which is intended to be only called by the contract owner.

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