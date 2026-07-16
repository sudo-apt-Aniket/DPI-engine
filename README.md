# DPI Engine

A multithreaded **Deep Packet Inspection (DPI) engine** written in pure Python with **zero third-party dependencies**.

It reads raw PCAP files, parses Ethernet/IPv4/TCP/UDP packets manually, extracts TLS SNI, HTTP Host headers and DNS queries, classifies traffic by application, and filters packets using configurable rules before writing a clean output PCAP.

---

## Features

- Parses raw PCAP files at the byte level (no libpcap or Scapy)
- Extracts TLS Server Name Indication (SNI) from Client Hello packets
- Extracts HTTP Host headers and DNS query names
- Classifies traffic into 20+ applications
- Blocks traffic by:
  - IP Address
  - Application
  - Domain
  - Port
- Tracks per-flow connection state using 5-tuples
- Real multithreaded packet-processing pipeline
- Generates a per-application classification report
- Pure Python standard library only

---

# Architecture

```text
PCAP Reader
    │
    ▼ (5-tuple hash → select Load Balancer)
Load Balancer Threads (2)
    │
    ▼ (5-tuple hash → select Fast Path)
Fast Path Threads (4)
    │
    ├── Rule Manager
    │      └── Block by IP / App / Domain / Port
    │
    ├── SNI Extractor
    │      └── TLS / HTTP / DNS
    │
    ├── Application Classifier
    │      └── SNI → AppType
    │
    └── Connection Tracker
           └── Flow State per 5-Tuple
    │
    ▼
Output Queue
    │
    ▼
Output Writer
    │
    ▼
filtered.pcap
```

### Connection Affinity

All packets belonging to the same flow are always processed by the same Fast Path thread through consistent hashing.

This guarantees that per-flow connection state is never accessed across multiple threads, eliminating the need for locking on connection state.

---

# Project Structure

```text
DPI-engine/
│
├── src/
│   ├── main.py
│   │      CLI entry point (analyze / dpi / generate)
│   │
│   ├── pcap_reader.py
│   │      PCAP global header + packet reader/writer
│   │
│   ├── packet_parser.py
│   │      Ethernet → IPv4 → TCP/UDP parser
│   │
│   ├── sni_extractor.py
│   │      TLS SNI / HTTP Host / DNS extraction
│   │
│   ├── dpi_types.py
│   │      AppType enum + SNI mappings
│   │
│   ├── dpi_engine.py
│   │      Threaded DPI engine
│   │
│   ├── connection_tracker.py
│   │      Per-flow state machine
│   │
│   ├── rule_manager.py
│   │      Thread-safe block rules
│   │
│   ├── load_balancer.py
│   │      Load Balancer threads
│   │
│   ├── fast_path.py
│   │      Fast Path worker threads
│   │
│   ├── thread_safe_queue.py
│   │      Bounded thread-safe queue
│   │
│   └── generate_test_pcap.py
│          Synthetic traffic generator
│
├── test_dpi.pcap
│      Sample capture
│
├── pyproject.toml
│      Package configuration
│
├── .gitignore
│
└── LICENSE
```

---

# Requirements

- Python 3.7+
- No third-party dependencies

---

# Installation

```bash
git clone https://github.com/sudo-apt-Aniket/DPI-engine.git

cd DPI-engine

python -m venv venv
```

### Windows

```bash
venv\Scripts\activate
```

### macOS / Linux

```bash
source venv/bin/activate
```

Install the package:

```bash
pip install -e .
```

This registers the command:

```bash
dpi-engine
```

which can be run from anywhere.

---

# Usage

## Generate Test Traffic

```bash
cd src

python generate_test_pcap.py
```

Creates:

- 16 TLS flows
- 2 HTTP flows
- 4 DNS queries

---

## Analyze Packets

```bash
dpi-engine analyze test_dpi.pcap 10
```

Shows a detailed breakdown of the first 10 packets.

---

## Run DPI Classification

```bash
dpi-engine dpi test_dpi.pcap -o out.pcap
```

---

## Block Applications

```bash
dpi-engine dpi test_dpi.pcap -o out.pcap \
    -b youtube \
    -b tiktok
```

Supported application names:

```
facebook
twitter
instagram
youtube
netflix
tiktok
whatsapp
telegram
discord
zoom
spotify
github
amazon
microsoft
apple
google
```

---

## Block a Domain

```bash
dpi-engine dpi test_dpi.pcap \
    -o out.pcap \
    -b facebook.com
```

Unknown names are automatically treated as domains.

---

## Block an IP Address

```bash
dpi-engine dpi test_dpi.pcap \
    -o out.pcap \
    --block-ip 192.168.1.50
```

---

## Load Rules From a File

```bash
dpi-engine dpi test_dpi.pcap \
    -o out.pcap \
    -r rules.txt
```

Example `rules.txt`

```text
# Block Applications
app:YOUTUBE
app:TIKTOK

# Block Domains
domain:facebook.com

# Block IP
ip:192.168.1.50

# Block Port
port:8080
```

---

## Full CLI Help

```bash
dpi-engine dpi --help
```

---

# Sample Output

```text
============================================================

DPI Engine v1.0 (Python)

Processing: test_dpi.pcap
Output to: out.pcap

Opened PCAP file: test_dpi.pcap

Version: 2.4
Snaplen: 65535 bytes
Link Type: Ethernet

============================================================

DPI Engine Statistics Report

Total Packets:       77
Total Bytes:         5738

Forwarded Packets:   75
Dropped Packets:     2

TCP Packets:         73
UDP Packets:         4
Other Packets:       0

Active Connections:  0

Fast Path Statistics

Total Processed:     77
Total Forwarded:     75
Total Dropped:       2

============================================================

Classification Report

Google      : 2
YouTube     : 2   ← blocked (2 packets dropped)
Facebook    : 2
Twitter     : 2
Instagram   : 1
TikTok      : 1
Microsoft   : 1
Apple       : 1
Amazon      : 1
Netflix     : 1
Discord     : 1
Zoom        : 1
Telegram    : 1
Spotify     : 1
```

---

# Known Limitations

### QUIC / HTTP3

The extractor currently runs the TLS parser directly on QUIC payloads.

Since QUIC encrypts and frames TLS differently, reliable HTTP/3 support would require a dedicated QUIC parser.

---

### DNS Compression

Compressed DNS names are not fully resolved.

The parser safely stops at compression pointers instead of following them.

Standard uncompressed DNS queries parse correctly.

---

### IPv6

Only IPv4 traffic is inspected.

IPv6 packets are forwarded without classification.

---

### Python GIL

The architecture mirrors a production C-style multithreaded pipeline.

Because of CPython's Global Interpreter Lock (GIL), worker threads do not execute CPU-bound workloads in true parallel.

The design accurately demonstrates the architecture; for production throughput, PyPy or a compiled implementation would be preferred.

---

# License

MIT
