# Lab 04 — Nmap Port Scanning

**Objective:** use Nmap to discover a host, examine selected TCP ports,
identify a running service, and save evidence. Compare scan results before
and after starting a local web server, then explain what the results mean
for security.

**Environment:** Kali Linux · Nmap 7.99 · Python 3.13.12
**Scope:** 127.0.0.1 only (loopback — my own machine).

## 1. Tool check

```
nmap --version   →  Nmap version 7.99
python3 --version →  Python 3.13.12
```

Worked inside a fresh empty `nmap-lab/` folder so the test server couldn't
serve personal documents.

## 2. Host discovery + baseline

```
nmap -sn 127.0.0.1              →  Host is up.
nmap -sT -p 8000,8001 127.0.0.1
```

| Port | State | Service |
|---|---|---|
| 8000/tcp | closed | http-alt |
| 8001/tcp | closed | vcom-tunnel |

`-sn` checks reachability without port scanning; `-sT` does a full TCP
connect scan on the selected ports. Baseline: both closed.

## 3. Start a local web server

```
python3 -m http.server 8000 --bind 127.0.0.1
```

`http://127.0.0.1:8000` returned a directory listing (HTTP 200, server
header `SimpleHTTP/0.6 Python/3.13.12`). Loopback binding keeps it reachable
only from this machine.

## 4. Scan while the server runs

```
nmap -sT -p 8000,8001 127.0.0.1
```

| Port | State | Service |
|---|---|---|
| 8000/tcp | **open** | http-alt |
| 8001/tcp | closed | vcom-tunnel |

Port 8000 flipped closed → open because the Python server is now listening
on it. Port 8001 is untouched. Note: `http-alt` here is just a port-number
lookup — it doesn't confirm the actual application.

## 5. Service detection

```
nmap -sT -sV -p 8000,8001 -oN scan_running.txt 127.0.0.1
```

| Port | State | Service | Version |
|---|---|---|---|
| 8000/tcp | open | http | SimpleHTTPServer 0.6 (Python 3.13.12) |
| 8001/tcp | closed | vcom-tunnel | — |

`-sV` probes the port instead of guessing from the port number, and
correctly identified the real software. Output saved to
[`scans/scan_running.txt`](scans/scan_running.txt).

## 6. Stop the server, rescan

```
nmap -sT -p 8000,8001 -oN scan_stopped.txt 127.0.0.1
```

Port 8000 returned to **closed** — nothing listening, connections refused.
Saved to [`scans/scan_stopped.txt`](scans/scan_stopped.txt).

## Observation table

| Stage | 8000 | 8001 | Evidence |
|---|---|---|---|
| Before server | closed | closed | Baseline: both closed |
| Server running | open | closed | 8000 open; `http-alt` is a lookup label only |
| Service detection | open | closed | `SimpleHTTPServer 0.6 (Python 3.13.12)` |
| After stopping | closed | closed | Both closed again |

## Takeaways

- **Host up ≠ ports open.** Reachability and attack surface are different
  questions — that's why discovery and port scanning are separate steps.
- **Port state mirrors the running service.** Closed → open → closed tracked
  the web server's lifetime exactly.
- **Basic scan labels are guesses.** `http-alt` was wrong; only `-sV`
  probing identified the real service. Never trust the SERVICE column alone.
- **Open port ≠ vulnerability.** It proves something listens, not that it's
  exploitable. Unneeded services should still be stopped and firewalled —
  smaller attack surface, fewer surprises.
