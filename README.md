# DPI Engine

A multithreaded Deep Packet Inspection engine written in pure Python — zero third-party dependencies. It reads PCAP files, parses Ethernet/IPv4/TCP/UDP frames by hand, extracts TLS SNI and HTTP Host headers, classifies traffic by application, and filters packets based on configurable rules before writing a clean output PCAP.

---

## What it does

- Parses raw PCAP files at the byte level — no libpcap, no Scapy
- Extracts TLS Server Name Indication (SNI) from Client Hello packets
- Extracts HTTP Host headers and DNS query names
- Classifies traffic into 20+ applications (YouTube, Netflix, Discord, Spotify, etc.)
- Blocks traffic by IP address, app name, domain, or port
- Tracks per-flow connection state using 5-tuple identifiers
- Runs a real multi-threaded pipeline: Reader → Load Balancers → Fast Path processors → Output writer
- Generates a per-app classification report after each run

---

## Architecture
PCAP Reader
│
▼  (5-tuple hash → select LB)
Load Balancer threads  (2 LB threads)
│
▼  (5-tuple hash → select FP within pool)
Fast Path threads  (4 FP threads, 2 per LB)
│  ┌─────────────────┐
│  │  Rule Manager   │  ← block by IP / app / domain / port
│  │  SNI Extractor  │  ← TLS, HTTP, DNS
│  │  App Classifier │  ← SNI → AppType
│  │  Conn Tracker   │  ← flow state per 5-tuple
│  └─────────────────┘
▼
Output Queue → Output Writer → filtered.pcap
Connection affinity is guaranteed: all packets belonging to the same flow always go to the same Fast Path thread via consistent hashing, so connection state is never accessed across threads without locking.

---

## Project structure
DPI-engine/
├── src/
│   ├── main.py                # CLI entry point (analyze / dpi / generate)
│   ├── pcap_reader.py         # PCAP global header + per-packet read/write
│   ├── packet_parser.py       # Ethernet → IPv4 → TCP/UDP parser, builds 5-tuple
│   ├── sni_extractor.py       # TLS Client Hello SNI, HTTP Host, DNS query extraction
│   ├── dpi_types.py           # AppType enum, SNI→app mapping, shared data classes
│   ├── dpi_engine.py          # DPIEngine (threaded) + SimpleDPIEngine (reference)
│   ├── connection_tracker.py  # Per-flow state machine (NEW→ESTABLISHED→CLASSIFIED→CLOSED)
│   ├── rule_manager.py        # Thread-safe block lists: IP, app, domain, port
│   ├── load_balancer.py       # LB thread: receives packets, hashes to FP pool
│   ├── fast_path.py           # FP thread: inspects, classifies, applies rules
│   ├── thread_safe_queue.py   # Bounded queue with try_push / blocking pop
│   └── generate_test_pcap.py  # Generates synthetic test traffic (TLS + HTTP + DNS)
├── test_dpi.pcap              # Sample capture — ready to use immediately
├── pyproject.toml             # pip-installable package config
├── .gitignore
└── LICENSE

---

## Requirements

- Python 3.7+
- No third-party packages — pure standard library

---

## Install

```bash
git clone https://github.com/sudo-apt-Aniket/DPI-engine.git
cd DPI-engine

python -m venv venv

# Windows
venv\Scripts\activate
# macOS / Linux
source venv/bin/activate

pip install -e .
```

This registers a `dpi-engine` command so you can run it from anywhere — no need to `cd src/` first.

---

## Usage

### Generate test traffic

```bash
cd src
python3 generate_test_pcap.py
# Creates test_dpi.pcap: 16 TLS flows, 2 HTTP flows, 4 DNS queries
```

### Inspect packets

```bash
dpi-engine analyze test_dpi.pcap 10
```
Shows full Ethernet/IP/TCP/UDP header breakdown for the first 10 packets.

### Run DPI classification

```bash
dpi-engine dpi test_dpi.pcap -o out.pcap
```

### Block by app

```bash
dpi-engine dpi test_dpi.pcap -o out.pcap -b youtube -b tiktok
```

App names: `facebook`, `twitter`, `instagram`, `youtube`, `netflix`, `tiktok`, `whatsapp`, `telegram`, `discord`, `zoom`, `spotify`, `github`, `amazon`, `microsoft`, `apple`, `google`

### Block by domain

```bash
dpi-engine dpi test_dpi.pcap -o out.pcap -b somedomain.com
```

Anything that isn't a known app name is treated as a domain.

### Block by IP

```bash
dpi-engine dpi test_dpi.pcap -o out.pcap --block-ip 192.168.1.50
```

### Load rules from a file

```bash
dpi-engine dpi test_dpi.pcap -o out.pcap -r rules.txt
```

Rules file format (one rule per line, `#` for comments):
Block by app (use AppType enum name, uppercase)
app:YOUTUBE
app:TIKTOK
Block by domain
domain:facebook.com
Block by IP
ip:192.168.1.50
Block by port
port:8080

### Full flag reference
dpi-engine dpi --help

---

## Sample output
====================================
DPI Engine v1.0 (Python)
Processing: test_dpi.pcap
Output to:   out.pcap
Opened PCAP file: test_dpi.pcap
Version: 2.4
Snaplen: 65535 bytes
Link type: Ethernet
============================================================
DPI Engine Statistics Report
Total Packets:      77
Total Bytes:        5738
Forwarded Packets:  75
Dropped Packets:    2
TCP Packets:        73
UDP Packets:        4
Other Packets:      0
Active Connections: 0
Fast Path Statistics:
Total Processed:  77
Total Forwarded:  75
Total Dropped:    2
============================================================
Classification Report
Google:     2
YouTube:    2   ← blocked (2 packets dropped)
Facebook:   2
Twitter:    2
Instagram:  1
TikTok:     1
Microsoft:  1
Apple:      1
Amazon:     1
Netflix:    1
Discord:    1
Zoom:       1
Telegram:   1
Spotify:    1

---

## Known limitations

- **QUIC / HTTP3**: the extractor runs the TLS parser on raw QUIC bytes, which won't work reliably — QUIC payloads are encrypted and framed differently from plaintext TLS records. Proper QUIC support would require QUIC header parsing before the TLS layer.
- **DNS compression pointers**: the parser stops cleanly at a pointer byte rather than following it, so compressed DNS names are partially read. Simple uncompressed queries (the most common case) parse correctly.
- **IPv6**: not implemented. Only IPv4 packets are inspected; IPv6 frames are passed through unclassified.
- **Python GIL**: the threading architecture mirrors the C-style pipeline exactly, but CPython's GIL means threads don't achieve true CPU parallelism. The design is correct and demonstrates the architecture clearly; for throughput-critical use you'd want PyPy or a compiled extension for the hot path.

---

## License

MIT
To update it:
powershell# paste the above content into README.md, then:
git add README.md
git commit -m "Rewrite README — accurate, no legacy content"
git push origin main
Ready to write the Medium post whenever you are.
---

## Requirements

- Python 3.7+
- No third-party packages — pure standard library

---

## Install

```bash
git clone https://github.com/sudo-apt-Aniket/DPI-engine.git
cd DPI-engine

python -m venv venv

# Windows
venv\Scripts\activate
# macOS / Linux
source venv/bin/activate

pip install -e .
```

This registers a `dpi-engine` command so you can run it from anywhere — no need to `cd src/` first.

---

## Usage

### Generate test traffic

```bash
cd src
python3 generate_test_pcap.py
# Creates test_dpi.pcap: 16 TLS flows, 2 HTTP flows, 4 DNS queries
```

### Inspect packets

```bash
dpi-engine analyze test_dpi.pcap 10
```
Shows full Ethernet/IP/TCP/UDP header breakdown for the first 10 packets.

### Run DPI classification

```bash
dpi-engine dpi test_dpi.pcap -o out.pcap
```

### Block by app

```bash
dpi-engine dpi test_dpi.pcap -o out.pcap -b youtube -b tiktok
```

App names: `facebook`, `twitter`, `instagram`, `youtube`, `netflix`, `tiktok`, `whatsapp`, `telegram`, `discord`, `zoom`, `spotify`, `github`, `amazon`, `microsoft`, `apple`, `google`

### Block by domain

```bash
dpi-engine dpi test_dpi.pcap -o out.pcap -b somedomain.com
```

Anything that isn't a known app name is treated as a domain.

### Block by IP

```bash
dpi-engine dpi test_dpi.pcap -o out.pcap --block-ip 192.168.1.50
```

### Load rules from a file

```bash
dpi-engine dpi test_dpi.pcap -o out.pcap -r rules.txt
```

Rules file format (one rule per line, `#` for comments):
Block by app (use AppType enum name, uppercase)
app:YOUTUBE
app:TIKTOK
Block by domain
domain:facebook.com
Block by IP
ip:192.168.1.50
Block by port
port:8080

### Full flag reference
dpi-engine dpi --help

---

## Sample output
====================================
DPI Engine v1.0 (Python)
Processing: test_dpi.pcap
Output to:   out.pcap
Opened PCAP file: test_dpi.pcap
Version: 2.4
Snaplen: 65535 bytes
Link type: Ethernet
============================================================
DPI Engine Statistics Report
Total Packets:      77
Total Bytes:        5738
Forwarded Packets:  75
Dropped Packets:    2
TCP Packets:        73
UDP Packets:        4
Other Packets:      0
Active Connections: 0
Fast Path Statistics:
Total Processed:  77
Total Forwarded:  75
Total Dropped:    2
============================================================
Classification Report
Google:     2
YouTube:    2   ← blocked (2 packets dropped)
Facebook:   2
Twitter:    2
Instagram:  1
TikTok:     1
Microsoft:  1
Apple:      1
Amazon:     1
Netflix:    1
Discord:    1
Zoom:       1
Telegram:   1
Spotify:    1

---

## Known limitations

- **QUIC / HTTP3**: the extractor runs the TLS parser on raw QUIC bytes, which won't work reliably — QUIC payloads are encrypted and framed differently from plaintext TLS records. Proper QUIC support would require QUIC header parsing before the TLS layer.
- **DNS compression pointers**: the parser stops cleanly at a pointer byte rather than following it, so compressed DNS names are partially read. Simple uncompressed queries (the most common case) parse correctly.
- **IPv6**: not implemented. Only IPv4 packets are inspected; IPv6 frames are passed through unclassified.
- **Python GIL**: the threading architecture mirrors the C-style pipeline exactly, but CPython's GIL means threads don't achieve true CPU parallelism. The design is correct and demonstrates the architecture clearly; for throughput-critical use you'd want PyPy or a compiled extension for the hot path.

---

## License

MIT
