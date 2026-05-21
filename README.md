<div align="center">
<pre>
  ███╗   ██╗███╗   ███╗███████╗██╗   ██╗███████╗███████╗
  ████╗  ██║████╗ ████║██╔════╝██║   ██║╚════██║╚════██║
  ██╔██╗ ██║██╔████╔██║█████╗  ██║   ██║    ██╔╝    ██╔╝
  ██║╚██╗██║██║╚██╔╝██║██╔══╝  ██║   ██║   ██╔╝    ██╔╝
  ██║ ╚████║██║ ╚═╝ ██║██║     ╚██████╔╝██████╔╝██████╔╝
  ╚═╝  ╚═══╝╚═╝     ╚═╝╚═╝      ╚═════╝╚═════╝ ╚═════╝
</pre>

**CTF Enumeration Automation Tool — a bash script that automates port recon (nmap), tcp (SYN scan) and udp ports. Detects http/https services and launches both directory (gobuster) and subdomain (wfuzz) fuzzing.**

![Bash](https://img.shields.io/badge/bash-5.0%2B-4EAA25?style=flat-square&logo=gnubash&logoColor=white)
![Platform](https://img.shields.io/badge/platform-Linux-blue?style=flat-square&logo=linux&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-orange?style=flat-square)

</div>

---

## Features

- **Modular design** — call any stage standalone or run the full pipeline at once
- **Persistent target config** — set the target once, reuse it across commands
- **Auto workspace** — creates an organized directory structure per machine
- **SYN scan + targeted scan** — fast full-port discovery followed by version/script detection
- **Optional UDP scan** — top 200 UDP ports on demand
- **Smart HTTP detection** — reads nmap results to find HTTP ports automatically
- **Gobuster directory brute-force** — with domain fallback if IP scan fails
- **wfuzz subdomain fuzzing** — smart baseline probe to filter false positives silently
- **Silent tool output** — tools run in the background; only a clean summary is shown

---

## Requirements

| Tool | Purpose | Install |
|---|---|---|
| `nmap` | Port scanning | `apt install nmap` |
| `gobuster` | Directory brute-force | `apt install gobuster` |
| `wfuzz` | Subdomain fuzzing | `apt install wfuzz` |

> **Root is required** for `portscan` and `enum` (nmap SYN scan `-sS`).

### Recommended wordlists

nmfuzz uses these paths by default. Install [SecLists](https://github.com/danielmiessler/SecLists) to have them available:

```
/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt   ← gobuster
/usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt  ← wfuzz
```

```bash
apt install seclists
# or
git clone https://github.com/danielmiessler/SecLists /usr/share/wordlists/SecLists
```

---

## Installation

```bash
git clone https://github.com/jnavarroO9/nmfuzz.git
cd nmfuzz
chmod +x nmfuzz.sh
sudo mv nmfuzz.sh /usr/local/bin/nmfuzz
```

---

## Usage

```
nmfuzz <command> [options]
```

### Commands

#### `settarget <ip> <name>`
Configures the active target and creates the workspace directory structure. Saves the configuration to `~/.nmfuzz.conf` so subsequent commands don't need `--target`.

```bash
nmfuzz settarget 10.10.11.23 HackMe
```

```
HackMe/
├── nmap/       ← scan output files
├── content/    ← web enumeration, screenshots, notes
└── exploits/   ← exploits and payloads
```

---

#### `portscan [--target <ip> <name>] [--udp]`

Runs a three-phase nmap enumeration:

| Phase | Command | Output file |
|---|---|---|
| 1 | SYN scan all ports (`-sS -p- --min-rate 5000 -n -Pn`) | `nmap/allPorts` (grepable) |
| 2 | Version + scripts on open ports (`-sCV`) | `nmap/targeted` (nmap format) |
| 3 *(optional)* | UDP top 200 ports (`-sU --top-ports 200`) | `nmap/udp-targeted` (nmap format) |

If any phase fails, the scan is aborted and the error is reported.

```bash
sudo nmfuzz portscan
sudo nmfuzz portscan --udp
sudo nmfuzz portscan --target 10.10.11.23 HackMe --udp
```

---

#### `webscan [--target <ip> <name>] [options]`

Runs a Gobuster directory scan against every HTTP port detected in the `targeted` file. If the scan against the IP fails, it automatically retries using `<name>.htb` as the hostname.

```bash
nmfuzz webscan
nmfuzz webscan -w /opt/SecLists/Discovery/Web-Content/common.txt
nmfuzz webscan -w /opt/wordlists/big.txt -x php,html,txt -t 100
```

| Option | Default | Description |
|---|---|---|
| `-w / --wordlist` | dirbuster medium | Path to wordlist |
| `-x / --ext` | `php,html,txt,js,bak` | Extensions to probe |
| `-t / --threads` | `50` | Concurrent threads |

Output files: `content/gobuster_<port>.txt` (and `gobuster_<port>_domain.txt` on fallback).

---

#### `subfuzz [--target <ip> <name>] [options]`

Runs a wfuzz subdomain fuzzing scan using the `Host:` header. Before the full run, it fires a **50-word probe** to detect the baseline response size of invalid subdomains, then filters that size out automatically (`--hh`). The screen is kept clean; only the result summary is printed.

```bash
nmfuzz subfuzz
nmfuzz subfuzz --domain hackme.htb
nmfuzz subfuzz -d target.thm -w /opt/SecLists/Discovery/DNS/subdomains-top1million-20000.txt
```

| Option | Default | Description |
|---|---|---|
| `-d / --domain` | `<name>.htb` | Base domain to fuzz |
| `-w / --wordlist` | SecLists top 5000 | Path to wordlist |
| `-t / --threads` | `50` | Concurrent threads |

Output file: `content/subdomains_<domain>.txt`

---

#### `enum [--target <ip> <name>] [options]`

Runs the complete pipeline in order: `portscan → webscan → subfuzz`. If the port scan fails, the pipeline stops. Web stages are skipped if no HTTP ports are detected.

```bash
sudo nmfuzz enum --target 10.10.11.23 HackMe
sudo nmfuzz enum --target 10.10.11.23 HackMe --udp --domain hackme.htb
sudo nmfuzz enum --target 10.10.11.23 HackMe -w /opt/wordlists/common.txt --sub-wordlist /opt/wordlists/dns.txt
```

| Option | Description |
|---|---|
| `--target <ip> <name>` | Set target inline (also saves config) |
| `--udp` | Include UDP scan |
| `-w / --wordlist` | Wordlist for gobuster |
| `--sub-wordlist` | Wordlist for wfuzz |
| `-d / --domain` | Domain for subdomain fuzzing |

---

#### `status`

Shows the currently configured target and which output files have been generated.

```bash
nmfuzz status
```

---

## Workflows

### Step-by-step

```bash
# 1. Configure target (creates workspace, saves config)
nmfuzz settarget 10.10.11.23 HackMe

# 2. Port enumeration
sudo nmfuzz portscan

# 3. Directory brute-force (auto-detects HTTP ports from step 2)
nmfuzz webscan -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

# 4. Subdomain fuzzing
nmfuzz subfuzz --domain hackme.htb

# 5. Check what was generated
nmfuzz status
```

### One-liner

```bash
sudo nmfuzz enum --target 10.10.11.23 HackMe --udp --domain hackme.htb \
  -w /opt/SecLists/Discovery/Web-Content/common.txt \
  --sub-wordlist /opt/SecLists/Discovery/DNS/subdomains-top1million-5000.txt
```

### Inline target (skip settarget)

```bash
sudo nmfuzz portscan --target 10.10.11.23 HackMe --udp
nmfuzz webscan  --target 10.10.11.23 HackMe -w /opt/wordlists/big.txt
nmfuzz subfuzz  --target 10.10.11.23 HackMe -d hackme.htb
```

---

## Output structure

```
HackMe/
├── nmap/
│   ├── allPorts              SYN scan — grepable format
│   ├── targeted              Version/script scan — nmap format
│   └── udp-targeted          UDP scan — nmap format (if requested)
├── content/
│   ├── gobuster_80.txt       Directory scan results for port 80
│   ├── gobuster_80_domain.txt  (domain fallback, if IP scan failed)
│   ├── gobuster_443.txt      Directory scan results for port 443
│   └── subdomains_hackme.htb.txt  Subdomain fuzzing results
└── exploits/                 Your exploits and payloads go here
```

---

## Configuration

nmfuzz saves the active target to `~/.nmfuzz.conf`:

```ini
TARGET_IP=10.10.11.23
TARGET_NAME=HackMe
WORKSPACE=/home/user/HackMe
```

This file is automatically loaded by every command, so you only need to call `settarget` (or `--target`) once per machine.

---

## Notes

- `portscan` and `enum` **require root** due to nmap's raw socket SYN scan (`-sS`).
- If `gobuster` fails against the IP (e.g. the app requires a specific `Host:` header), it automatically retries using `<name>.htb` as the domain.
- The wfuzz probe run uses the first 50 entries of the wordlist to detect the baseline response. If detection fails, the full scan still runs — just without the char filter.
- Both `gobuster` and `wfuzz` run silently. Only a count summary is printed to the terminal; full results go to the output files.

---

## Roadmap

- [ ] **Standalone UDP scan** — run `portscan --udp-only` without going through the TCP phases first
- [ ] **Skip already-done scans** — if `allPorts` / `targeted` already exist when `portscan` is called, skip those phases and reuse the existing results instead of re-scanning
- [ ] **Concurrent webscan + subfuzz** — a `--concurrent` flag on `enum` / `webscan` to run gobuster and wfuzz in parallel, with a shared thread pool split between both
- [ ] **Auto `/etc/hosts` entry** — optionally append `<ip>  <name>.htb` to `/etc/hosts` on `settarget`
- [ ] **Resume support** — detect mid-run interruptions and pick up from the last completed phase

---



MIT — do whatever you want, happy hacking.
