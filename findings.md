### [H-1] Storing the password on-chain makes it visible to anyone, and no longer private.

**Description:** All data store on-chain is visible to to anyone, and can be read directly from the blockchain. The `PasswordStore::s_password` varaible is intended to be a private variable and only accessed throught the `PasswordStore::getPassword` function, which is intended to be only called by the owner of the contract.

I show one such method of reading any data off chain below.

**Impact:** Anyone is able to read the private password, severly breaking the functionality of the protocol.

**Proof of Concept:** (Proof of Code)
The below test case shows that anyone can read the password directly from the blockchain
1. Create a locally running chain

```bash
make anvil
```

2. Deploy the contract to the chain

``` bash
make deploy
```

3. Run the storage tool
I made use of `1` because it is the storage slot of `s_password` in the contract

``` bash
cast storage <ADDRESS_HERE> 1 --rpc-url http://127.0.0.1:8545
```
You will get an output like this
0x6d7950617373776f726400000000000000000000000000000000000000000014

You can the parse the hex into a string with
``` bash
cast parse-bytes32-string 0x6d7950617373776f726400000000000000000000000000000000000000000014
```
And get an output of:
``` bash
myPassword
```

**Recommended Mitigation:** Due to this, the overall architecture of the contract should be rethought. One could encrypt the password off-chain, and then store the encrypted password on-chain. This would require the user to remember another password off-chain to decrypt the stored password. However, you're also likely want to remove the view function as you wouldn't want the user to accidentally send a transaction with this decryption key. 

**Likelihood & Impact**
- Impact: HIGH
- Likelihood: HIGH
- Severity: HIGH

### [H-2] TITLE `PasswordStore::setPassword` has no access control, meaning a non-owner can change the password

**Description:** The `PasswordStore::setPassword` function is set to be an `external` function, however the purpose of the smart contract and function's natspec indicate that `This function allows only the owner to set a new password.`

``` javascript
    function setPassword(string memory newPassword) external {
    @>      // @audit - No access controls
            s_password = newPassword;
            emit SetNetPassword();
        }
```

**Impact:** Anyone can `set or change` the stored password, severly breaking the contract's intended functionality

**Proof of Concept:** Add the following to the `PasswordStore.t.sol` test file.

<details>
<summary>View Code</summary>

``` javascript
    function test_anyone_can_set_password(address randomAddress) public {
            vm.assume(randomAddress != owner);
            vm.startPrank(randomAddress);
            string memory expectedPassword = "myNewPassword";
            passwordStore.setPassword(expectedPassword);

            vm.startPrank(owner);
            string memory actualPassword = passwordStore.getPassword();
            assertEq(actualPassword, expectedPassword);
        }
```

</details>

**Recommended Mitigation:** Add an access control conditional to `PasswordStore::setPassword`.

```javascript
    if(msg.sender != s_owner){
        revert PasswordStore__NotOwner();
    }
```

Likelihood & Impact
- Impact: HIGH
- Likelihood: HIGH
- Severity: HIGH

### [I-1] TITLE `PasswordStore::getPassword` natspec indicates a parameter that doesn't exist, causing the natspec to be incorrect.

**Description:**

``` javascript
    /*
    * @notice This allows only the owner to retrieve the password.
@>  * @param newPassword The new password to set.
    */
    function getPassword() external view returns (string memory) {}
```

The `PasswordStore::getPassword` function signature is `getPassword()` while the natspec says it should be `getPassword(string)`.

**Impact** The natspec is incorrect

**Recommended Mitigation:** Remove the incorrect natspec line

```diff
/*
*@notice This allows only the owner to retrieve the password.
- *@param newPassword The new password to set.
*/
```

Likelihood & Impact
- Impact: HIGH
- Likelihood: NONE
- Severity: INFORMATIONAL/NON-CRITICAL

