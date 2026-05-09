# what's a net cat? — picoCTF Writeup

**Challenge:** what's a net cat?  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{nEtCat_Mast3ry_0d33dA2C}`  

---

## Description

> Using netcat (nc) is going to be pretty important. Can you connect to fickle-tempest.picoctf.net at port 61528 to get the flag?

**Hint 1:** `nc tutorial`

---

## Background Knowledge (Read This First!)

### What is netcat (`nc`)?

**Netcat** (nc) is a versatile networking tool that opens a raw TCP connection to any server and port. Think of it like a phone call to a remote computer — you connect, it talks, you read what it says.

```bash
nc hostname port
```

It's used constantly in CTFs to interact with challenge servers.

---

## Solution

```
┌──(zham㉿kali)-[~]
└─$ nc fickle-tempest.picoctf.net 61528
You're on your way to becoming the net cat master
picoCTF{nEtCat_Mast3ry_0d33dA2C}
```

Connect → server prints the flag → done. ✅ Got the flag! 🎯

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `nc` | Connect to the challenge server | ⭐ Easy |

---

## Key Takeaways

- **`nc hostname port`** is the basic netcat command — the server prints the flag immediately on connection
- Netcat requires no setup or authentication here — just connect and read
- The flag `nEtCat_Mast3ry` → "netcat mastery" — your first step toward mastering nc
