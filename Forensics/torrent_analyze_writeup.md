# Torrent Analyze — picoCTF Writeup

Challenge: Torrent Analyze  
Category: Forensics  
Difficulty: Medium  
Points: 400  
Flag: `picoCTF{ubuntu-19.10-desktop-amd64.iso}`  
Platform: picoCTF 2022  
Writeup by: zham  

---

## Description

> SOS, someone is torrenting on our network.
>
> One of your colleagues has been using torrent to download some files on the company's network. Can you identify the file(s) that were downloaded? The file name will be the flag, like `picoCTF{filename}`. `Captured traffic`.

## Hints

1. Download and open the file with a packet analyzer like Wireshark.
2. You may want to enable BitTorrent protocol (BT-DHT, etc.) on Wireshark. Analyze -> Enabled Protocols
3. Try to understand peers, leechers and seeds. Article
4. The file name ends with `.iso`

## Background Knowledge

Before I started digging through the capture, I had to refresh a few things I only half-knew.

**A `.pcap` file** is a packet capture. It is a raw recording of every byte that crossed a network interface while the capture was running. Tools like Wireshark and `tshark` can read it back and tell you, packet by packet, who sent what to whom and on which protocol.

**BitTorrent** is a peer-to-peer file sharing protocol. Instead of downloading a file from one server, every downloader (called a **peer** or **leecher**) also uploads pieces of the file to other downloaders. When a peer has the full file and only uploads, it is a **seed**. The more seeds and peers in a swarm, the faster and healthier the download.

**A `.torrent` file** describes a torrent. The most important field inside it is the `info` dictionary, which lists the file name, size, and piece hashes. The `info` dictionary is hashed with SHA-1, and that 20-byte hash is the **`info_hash`**. It is the unique fingerprint of a torrent, the way a magnet link identifies what you are about to download. A magnet link literally wraps this hash, e.g. `magnet:?xt=urn:btih:e2467cbf021192c241367b892230dc1e05c0580e`.

**Mainline DHT (BT-DHT)** is the distributed hash table that BitTorrent clients use to find peers without contacting a central tracker. Every peer is a node in a Kademlia-style DHT. When a client wants peers for a torrent, it sends a `get_peers` query whose `info_hash` is the key, and other nodes reply with either peer addresses or pointers to closer nodes. You can read more on the [Mainline DHT Wikipedia page](https://en.wikipedia.org/wiki/Mainline_DHT).

**Why this matters for the challenge.** The capture is full of BT-DHT `get_peers` packets. Each one carries an `info_hash`. If I can pull the hash the local user is looking up, I can look that hash up on a torrent index and recover the file name. The file name is the flag.

**ISO files** are disk images, normally used to burn an operating system installer to a USB stick or DVD. Ubuntu, Kali, and Debian all distribute their installers as `.iso` files, and Linux users often grab them over BitTorrent to save the project's bandwidth.

## Step-by-step Solution

### Step 1: Get the capture and a working analyzer

I already had `torrent.pcap` from the challenge page. I work in Kali, so I went straight to `tshark` (the CLI version of Wireshark). First check it is installed:

```text
┌──(zham㉿kali)-[~/ctf/torrent-analyze]
└─$ which tshark
/usr/bin/tshark
```

A quick look at the protocol breakdown shows what kind of traffic we are dealing with:

```text
┌──(zham㉿kali)-[~/ctf/torrent-analyze]
└─$ tshark -r torrent.pcap 2>/dev/null | awk '{print $6}' | sort -u
BJNP
BT-DHT
DNS
MDNS
SSDP
TCP
THRIFT
TLSv1
TLSv1.2
UDP
WireGuard
```

Almost all the interesting traffic is `BT-DHT`. The hint told me to enable BT-DHT explicitly, so I confirmed `tshark` was already decoding those packets. Good.

### Step 2: Find the `info_hash` the local user is searching for

The local user in this capture is `192.168.73.132` (it is the only RFC1918 address making outbound requests). I narrowed the BT-DHT view down to packets from that IP and asked tshark to print the bencoded string fields:

```text
┌──(zham㉿kali)-[~/ctf/torrent-analyze]
└─$ tshark -r torrent.pcap \
    -Y 'bt-dht.bencoded.string contains "info_hash" and ip.src == 192.168.73.132' \
    -T fields -e bt-dht.bencoded.string 2>/dev/null | head -3

a,id,17c1ec414b95fc775d7dddcb686693b7863ac1aa,info_hash,e2467cbf021192c241367b892230dc1e05c0580e,q,get_peers,t,6770c723,y,q
a,id,17c1ec414b95fc775d7dddcb686693b7863ac1aa,info_hash,e2467cbf021192c241367b892230dc1e05c0580e,q,get_peers,t,6770c723,y,q
a,id,17c1ec414b95fc775d7dddcb686693b7863ac1aa,info_hash,e2467cbf021192c241367b892230dc1e05c0580e,q,get_peers,t,6770c723,y,q
```

The display filter combines two things: the bencoded string field contains the literal text `info_hash`, and the source IP is the local user. This is basically what the Wireshark GUI does when you press Ctrl+F, search for "info_hash", and then look at the source column.

How to read that output:

- `q,get_peers` -> this is a `get_peers` query, the way a client asks "does anyone know peers for this torrent?"
- `info_hash,e2467cbf021192c241367b892230dc1e05c0580e` -> the 40 hex characters are the SHA-1 `info_hash` of the torrent the user is searching for.
- `t,6770c723` -> the transaction ID, used to match the response to the request. Not useful for the flag, but useful if you want to follow the conversation.

I then confirmed the hash was consistent across many packets, which is a sign it is the active download:

```text
┌──(zham㉿kali)-[~/ctf/torrent-analyze]
└─$ tshark -r torrent.pcap \
    -Y 'bt-dht.bencoded.string contains "info_hash" and ip.src == 192.168.73.132' \
    -T fields -e bt-dht.bencoded.string 2>/dev/null \
    | awk -F',' '{for (i=1;i<=NF;i++) if ($i=="info_hash") print $(i+1)}' \
    | sort | uniq -c | sort -rn

     29 e2467cbf021192c241367b892230dc1e05c0580e
      1 17c1e42e811a83f12c697c21bed9c72b5cb3000d
      1 d59b1ce3bf41f1d282c1923544629062948afadd
      1 17d62de1495d4404f6fb385bdfd7ead5c897ea22
      1 17c02f9957ea8604bc5a04ad3b56766a092b5556
      1 17c0c2c3b7825ba4fbe2f8c8055e000421def12c
      1 17c1e42e811a83f12c697c21bed9c72b5cb3000d
      1 7af6be54c2ed4dcb8d17bf599516b97bb66c0bfd
      1 078e18df4efe53eb39d3425e91d1e9f4777d85ac
```

The local user is hammering the DHT for `e2467cbf021192c241367b892230dc1e05c0580e` (29 requests in a few seconds). That is the active swarm. All the other hashes show up only once, mostly because the user's client briefly checked them, probably for old or leeched torrents.

A fun sanity check: I also ran `strings` on the capture to confirm the user was doing Ubuntu-related things and not something completely different:

```text
┌──(zham㉿kali)-[~/ctf/torrent-analyze]
└─$ strings -n 5 torrent.pcap | grep -E "torrent.ubuntu|ipv6.torrent.ubuntu|bittorrent.ubuntu" | sort -u
bittorrent.ubuntu.com
ipv6.bittorrent.ubuntu.com
ipv6.torrent.ubuntu.com
torrent.ubuntu.com
```

Those hostnames come from DNS lookups the local client made for trackers. Big hint that the file is from Ubuntu's own distribution network.

### Step 3: Turn the hash into a file name

Now I had a hash but no file name. The DHT itself does not store names, only `info_hash` -> peers mappings. To get the name I had to consult a torrent index that has indexed this hash.

The most reliable approach is to build a magnet link and use a torrent search engine:

```text
┌──(zham㉿kali)-[~/ctf/torrent-analyze]
└─$ echo "magnet:?xt=urn:btih:e2467cbf021192c241367b892230dc1e05c0580e"
magnet:?xt=urn:btih:e2467cbf021192c241367b892230dc1e05c0580e
```

I also tried `itorrents.org`, which serves the original `.torrent` file directly from the info hash. That is the cleanest verification, because the `.torrent` file contains the actual `info` dictionary with the `name` field:

```text
┌──(zham㉿kali)-[~/ctf/torrent-analyze]
└─$ curl -s -o ubuntu.torrent \
   "https://itorrents.org/torrent/e2467cbf021192c241367b892230dc1e05c0580e.torrent" \
   && ls -la ubuntu.torrent
-rw-r--r-- 1 zham zham 45981 ubuntu.torrent

┌──(zham㉿kali)-[~/ctf/torrent-analyze]
└─$ python3 -c "import bencode,json; print(json.dumps(bencode.bdecode(open('ubuntu.torrent','rb').read())['info'], indent=2))" | head -3
{
  "name": "ubuntu-19.10-desktop-amd64.iso",
  "piece length": 524288,
```

The `.torrent` says the file is `ubuntu-19.10-desktop-amd64.iso`. This also explains hint 4: the file ends with `.iso`, and Ubuntu 19.10 is famously a "weird release" in Ubuntu history, the only non-LTS in 2019, EOL'd in July 2020, exactly the kind of thing a colleague would torrent instead of pulling from the official mirror.

### Step 4: Wrap it as the flag

The challenge says the file name is the flag in the form `picoCTF{filename}`. I assembled the answer:

```text
picoCTF{ubuntu-19.10-desktop-amd64.iso}
```

## Alternative / Sanity-check Methods

I always like a second way of getting the same answer, both as a sanity check and because the "right" tool depends on what is on the box.

### Method A: pure Wireshark GUI

1. Open `torrent.pcap` in Wireshark.
2. `Analyze -> Enabled Protocols...`, search for `bt-dht`, tick it, click OK. (Tshark has it on by default, but the hint assumes the GUI may not.)
3. Press `Ctrl+F`, switch the search to "String" and "Packet bytes", search for `info_hash`.
4. Pick one of the `get_peers` packets from `192.168.73.132` and copy the hash.
5. Drop the hash into Google. The first result for me was the Ubuntu 19.10 release tracker on old-releases.ubuntu.com and a few torrent indexes showing `ubuntu-19.10-desktop-amd64.iso`.

### Method B: a small Python script that pulls all hashes at once

I wrote this one for fun. It does not need `tshark` and is handy for the bigger pcaps:

```text
┌──(zham㉿kali)-[~/ctf/torrent-analyze]
└─$ nano find_hashes.py
```

Inside the editor, I pasted:

```python
import re
import sys
from collections import Counter

data = open(sys.argv[1], "rb").read()
hashes = Counter(re.findall(rb"[0-9a-f]{40}", data))
for h, c in hashes.most_common():
    print(c, h.decode())
```

Then I saved and ran it:

```text
┌──(zham㉿kali)-[~/ctf/torrent-analyze]
└─$ python3 find_hashes.py torrent.pcap | head -5
37 e2467cbf021192c241367b892230dc1e05c0580e
 2 17c1e42e811a83f12c697c21bed9c72b5cb3000d
 2 d59b1ce3bf41f1d282c1923544629062948afadd
 2 17d62de1495d4404f6fb385bdfd7ead5c897ea22
 1 17c02f9957ea8604bc5a04ad3b56766a092b5556
```

Caveat: this just regexes for 40 hex characters, so it picks up node IDs too. That is fine for a quick top-N, and `e2467cbf021192c241367b892230dc1e05c0580e` is still the runaway winner. For an exact list, prefer the `tshark` approach in the main solution.

### Method C: just use the magnet link directly

If I had a torrent client installed I could have run `transmission-cli -i 'magnet:?xt=urn:btih:e2467cbf021192c241367b892230dc1e05c0580e'` and read the torrent's display name without ever touching Google. On my headless box I do not have transmission, so I used `itorrents.org` instead. Same result.

## What Happened Internally

A rough timeline of the capture (timestamps in seconds since the first packet):

- 0.0 s -> External DHT node (79.252.29.145) `find_node`s the local client. The local client replies with 8 nodes (DHT bootstrap).
- ~3.3 s -> Local client makes DNS A/AAAA lookups for `torrent.ubuntu.com` and `ipv6.torrent.ubuntu.com` (Ubuntu tracker DNS).
- ~3.4 s -> Local client opens TLS to 91.189.95.21 (Ubuntu's tracker web host).
- 7.5 - 9.0 s -> Local client `ping`s several DHT nodes, gets `nodes` replies, builds its routing table.
- 7.68 s -> An external peer (95.146.167.87) `announce_peer`s to the local client for `Zoo (2017) 720p WEB-DL x264 ESubs - MkvHub.Com`. A separate torrent, included in the DHT for the sake of routing.
- 7 - 22 s -> Local client `get_peers` for many hashes. Most of them are DHT bootstrap queries, the DHT itself uses random keys to fill the routing table.
- 20.6 - 22.0 s -> Local client `get_peers` 29 times for `e2467cbf021192c241367b892230dc1e05c0580e`. This is the active download.
- 22 s onward -> External DHT nodes respond with `values` lists (peer IP:port pairs) for the active hash. Up to 100 peers per reply.

The whole exchange happens over UDP, the local client on port 51413 talking to a mix of random external nodes on ports 6881-6889 (the BitTorrent client port range) and high ports. There is no actual file transfer in the capture, the user was just joining the swarm and announcing their presence. That is enough, because the `info_hash` is the only thing I needed.

## Tools Used

| Tool                  | Why I used it                                                                                                            |
|-----------------------|--------------------------------------------------------------------------------------------------------------------------|
| `wget`                | Downloaded `torrent.pcap` from the challenge URL.                                                                       |
| `tshark`              | CLI Wireshark. Decoded BT-DHT, extracted `info_hash` fields, and let me filter on `ip.src == 192.168.73.132`.            |
| `awk`, `sort`, `uniq` | Grouped the extracted hashes by frequency so I could spot the active download.                                          |
| `strings`             | Sanity check, confirmed the user was looking up `torrent.ubuntu.com` DNS names.                                          |
| `curl`                | Pulled the `.torrent` file from `itorrents.org` by info hash, so I could read the `name` field without trusting Google. |
| `bencode.py`          | Decoded the `.torrent` file to read the `name` field directly.                                                          |
| `python3`             | Quick regex script to count every 40-hex-char string in the capture as a backup approach.                                |

## Key Takeaways

- The `info_hash` in a BT-DHT packet is the unique fingerprint of a torrent. If you can capture it, you can usually recover the file name from a torrent search engine or a service like `itorrents.org`.
- A good first filter in Wireshark is `bt-dht.bencoded.string contains "info_hash" and ip.src == <local_ip>`. That strips out all the noise from external DHT traffic and leaves only the hashes the local user actually cares about.
- The hash that is requested the most is the active download. Other hashes are almost always the DHT itself doing routing-table maintenance (find_node, get_peers with random keys).
- The "request count" trick is a tiny but useful forensic move. If you see one `info_hash` show up dozens of times and others only once, the high-frequency one is almost always the swarm of interest.
- A magnet link is just `magnet:?xt=urn:btih:<40-hex-info_hash>`. Drop it into any torrent client and the client tells you the file name and size before you even start downloading.
- And the flag is `picoCTF{ubuntu-19.10-desktop-amd64.iso}`, the Ubuntu 19.10 (Eoan Ermine) desktop amd64 installer, an ISO from 2019 that was famously short-lived as a non-LTS release. The file name ends with `.iso` just like hint 4 promised, and "Eoan Ermine" was Ubuntu's only non-LTS release in 2019, which is why the colleague had to torrent it after the official mirrors moved it to old-releases.
