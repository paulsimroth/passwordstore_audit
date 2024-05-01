<!DOCTYPE html>
<html>
<head>
<title>PasswordStore Audit</title>
<style>
    .full-page {
        width:  100%;
        height:  100vh; /* This will make the div take up the full viewport height */
        display: flex;
        flex-direction: column;
        justify-content: center;
        align-items: center;
    }
    .full-page img {
        max-width:  200;
        max-height:  200;
        margin-bottom: 5rem;
    }
    .full-page div{
        display: flex;
        flex-direction: column;
        justify-content: center;
        align-items: center;
    }
</style>

</head>
<body>

<div class="full-page">
    <img src="https://www.paulsimroth.at/_next/image?url=%2Flogo.png&w=256&q=75" alt="Logo" align="center">
    <div>
    <h1>Protocol Audit Report</h1>
    <h3>Prepared by: paulsimroth</h3>
    <a href="https://www.paulsimroth.at/">paulsimroth.at</a>
    </div>
</div>

</body>
</html>


<!-- Your report starts here! -->

Prepared by: [paulsimroth](https://www.paulsimroth.at)
Lead Auditors: 
- Paul Simroth

# Table of Contents
- [Table of Contents](#table-of-contents)
- [Protocol Summary](#protocol-summary)
- [Disclaimer](#disclaimer)
- [Risk Classification](#risk-classification)
- [Audit Details](#audit-details)
  - [Scope](#scope)
  - [Roles](#roles)
- [Executive Summary](#executive-summary)
  - [Issues found](#issues-found)
- [Findings](#findings)
  - [High](#high)
    - [\[High-1\] Variables stored in Storage are visible to anyone; password is not a private password](#high-1-variables-stored-in-storage-are-visible-to-anyone-password-is-not-a-private-password)
    - [\[High-2\] `PasswordStore::setPassword()` has no access control; anyone could change the password](#high-2-passwordstoresetpassword-has-no-access-control-anyone-could-change-the-password)
  - [Medium](#medium)
  - [Low](#low)
  - [Informational](#informational)
    - [\[Informational-1\] `PasswordStore::getPassword()` has no input parameter; the password can´t be changed](#informational-1-passwordstoregetpassword-has-no-input-parameter-the-password-cant-be-changed)
  - [Gas](#gas)

# Protocol Summary

PasswordStore is a protocol designed to be able to restore and retrieve a password of a user. It is designed to be used by a single user. Only the Owner must be able to set and read the password.

# Disclaimer

The team makes all effort to find as many vulnerabilities in the code in the given time period, but holds no responsibilities for the findings provided in this document. A security audit by the team is not an endorsement of the underlying business or product. The audit was time-boxed and the review of the code was solely on the security aspects of the Solidity implementation of the contracts.

# Risk Classification

|            |        | Impact |        |     |
| ---------- | ------ | ------ | ------ | --- |
|            |        | High   | Medium | Low |
|            | High   | H      | H/M    | M   |
| Likelihood | Medium | H/M    | M      | M/L |
|            | Low    | M      | M/L    | L   |

We use the [CodeHawks](https://docs.codehawks.com/hawks-auditors/how-to-evaluate-a-finding-severity) severity matrix to determine severity. See the documentation for more details.

# Audit Details 

## Scope 

```
./src
-- PasswordStore.sol
```

## Roles

- Owner: User who can set and read the password.
- Outsiders: Should not be able to access the contract functionality.


# Executive Summary

Audit exercise as part of the Cyfrin Updraft Auditing course.

## Issues found

| Severtiy      | Number of Issues found |
| ------------- | ---------------------- |
| High          | 2                      |
| Medium        | 0                      |
| Low           | 0                      |
| Informational | 1                      |
| Total         | 3                      |

# Findings

## High

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

## Medium

X

## Low 

X

## Informational

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

## Gas 

X