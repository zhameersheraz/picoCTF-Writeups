# KSECRETS — picoCTF Writeup

**Challenge:** KSECRETS  
**Category:** General Skills  
**Difficulty:** Medium  
**Flag:** `picoCTF{ks3cr375_41n7_s4f3_e0eeefa6}`  
**Platform:** picoCTF 2026  
**Writeup by:** zham  

---

## Description

> We have a kubernetes cluster setup and flag is in the secrets. You think you can get it?
> Please wait for a minute for the application to be configured!
> Kubernetes is running at: `green-hill.picoctf.net:64209`
> You can find your configuration file here.

**Hint 1:** `Where are secrets usually stored in Kubernetes`  
**Hint 2:** `How are Kubernetes secrets stored internally? Can you decode them?`  
**Hint 3:** `Please ignore TLS`

**Download:** `kubeconfig.yaml`

---

## Background Knowledge (Read This First!)

### What is Kubernetes?

**Kubernetes** (K8s) is a container orchestration platform — it manages and runs containerised applications across multiple servers. It organises everything into **namespaces**, and resources like pods, services, and secrets live inside namespaces.

### What are Kubernetes Secrets?

**Secrets** are Kubernetes objects used to store sensitive data like passwords, API keys, and flags. They appear to be protected but they are actually just **base64-encoded** — not encrypted. Anyone with the right permissions to read a secret can decode it instantly. This is what Hint 2 points to.

### What is kubeconfig?

A **kubeconfig** file tells `kubectl` (the Kubernetes CLI) how to connect to a cluster. It contains:
- The **server address** to connect to
- **TLS certificates** for authentication
- A **client key** to prove identity

The provided `kubeconfig.yaml` had `127.0.0.1:6443` as the server address — a localhost placeholder. We need to replace it with the actual challenge server `green-hill.picoctf.net:64209`.

### What is `kubectl`?

**`kubectl`** is the command-line tool for interacting with Kubernetes clusters. Key commands used in this challenge:

```bash
kubectl get secrets --all-namespaces    # list all secrets across all namespaces
kubectl get secret <name> -n <ns> -o yaml  # read a specific secret
```

---

## Solution — Step by Step

### Step 1 — Download kubectl

`kubectl` is not installed by default on Kali. Download it directly from the official source:

```
┌──(zham㉿kali)-[~]
└─$ curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
┌──(zham㉿kali)-[~]
└─$ chmod +x ~/kubectl
```

### Step 2 — Fix the kubeconfig server address

The kubeconfig points to `127.0.0.1:6443` (localhost) which does not work from our machine. Replace it with the actual challenge server:

```
┌──(zham㉿kali)-[~]
└─$ cp /media/sf_downloads/kubeconfig.yaml /tmp/kube.yaml
┌──(zham㉿kali)-[~]
└─$ sed -i 's|https://127.0.0.1:6443|https://green-hill.picoctf.net:64209|' /tmp/kube.yaml
```

### Step 3 — Try the default namespace first

```
┌──(zham㉿kali)-[~]
└─$ ~/kubectl --kubeconfig=/tmp/kube.yaml --insecure-skip-tls-verify get secrets
No resources found in default namespace.
```

Nothing in the default namespace. The flag must be in a different namespace.

### Step 4 — List secrets across all namespaces

```
┌──(zham㉿kali)-[~]
└─$ ~/kubectl --kubeconfig=/tmp/kube.yaml --insecure-skip-tls-verify get secrets --all-namespaces
NAMESPACE     NAME                       TYPE                               DATA   AGE
kube-system   chart-values-traefik       helmcharts.helm.cattle.io/values   1      8m9s
kube-system   chart-values-traefik-crd   helmcharts.helm.cattle.io/values   0      8m9s
kube-system   k3s-serving                kubernetes.io/tls                  2      8m11s
picoctf       ctf-secret                 Opaque                             1      8m3s
```

There it is — `ctf-secret` in the `picoctf` namespace.

### Step 5 — Read the secret

```
┌──(zham㉿kali)-[~]
└─$ ~/kubectl --kubeconfig=/tmp/kube.yaml --insecure-skip-tls-verify get secret ctf-secret -n picoctf -o yaml
apiVersion: v1
data:
  flag: cGljb0NURntrczNjcjM3NV80MW43X3M0ZjNfZTBlZWVmYTZ9Cg==
kind: Secret
...
type: Opaque
```

The flag field contains a base64-encoded value.

### Step 6 — Decode the base64 value

```
┌──(zham㉿kali)-[~]
└─$ echo "cGljb0NURntrczNjcjM3NV80MW43X3M0ZjNfZTBlZWVmYTZ9Cg==" | base64 -d
picoCTF{ks3cr375_41n7_s4f3_e0eeefa6}
```

---

## One-liner to Extract and Decode Automatically

Once connected, this single command extracts and decodes the flag without manual copying:

```bash
~/kubectl --kubeconfig=/tmp/kube.yaml --insecure-skip-tls-verify \
  get secret ctf-secret -n picoctf \
  -o jsonpath='{.data.flag}' | base64 -d
```

`-o jsonpath='{.data.flag}'` extracts only the flag field value, which is then piped directly into `base64 -d`.

---

## Why `--insecure-skip-tls-verify`?

The kubeconfig contains a self-signed TLS certificate for `127.0.0.1`. Since we changed the server address to `green-hill.picoctf.net`, the certificate no longer matches and TLS verification would fail. Hint 3 says "Please ignore TLS" — hence `--insecure-skip-tls-verify`.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `curl` | Download the kubectl binary | Easy |
| `sed` | Replace the server address in kubeconfig | Easy |
| `kubectl get secrets --all-namespaces` | Find the secret across all namespaces | Medium |
| `kubectl get secret -o yaml` | Read the raw secret including base64 data | Medium |
| `base64 -d` | Decode the flag value | Easy |

---

## Key Takeaways

- **Kubernetes secrets are base64-encoded, not encrypted** — anyone with read access to a secret can trivially decode it
- **Always check all namespaces** — the default namespace is often empty in CTF challenges; use `--all-namespaces` to see everything
- **kubeconfig server addresses may need updating** — the provided file pointed to localhost; always check and correct the server field
- **`-o jsonpath`** is the cleanest way to extract a specific field from Kubernetes output without parsing YAML manually
- The flag `ks3cr375_41n7_s4f3` is "ksecrets ain't safe" — Kubernetes secrets provide access control but not real encryption
