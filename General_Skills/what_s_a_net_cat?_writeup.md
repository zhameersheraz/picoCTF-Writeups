# what's a net cat? — picoCTF Writeup

**Challenge:** what's a net cat?  
**Category:** General Skills  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{nEtCat_Mast3ry_aC66D475}`  
**Platform:** picoCTF 2019  
**Writeup by:** zham  

---

## Description

> Using netcat (nc) is going to be pretty important. Can you connect to fickle-tempest.picoctf.net at port 55314 to get the flag?

**Instance:** `fickle-tempest.picoctf.net:55314`

---

## Hints

1. nc tutorial

---

## Background Knowledge (Read This First!)

### What is netcat?

`netcat` (often invoked as `nc`) is sometimes called the "TCP/IP Swiss army knife." It can open a TCP connection to a host and port, send data, and print whatever comes back. It is one of the most useful tools in a CTF player's toolbox — a chunk of the "General Skills" challenges in picoCTF (and most other beginner CTFs) are literally just "connect here and read the flag."

The basic syntax is:

```
nc <host> <port>
```

Once connected, the server will print some bytes and the connection will usually stay open until either side hangs up. If the server writes the flag and then closes, you will see the flag in your terminal.

### nc vs ncat vs socat

`nc` exists in several flavors:
- **OpenBSD nc** — minimal, no `-e` for executing commands
- **Traditional/GNU nc** — supports `-e` (often disabled in distro builds)
- **nmap's ncat** — modern, supports SSL/TLS, has `--ssl`
- **socat** — netcat on steroids, much more configuration

For a basic "connect and read" challenge, any of them will do. Kali ships with the OpenBSD version by default; both work fine here.

---

## Solution

### Step 1: Make sure the instance is up

The challenge pane shows the instance is running with about 30 minutes on the clock. If you reload the page, the host and port might change — always copy them from the live instance panel, not from a writeup.

### Step 2: Connect with nc

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nc fickle-tempest.picoctf.net 55314
You're on your way to becoming the net cat master
picoCTF{nEtCat_Mast3ry_aC66D475}
```

That is the whole solve. The server prints a friendly message, the flag, and then closes the connection (or stays open — `Ctrl+C` to drop it).

### Step 3: Submit the flag

**Flag:** `picoCTF{nEtCat_Mast3ry_aC66D475}`

---

## What Happened Internally (Timeline)

| Step | What I did | What the server did |
|---|---|---|
| 1 | Typed `nc fickle-tempest.picoctf.net 55314` | Opened a TCP socket on port 55314. |
| 2 | Read incoming bytes | Server wrote the welcome line and the flag. |
| 3 | `Ctrl+C` to disconnect | Server closed the socket. |

There is no client-server interaction here. The flag is dumped on connect — there is nothing to send, no input prompt, no game. Pure "read the wire" exercise.

---

## Alternative Solve Methods

### Method 1: PowerShell on Windows (no nc needed)

If you do not have `nc` handy (you are on a fresh Windows box, for example), PowerShell can do the same thing with `TcpClient`:

```powershell
$client = New-Object System.Net.Sockets.TcpClient('fickle-tempest.picoctf.net', 55314)
$stream = $client.GetStream()
$reader = New-Object System.IO.StreamReader($stream)
$reader.ReadToEnd()
```

The downside is that `ReadToEnd` blocks until the server closes the connection, so this only works for one-shot dumps. For a "wait and read" pattern use a small loop with `$stream.DataAvailable`.

### Method 2: Python one-liner

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 -c "import socket; s=socket.socket(); s.connect(('fickle-tempest.picoctf.net', 55314)); print(s.recv(4096).decode())"
```

Useful if you want to script it or pipe the output to another tool.

### Method 3: curl / wget

`curl` and `wget` speak HTTP, not raw TCP, so they will not help with a non-HTTP server. If the server happens to also respond to HTTP (it does not here), `curl` would work. For raw TCP, `nc` is the right tool.

### Method 4: telnet

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ telnet fickle-tempest.picoctf.net 55314
```

`telnet` predates `nc` and does basically the same thing for TCP. You will see "Connection closed by foreign host" after the flag, which is normal.

### Method 5: Browser (if the server speaks HTTP)

If the challenge ever wraps the response in HTTP, you can just open the URL in a browser:

```
http://fickle-tempest.picoctf.net:55314/
```

For this challenge, the server sends plain TCP without HTTP framing, so the browser would just spin. Stick with `nc` for raw TCP and use the browser only when the server actually serves HTML.

---

## Tools Used

| Tool | Purpose |
|---|---|
| `nc` (netcat) | Open a raw TCP connection and read the response |
| PowerShell `TcpClient` (alt) | Windows-native fallback when nc is not installed |
| Python `socket` (alt) | Scripted alternative for when you want to automate it |

---

## Key Takeaways

- **`nc <host> <port>` is the single most useful first command in any CTF.** When in doubt, point netcat at it. PicoCTF will throw this command at you in dozens of challenges; the variations are mostly about the protocol layered on top of the TCP connection.
- **The instance host and port change on every reset.** Always copy them from the live challenge panel, not from a writeup or your own history. Treating them as ephemeral is a habit worth keeping.
- **If the server sends the flag and closes, you are done.** No need to send anything — just read and submit. Reading more than you need to is fine; sending random input is usually not.
- **For raw TCP, `nc` is the answer. For HTTP, use `curl` or a browser.** Knowing which tool fits which protocol saves a lot of head-scratching on General Skills challenges.
- **Wordplay, as always:** the challenge name "what's a net cat?" is a pun on `nc` (netcat) sounding like "net cat" (a feline that surfs the net, presumably). The flag decodes from leetspeak as **netcat mastery** — the same pun, doubled. The author is having fun with the homophone.
