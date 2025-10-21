# Fast, Reliable UDP File Transfer

This is a clean-room implementation of a UDP-based, reliable file transfer tool
designed for EE 542 Lab 2. It uses a simple custom protocol (INFO/DATA/NACK/FIN)
with batched negative acknowledgements (NACK) for efficient recovery over
lossy/high-latency links.

Important: Follow your institution's policy on collaboration and citation. If you
use this code, cite it in your submission per course rules.

## Build

```
make
```

Generates two binaries: `ft_client` and `ft_server`.

## Run

Client sends a file to the server:

```
./ft_server <output_file_path> <listen_port>
./ft_client <input_file_path> <server_ip> <server_port>
```

- The client timestamps from first INFO send to final FIN-ACK receive and prints
  total time and throughput (Mbit/s).
- The server writes the received file to the specified output path.

Environment assumptions: Linux, UDP allowed, 3-node setup (client, VyOS router,
server). Test with MTU 1500 and 9000/9001 consistently across client, router,
and server.

### CLI Options and Tuning

- `ft_client`:
  - `--chunk N`: application payload bytes per DATA message (default 1460; max 65535). Use 1460 for MTU=1500; ~8960 for MTU≈9000.
  - `--rate-mbit X`: request kernel pacing at X Mbit/s (Linux only, best-effort via `SO_MAX_PACING_RATE`). Use with `fq` qdisc (below).
  - `--batch N`: batch N DATA messages per syscall using `sendmmsg` (Linux). Reduces syscall overhead; try 16–64.
  - `--mmap`: memory-map the input file (avoids an extra user-space copy before send).
  - `--zerocopy`: Linux only. Enables `SO_ZEROCOPY` and uses zero-copy send path with batching. Requires a reasonably new kernel/NIC; falls back automatically if unsupported.
  - `--verbose`: print chosen chunk size and pacing; detailed timing.

- `ft_server`:
  - `--verbose`: log retransmission rounds and completion status.

Examples
```
# MTU 1500
./ft_server --verbose /tmp/out.bin 5000
./ft_client --verbose --chunk 1460 --batch 32 /tmp/data.bin <server_ip> 5000

# MTU 9000/9001 (jumbo)
./ft_server --verbose /tmp/out.bin 5000
./ft_client --verbose --chunk 8960 --batch 32 /tmp/data.bin <server_ip> 5000

# Optional kernel pacing to ~100 Mbit/s (Linux only)
sudo tc qdisc add dev <iface> root fq
./ft_client --rate-mbit 100 --chunk 1460 /tmp/data.bin <server_ip> 5000

# Optional mmap + zero-copy (Linux only, advanced)
./ft_client --mmap --zerocopy --chunk 1460 --batch 32 /tmp/data.bin <server_ip> 5000
```

Notes
- For best performance on high RTT/loss paths, prefer larger chunks with jumbo MTU to reduce syscalls/overhead.
- If `SO_MAX_PACING_RATE` is unsupported, `--rate-mbit` is ignored; you can still rely on the app’s light pacing.
- `--zerocopy` uses Linux `SO_ZEROCOPY` to reduce kernel data copies. It requires recent kernels and compatible NICs. If unavailable, the client automatically falls back to the normal path.
`
## Network Emulation (tc)

Case 1 (RTT ~10ms, loss 1%, 100 Mbit/s everywhere):

Router
```
tc qdisc del dev eth0 root || true
tc qdisc del dev eth1 root || true
tc qdisc add dev eth0 root handle 1: netem delay 5ms drop 1%
tc qdisc add dev eth0 parent 1: handle 2: tbf rate 100mbit burst 90155 latency 0.001ms
tc qdisc add dev eth1 root handle 1: netem delay 5ms drop 1%
tc qdisc add dev eth1 parent 1: handle 2: tbf rate 100mbit burst 90155 latency 0.001ms
```

Client & Server
```
sudo tc qdisc del dev <iface> root || true
sudo tc qdisc add dev <iface> root tbf rate 100mbit latency 0.001ms burst 90155
sudo ip link set dev <iface> mtu 1500 up   # or 9000/9001
```

Case 2 (RTT ~200ms, loss 20%, 100 Mbit/s everywhere):

Router
```
tc qdisc del dev eth0 root || true
tc qdisc del dev eth1 root || true
tc qdisc add dev eth0 root handle 1: netem delay 100ms drop 20%
tc qdisc add dev eth0 parent 1: handle 2: tbf rate 100mbit burst 90155 latency 0.001ms
tc qdisc add dev eth1 root handle 1: netem delay 100ms drop 20%
tc qdisc add dev eth1 parent 1: handle 2: tbf rate 100mbit burst 90155 latency 0.001ms
```

Client & Server
```
sudo tc qdisc del dev <iface> root || true
sudo tc qdisc add dev <iface> root tbf rate 100mbit latency 0.001ms burst 90155
sudo ip link set dev <iface> mtu 1500 up   # or 9000/9001
```

Case 3 (RTT ~200ms, no loss, router at 80 Mbit/s):

Router
```
tc qdisc del dev eth0 root || true
tc qdisc del dev eth1 root || true
tc qdisc add dev eth0 root handle 1: netem delay 100ms
tc qdisc add dev eth0 parent 1: handle 2: tbf rate 80mbit burst 90155 latency 0.001ms
tc qdisc add dev eth1 root handle 1: netem delay 100ms
tc qdisc add dev eth1 parent 1: handle 2: tbf rate 80mbit burst 90155 latency 0.001ms
```

Client & Server
```
sudo tc qdisc del dev <iface> root || true
sudo tc qdisc add dev <iface> root tbf rate 100mbit latency 0.001ms burst 90155
sudo ip link set dev <iface> mtu 1500 up   # or 9000/9001
```

## Verification Checklist

- ping both directions: `ping -i 0.2 -c 200 <dest>` (RTT ~10/200ms; loss ~1%/~20% in cases 1/2)
- iperf TCP ~100/80 Mbit/s depending on case; UDP bounded similarly
- Prepare test file on client: `dd if=/dev/urandom of=data.bin bs=1M count=1024`
- After transfer, verify integrity: `md5sum` on both ends

## VMware Setup (Client/VyOS/Server) Step-by-Step

This section provides a concrete walkthrough to run tests in VMware with a Client VM, a VyOS Router VM, and a Server VM.

- Topology
  - Client: `10.200.1.10/24`, gw `10.200.1.1`
  - Router (VyOS): `eth0=10.200.1.1/24`, `eth1=10.200.2.1/24`
  - Server: `10.200.2.10/24`, gw `10.200.2.1`
  - UDP port: `5000`

1) Identify interfaces
```
# Client/Server
ip -br link
# VyOS Router
show interfaces  # or: ip -br link
```

2) Configure IPs
```
# VyOS (Router)
configure
set interfaces ethernet eth0 address 10.200.1.1/24
set interfaces ethernet eth1 address 10.200.2.1/24
commit && save && exit

# Client
sudo ip addr add 10.200.1.10/24 dev <iface>
sudo ip route add default via 10.200.1.1

# Server
sudo ip addr add 10.200.2.10/24 dev <iface>
sudo ip route add default via 10.200.2.1
```

3) Build binaries (on Client and Server VMs)
```
make
# Produces: ./ft_client and ./ft_server
```

4) Prepare test file (Client)
```
mkdir -p ~/test && cd ~/test
dd if=/dev/urandom of=data.bin bs=1M count=1024
```

5) Verify connectivity
```
# Client → Server
ping -c 3 10.200.2.10
# Server → Client
ping -c 3 10.200.1.10
```

6) Install iperf (optional but recommended)
```
sudo apt-get update && sudo apt-get install -y iperf
# TCP test
Server: iperf -s
Client: iperf -c 10.200.2.10
# UDP test
Server: iperf -s -u
Client: iperf -u -c 10.200.2.10 -b 100m
```

7) MTU Pass 1: 1500 on all nodes
```
# Client
sudo ip link set dev <iface> mtu 1500 up
# Server
sudo ip link set dev <iface> mtu 1500 up
# Router (VyOS)
sudo ip link set dev eth0 mtu 1500
sudo ip link set dev eth1 mtu 1500
```

8) Case 1 (RTT≈10ms, loss 1%, 100 Mbit/s)
```
# Clear previous qdiscs
Router: sudo tc qdisc del dev eth0 root || true; sudo tc qdisc del dev eth1 root || true
Client: sudo tc qdisc del dev <iface> root || true
Server: sudo tc qdisc del dev <iface> root || true

# Apply shaping
Router:
sudo tc qdisc add dev eth0 root handle 1: tbf rate 100mbit burst 90155 latency 0.001ms
sudo tc qdisc add dev eth0 parent 1: handle 10: netem delay 5ms drop 1%
sudo tc qdisc add dev eth1 root handle 1: tbf rate 100mbit burst 90155 latency 0.001ms
sudo tc qdisc add dev eth1 parent 1: handle 10: netem delay 5ms drop 1%

Client: sudo tc qdisc add dev <iface> root tbf rate 100mbit latency 0.001ms burst 90155
Server: sudo tc qdisc add dev <iface> root tbf rate 100mbit latency 0.001ms burst 90155

# Verify
Client: ping -i 0.2 -c 200 10.200.2.10
Server: ping -i 0.2 -c 200 10.200.1.10
TCP: iperf ~100 Mbit/s; UDP: iperf -u -b 100m ~100 Mbit/s

# Run transfer
Server: ./ft_server ~/test/out.bin 5000
Client: ./ft_client ~/test/data.bin 10.200.2.10 5000
md5sum ~/test/data.bin
md5sum ~/test/out.bin
```

9) Case 2 (RTT≈200ms, loss 20%, 100 Mbit/s)
```
# Clear qdiscs (as above)
Router: sudo tc qdisc del dev eth0 root || true; sudo tc qdisc del dev eth1 root || true
Client: sudo tc qdisc del dev <iface> root || true
Server: sudo tc qdisc del dev <iface> root || true

# Apply shaping
Router:
sudo tc qdisc add dev eth0 root handle 1: tbf rate 100mbit burst 90155 latency 0.001ms
sudo tc qdisc add dev eth0 parent 1: handle 10: netem delay 100ms drop 20%
sudo tc qdisc add dev eth1 root handle 1: tbf rate 100mbit burst 90155 latency 0.001ms
sudo tc qdisc add dev eth1 parent 1: handle 10: netem delay 100ms drop 20%

Client/Server: sudo tc qdisc add dev <iface> root tbf rate 100mbit latency 0.001ms burst 90155

# Verify
ping -i 0.2 -c 200 (both directions) → RTT ~200ms; drop ~20% ± 20%
UDP iperf: ~20% loss (± 20% variation)

# Run transfer and verify MD5 (same commands as Case 1)
```

10) Case 3 (RTT≈200ms, no loss, Router at 80 Mbit/s)
```
# Clear qdiscs (as above)

# Apply shaping
Router:
sudo tc qdisc add dev eth0 root handle 1: tbf rate 80mbit burst 90155 latency 0.001ms
sudo tc qdisc add dev eth0 parent 1: handle 10: netem delay 100ms
sudo tc qdisc add dev eth1 root handle 1: tbf rate 80mbit burst 90155 latency 0.001ms
sudo tc qdisc add dev eth1 parent 1: handle 10: netem delay 100ms

Client/Server: sudo tc qdisc add dev <iface> root tbf rate 100mbit latency 0.001ms burst 90155

# Verify
ping -i 0.2 -c 200 (both directions) → RTT ~200ms; 0% loss
TCP iperf ~80 Mbit/s; UDP iperf -b 80m ~80 Mbit/s

# Run transfer and verify MD5 (same commands as Case 1)
```

11) MTU Pass 2: Jumbo (9000/9001) and repeat Cases 1–3
```
# Ensure VMware vSwitch and NICs support jumbo frames; set all three nodes to same MTU
Client: sudo ip link set dev <iface> mtu 9000 up   # or 9001 as required
Server: sudo ip link set dev <iface> mtu 9000 up
Router: sudo ip link set dev eth0 mtu 9000; sudo ip link set dev eth1 mtu 9000

# Repeat steps 8–10 for Cases 1–3 with jumbo MTU
```

12) Cleanup between runs
```
sudo tc qdisc del dev <iface> root || true
tc qdisc show dev <iface>
```

## Protocol Summary (High-Level)

- INFO: client -> server with file size, chunk size, count, session id; server replies IACK
- DATA: client -> server chunks with sequence numbers
- P1DN: client -> server indicates end of first pass
- NACK: server -> client with a batch of missing sequence ids (variable size)
- DONE: server -> client indicates no more missing chunks
- FIN / FACK: finalize and close

The server receives concurrently while it computes and sends NACK batches. Client
resends requested chunks until DONE, then exchanges FIN/FACK and exits.