# Networking Layers: A Practical TCP/IP Walkthrough

## 30-Second Intuition

Every network call you make — a gRPC call, a Postgres connection, a curl to an API — is four layers of envelopes stuffed inside each other: an application-layer message (HTTP, gRPC, a DNS query) gets wrapped in a TCP or UDP segment (adds ports + delivery semantics), that gets wrapped in an IP packet (adds source/destination addresses for routing), and that gets wrapped in an Ethernet frame (adds MAC addresses for the local physical hop). Each layer only trusts the layer below it to get the envelope to the next machine, and only the layer above it to make sense of what's inside. The one fact that matters operationally: **"the network" failing usually means one specific layer failed, and the symptom tells you which one** — `ping` (ICMP, network layer) succeeding while your app times out almost always means the network layer is fine and the problem is a firewall rule, a TLS handshake, or an application-layer timeout, not "the network."

Real-world TCP/IP practice runs on four layers, not OSI's seven — this doc uses the TCP/IP (DoD) model throughout and calls out the OSI mapping once, since interviewers still ask for it.

| TCP/IP layer | Rough OSI equivalent | This doc's section |
|---|---|---|
| Application | Application (7) + Presentation (6) + Session (5) | Application Layer |
| Transport | Transport (4) | Transport Layer |
| Internet | Network (3) | Network Layer |
| Link | Data Link (2) + Physical (1) | Link Layer |

OSI splits things TCP/IP doesn't bother distinguishing in practice (session management, presentation/encoding are just "something the application layer handles," e.g. TLS and serialization). Nobody runs an OSI stack; everybody runs TCP/IP and gets asked about OSI in interviews.

---

## The Full Stack: Encapsulation

Sending `GET /users/42` over HTTPS looks like this on the wire, from the application message down to what actually goes out the NIC. Sizes are approximate minimums (no TCP options, no IP options, IPv4):

```
Application data (HTTP request line + headers, e.g. "GET /users/42 HTTP/1.1...")
  ~100-800 bytes depending on headers/cookies
        │
        ▼  wrapped by TLS (if HTTPS) — adds ~20-40 bytes record header/MAC per record
        │
┌───────────────────────────────────────────────────────────────────────┐
│ TCP Header (20 bytes min)  │  TCP Payload (app data + TLS overhead)   │  ← Transport layer
│ src port, dst port, seq#,  │                                          │     adds: ports,
│ ack#, flags, window, cksum │                                          │     seq/ack, flow control
└───────────────────────────────────────────────────────────────────────┘
        │
        ▼  becomes the payload of an IP packet
┌───────────────────────────────────────────────────────────────────────┐
│ IP Header (20 bytes min)   │  IP Payload (= the TCP segment above)   │  ← Internet layer
│ src IP, dst IP, TTL, proto,│                                          │     adds: routable
│ checksum, fragment flags   │                                          │     addresses
└───────────────────────────────────────────────────────────────────────┘
        │
        ▼  becomes the payload of an Ethernet frame
┌───────────────────────────────────────────────────────────────────────┐
│ Eth Header (14 bytes)  │  IP Packet (above)  │  Eth Trailer (4 bytes)│  ← Link layer
│ dst MAC, src MAC,      │                      │  CRC32 FCS           │     adds: local
│ EtherType              │                      │                      │     hop addressing
└───────────────────────────────────────────────────────────────────────┘
        │
        ▼  bits on the wire / radio (Physical layer, folded into Link in TCP/IP)
```

Total overhead for a minimal HTTP request over plain TCP/IPv4/Ethernet: 20 (TCP) + 20 (IP) + 14+4 (Ethernet) = **58 bytes**, before a single byte of your actual request. Add TLS 1.3 record overhead (~29 bytes AEAD tag + header per record) and this is why a 40-byte "hello world" API response can turn into a ~150-byte frame on the wire — and why chatty protocols with many small messages pay a fixed per-message tax regardless of payload size.

Decapsulation on receipt is the exact mirror: the NIC strips the Ethernet header/trailer and hands the IP packet up, the kernel's IP stack strips the IP header and hands the TCP segment to the socket layer, the TCP stack strips its header and hands the byte stream to the application, which (if TLS) decrypts and finally parses HTTP. Each layer only ever looks at its own header — the Ethernet switch never parses your HTTP request, and your application never sees a MAC address.

---

## Link Layer: Ethernet, MAC Addresses, ARP

The link layer's job is delivery across a single physical hop (a switch-connected LAN segment, a point-to-point link, a Wi-Fi cell) — it has no concept of "the internet," only "the other side of this wire/radio."

**Ethernet framing**: every frame carries a 6-byte destination MAC, 6-byte source MAC, a 2-byte EtherType (e.g. `0x0800` for IPv4, `0x0806` for ARP, `0x86DD` for IPv6), then the payload, then a 4-byte CRC32 frame check sequence. MAC addresses are burned into NIC hardware (globally unique in principle, though virtualized/spoofed routinely in cloud environments) and are only meaningful on the local segment — a frame's destination MAC changes at every hop (every router rewrites it), while the IP addresses inside stay the same end-to-end. This is the single most-confused fact for people new to networking: **IP addresses are end-to-end, MAC addresses are hop-by-hop.**

**ARP (Address Resolution Protocol)**: given "I know the next-hop IP is 10.0.1.1, what's its MAC address?", a host broadcasts an ARP request ("who has 10.0.1.1? tell 10.0.1.5") on the local segment; the owner replies with its MAC, and the requester caches the mapping (Linux default ARP cache timeout is commonly ~60s-several minutes depending on `gc_stale_time`/`base_reachable_time` sysctls). This is why a freshly-replaced NIC or a failed-over load balancer can cause a brief window of packet loss even though IP routing itself is unaffected — ARP caches elsewhere on the segment are stale for one interval.

---

## Network (Internet) Layer: IP Addressing and Routing

The internet layer's job is getting a packet from any source to any destination across an arbitrary number of hops, using addresses that are meaningful end-to-end and routable.

**IPv4** addresses are 32 bits (4.3B addresses, exhausted for new allocations years ago — this is precisely why NAT exists, see below). **IPv6** addresses are 128 bits, designed with enough address space that NAT is unnecessary for host reachability (every device can have a globally routable address), plus a simplified, fixed 40-byte header (no header checksum, no fragmentation fields — routers don't fragment IPv6 packets in flight, the sender must path-MTU-discover). IPv6 adoption is real but partial: dual-stack (both IPv4 and IPv6 live side by side) is the practical reality in most production environments as of 2026, not a completed IPv4 replacement — plan for both, don't assume either is universally reachable.

**Routing** is table lookup at every hop: each router holds a forwarding table mapping destination-prefix → next-hop, matches the packet's destination IP against the longest matching prefix, decrements TTL (IPv4) / Hop Limit (IPv6) by one, and forwards. TTL reaching zero triggers an ICMP "Time Exceeded" back to the sender — this is exactly the mechanism `traceroute` exploits (send packets with TTL=1, 2, 3... and record who complains at each hop).

**Why NAT exists**: IPv4 address exhaustion made "one globally routable IP per device" untenable, so NAT (typically at a home router or cloud NAT gateway) lets many private-address devices (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 — RFC 1918) share one public IP, rewriting the source IP/port of outbound packets and reversing the rewrite on the reply using a translation table keyed by (private IP, private port) ↔ (public IP, public port). This is also why inbound connections to a NATed host need an explicit hole (port forwarding, or a rendezvous mechanism) — the NAT device has no entry in its table for a connection nobody inside initiated. See Key Gotchas for the peer-to-peer implications.

---

## Transport Layer: TCP and UDP

The transport layer is where "delivery" gets a contract: TCP promises ordered, reliable, flow-controlled, congestion-aware byte-stream delivery between two ports; UDP promises none of that — it's a fire-and-forget datagram with just enough header (8 bytes: src port, dst port, length, checksum) to get demultiplexed to the right application.

### TCP Three-Way Handshake — Worked Example

Establishing a TCP connection to a server requires one full round trip before any application data can flow:

```
Client (initial seq x=1000)                Server (initial seq y=5000)

1. SYN       seq=1000                  ──────────▶
                                                      "client wants to open
                                                       a connection, its first
                                                       byte will be seq 1001"

2. SYN-ACK   seq=5000, ack=1001        ◀──────────
                                                      "server agrees, ack=1001
                                                       means 'I've seen up
                                                       through 1000, send 1001
                                                       next'; server's own first
                                                       byte will be seq 5001"

3. ACK       seq=1001, ack=5001        ──────────▶
                                                      "client acks the server's
                                                       SYN; connection now
                                                       ESTABLISHED on both sides"
```

Initial sequence numbers (1000 and 5000 here) are randomized per connection in real stacks specifically to prevent off-path attackers from guessing/injecting into a connection (this was a real, exploited weakness in early naive implementations that incremented a global counter). After the handshake, every byte of application data is numbered starting from the next sequence number, and every ACK carries a cumulative "next byte I expect" — a single ACK for seq 1001..3000 tells the sender all 2000 bytes arrived, no per-byte acknowledgment needed.

**Cost**: this is 1 RTT before the client can even send its first HTTP request — this is exactly why connection reuse (HTTP keep-alive, connection pooling) matters, and why QUIC's ability to fold the transport and TLS handshake into fewer round trips is a real latency win (see Application Layer).

### Flow Control: The Receive Window

TCP's flow control protects the *receiver* from being overwhelmed, independent of network congestion. Every ACK also carries a "receive window" (rwnd) value — the number of bytes the receiver is currently willing to buffer. If a slow consumer's socket buffer fills up (application isn't reading fast enough), it advertises a shrinking window down to zero, and the sender must stop, periodically probing with a 1-byte "window probe" until the window reopens. This is a receiver-side control entirely separate from the sender-side congestion control below — a fast network with a slow application-layer consumer produces the exact same "sender stalls" symptom as a congested network, but for a completely different reason (check `rwnd` vs `cwnd` in a packet capture to tell them apart).

### Congestion Control: Slow Start → Congestion Avoidance → AIMD — Worked Example

Congestion control protects the *network*, using the congestion window (cwnd) as a self-imposed cap on how many unacknowledged bytes/segments the sender allows in flight, independent of what the receiver's rwnd allows (the sender uses `min(cwnd, rwnd)` as its effective limit). Classic Reno/CUBIC-style behavior, in units of MSS-sized segments, ssthresh (slow-start threshold) starting at 64:

```
RTT 1: cwnd = 1   (slow start: send 1 segment, wait for ACK)
RTT 2: cwnd = 2   (each ACK'd segment grows cwnd by 1 → doubles per RTT)
RTT 3: cwnd = 4
RTT 4: cwnd = 8
RTT 5: cwnd = 16
RTT 6: cwnd = 32
RTT 7: cwnd = 64   ← hits ssthresh, switches from slow start (exponential)
                      to congestion avoidance (linear): +1 segment per RTT
RTT 8: cwnd = 65
RTT 9: cwnd = 66
RTT 10: cwnd = 67
RTT 11: packet loss detected (e.g. triple-dup-ACK)
        → multiplicative decrease: ssthresh = cwnd/2 = 33, cwnd = 33
        (this is the "AI" additive-increase, "MD" multiplicative-decrease of AIMD)
RTT 12: cwnd = 34   (back to linear +1/RTT congestion avoidance from the new ssthresh)
RTT 13: cwnd = 35
```

This sawtooth — fast exponential ramp-up, slow linear climb, sharp halving on loss — is why a long-lived TCP connection's throughput graph looks like a sawtooth, and why a single bad network blip mid-transfer imposes a lasting throughput penalty that takes many RTTs to recover from, not just the one lost packet's retransmission cost. **Note**: this is textbook Reno/CUBIC-style AIMD; classic CUBIC uses a cubic (not strictly linear) growth function in congestion avoidance, and BBR (below) doesn't use loss as its primary congestion signal at all — this worked example is the mental model to have loaded, not literally what every stack does today.

**BBR vs CUBIC in 2026**: CUBIC (loss-based: back off on packet loss) remains the default kernel congestion control algorithm on most systems and is still what most CDNs and cloud providers ship by default, but AWS, GCP, and Azure all expose CUBIC/BBR as a per-instance or per-load-balancer choice, and BBRv3 has been upstreamed into mainline Linux and standardized in IETF draft form as of 2026. The practical pattern: BBR (model-based: infers the bottleneck bandwidth/RTT rather than reacting to loss) wins meaningfully on lossy, high-latency last-mile links (cellular, satellite, congested Wi-Fi) — reported 30-40% goodput gains over CUBIC in lossy conditions — while on clean low-loss datacenter links the two are close to a wash. If you operate a CDN or high-throughput service with a mobile-heavy client base, BBR is worth A/B testing per network class rather than assuming CUBIC's defaults are optimal.

### TCP vs UDP: Ordered Delivery and When to Choose UDP

TCP guarantees in-order, complete delivery by design: every byte is numbered, gaps are detected via sequence numbers, and the receiver either buffers out-of-order segments waiting for the gap to fill or the sender retransmits. This guarantee has a cost — a single lost segment blocks delivery of every segment after it to the application, even ones that already arrived, until the gap is retransmitted and filled (transport-level head-of-line blocking; see HTTP/3 below for why this specifically matters at the application layer). UDP guarantees nothing: no ordering, no retransmission, no flow/congestion control — a datagram either arrives or it doesn't, and a later datagram can arrive before an earlier one with no framework noticing unless the application adds its own sequence numbers.

UDP is the right choice exactly when: (1) a late/lost update is worthless anyway (an old video frame, an old game-state tick — TCP's retransmit-and-block behavior would deliver a *stale* frame late instead of a fresh one on time), or (2) the exchange is a single small request/response where connection setup cost dominates (DNS queries — one datagram out, one datagram back, and the application layer handles retry/timeout itself). TCP is the right choice whenever completeness matters more than freshness: file transfer, database connections, API calls, anything where a dropped byte silently corrupting the result is unacceptable and retransmission cost is a fair trade for correctness.

---

## Application Layer: HTTP, TLS, DNS

This is the layer your code actually talks to. Everything below it is (ideally) invisible to application logic — until it isn't, which is most of the Key Gotchas below.

### HTTP/1.1 vs HTTP/2 vs HTTP/3 — Head-of-Line Blocking Worked Example

Concretely, a page/API client needs to fetch 4 resources (A, B, C, D) from one origin, where resource B is slow (e.g. a slow database-backed endpoint) and the rest are fast.

**HTTP/1.1** (one request in flight per TCP connection at a time, without pipelining — the practical deployment reality since HTTP pipelining was never widely enabled): browsers open multiple parallel TCP connections (historically 6 per origin) to work around this, but on any single connection, request order is strict — if B is queued behind or ahead of A/C/D on the same connection, B's slowness blocks everything queued behind it on *that* connection. Latency shape: fast requests sharing a connection with a slow one visibly stall, then release in a burst once the slow one finally completes.

**HTTP/2** (single TCP connection, multiplexed streams): A, B, C, D are interleaved as independently-framed streams over the *same* TCP connection — C and D's response frames can be delivered and processed while B is still pending, no per-request queuing at the application layer. Latency shape: fast requests complete on schedule regardless of B; this is HTTP/2's whole value proposition over HTTP/1.1. But the streams still ride one TCP connection, so if a single *packet* is lost, TCP's in-order delivery (above) blocks *all* streams' data behind the retransmit — HTTP/2 fixed application-level head-of-line blocking but inherited transport-level head-of-line blocking from TCP.

**HTTP/3 (QUIC)**: QUIC replaces TCP with a UDP-based transport that implements its own per-stream reliability — each stream has independent sequencing and loss recovery, so a lost packet carrying stream B's data blocks only stream B; streams A, C, D continue delivering to the application even while B's lost packet is being retransmitted. Latency shape: fast requests are insulated from an unrelated stream's packet loss, not just an unrelated stream's *slowness* — this is the specific transport-level HOL-blocking problem QUIC solves that HTTP/2-over-TCP could not. QUIC also folds the transport handshake and TLS 1.3 handshake into a single round trip (1-RTT, or 0-RTT on session resumption — see TLS below), shaving a full RTT off connection setup versus TCP+TLS's separate handshakes.

**2026 adoption reality**: HTTP/3 support is essentially universal at the CDN and browser level (Cloudflare, Fastly, CloudFront, Chrome/Firefox/Safari/Edge all support it), but actual traffic share is still a minority of real page loads — commonly cited figures put it in the roughly 25-40% range of measured traffic depending on methodology (W3Techs vs Cloudflare-edge vs page-load telemetry disagree on exact numbers), with mobile/lossy-network markets adopting fastest since that's where QUIC's per-stream loss recovery pays off most. Treat HTTP/3 as broadly *available* but not yet the default majority protocol for arbitrary internal/backend traffic — many internal service-to-service paths and older infrastructure remain HTTP/1.1 or HTTP/2.

### TLS Handshake Basics

TLS 1.3 (RFC 8446) is the current baseline as of 2026: a full handshake completes in 1 RTT (client sends its key share and cipher preferences in the same flight as "hello"; server responds with its own key share, certificate, and a "Finished" message in one flight back), versus TLS 1.2's 2-RTT handshake. Session resumption (0-RTT, using a pre-shared key from a prior connection) lets a returning client send encrypted application data in its very first flight, at the cost of losing forward-secrecy-style anti-replay guarantees for that specific early data (0-RTT data is technically replayable by a network attacker, so it's normally restricted to idempotent requests). TLS 1.2 is not formally deprecated as of 2026 and remains the PCI-DSS-mandated floor, but TLS 1.3-preferred-with-1.2-fallback is the standard production posture; TLS 1.0/1.1 are formally deprecated (RFC 8996, 2021) and refused outright by any current-generation stack.

Every TLS handshake RTT is pure added latency before the first byte of application data can be exchanged — this is why TLS 1.3's 1-RTT (and 0-RTT resumption) matters disproportionately for latency-sensitive services: a cold HTTPS connection over TCP pays TCP's 1-RTT handshake *plus* TLS's 1-RTT handshake (2 RTTs before any HTTP request goes out), while QUIC (HTTP/3) collapses transport + TLS setup into the same 1-RTT, and session resumption can bring a *returning* client's connection setup to effectively 0 extra RTTs.

### DNS Resolution — Worked Example

Resolving `api.example.com` before any TCP/TLS handshake can even begin: the client's stub resolver (usually pointed at a recursive resolver — an ISP's, or 1.1.1.1/8.8.8.8) checks its cache, then queries the recursive resolver, which (on a cache miss) walks the delegation chain:

```
1. Recursive resolver ──▶ Root server:        "who handles .com?"
                       ◀── Root server:        "ask these .com TLD servers"
2. Recursive resolver ──▶ .com TLD server:     "who handles example.com?"
                       ◀── TLD server:          "ask ns1.example.com (with its IP, via glue record)"
3. Recursive resolver ──▶ example.com auth NS:  "what's the A/AAAA record for api.example.com?"
                       ◀── Auth NS:             "93.184.216.34, TTL=300"
4. Recursive resolver ──▶ Client:               "93.184.216.34" (cached locally for TTL seconds)
```

Each of those hops is itself a UDP (falling back to TCP for large responses) round trip — a fully cold DNS lookup with no caching at any level can add several RTTs of pure latency before the TCP handshake to the actual API server even starts, which is exactly why DNS TTL tuning, `Happy Eyeballs`/pre-resolution, and DNS caching layers (in the OS, in the application, in a sidecar resolver) matter for tail latency in production services. A DNS query is UDP precisely because it's the "single small request/response where a lost query is cheap to just retry" case described in TCP vs UDP above.

For how network-level latency and partitions between machines factor into distributed coordination protocols (as opposed to single-hop wire delay), see [distributed-systems/clocks-and-ordering.md](../distributed-systems/clocks-and-ordering.md) — that doc covers what happens *above* this layer when clocks and message delivery across these same links can't be trusted to be synchronous or ordered.

---

## TCP vs UDP — Comparative Summary

| | TCP | UDP |
|---|---|---|
| Ordering | Guaranteed (sequence numbers) | Not guaranteed |
| Reliability | Guaranteed (retransmission) | Best-effort, no retransmission |
| Flow/congestion control | Built in (rwnd, cwnd) | None (application must add its own if needed) |
| Connection setup cost | 1 RTT handshake before data | None — first datagram can carry data |
| Head-of-line blocking | Yes, at the byte-stream level | N/A — no ordering to block on |
| Use when | File transfer, API calls/RPC, database connections — completeness matters | Video/audio streaming, gaming, DNS queries — freshness matters more than completeness, or round-trip setup cost dominates a tiny exchange |

## HTTP/1.1 vs HTTP/2 vs HTTP/3 — Comparative Summary

| | HTTP/1.1 | HTTP/2 | HTTP/3 (QUIC) |
|---|---|---|---|
| Transport | TCP | TCP | UDP (QUIC) |
| Multiplexing | No (parallel connections as workaround) | Yes, single connection | Yes, single connection |
| App-layer HOL blocking | Yes | No | No |
| Transport-layer HOL blocking | N/A (one request at a time per connection) | Yes (inherited from TCP) | No (per-stream loss recovery) |
| Handshake cost (cold) | TCP (1 RTT) + TLS (1 RTT, TLS 1.3) | Same as HTTP/1.1 | Combined into ~1 RTT (0-RTT on resumption) |
| 2026 real-world status | Still common for internal/legacy traffic | Widely deployed default at most CDNs/origins | Broadly available at CDN/browser level; a growing minority of measured traffic, strongest on mobile/lossy networks |

---

## Key Gotchas

- **TCP "reliable" does not mean "fast"**: a single lost packet doesn't just cost one retransmission — the receiver holds everything after the gap unusable until it's filled, and the retransmission timeout (RTO) that triggers a resend (when no duplicate ACKs signal loss directly) is typically much larger than the RTT itself, so a lost packet in a low-traffic period can stall a connection for hundreds of milliseconds to seconds. This is the dominant cause of tail latency (p99/p999) in TCP-based services even when average throughput looks fine — one unlucky retransmit ruins one request's latency without moving the average at all.
- **NAT breaks the "any host can connect to any host" assumption**: a NATed host has no public reachability for unsolicited inbound connections, which is why peer-to-peer protocols (WebRTC, some VoIP, certain P2P file-sharing) need NAT traversal techniques (STUN to discover the public mapping, TURN to relay when direct traversal fails, ICE to negotiate) — and why "it works on my home network but not for both peers behind different NATs" is a NAT-traversal problem, not an application bug.
- **MTU mismatches cause silent packet drops, not errors**: if a packet exceeds the path's smallest MTU (commonly 1500 bytes on Ethernet, less over VPN/tunnel overlays), IPv4 routers can fragment it (or drop it and send ICMP "fragmentation needed" if the Don't-Fragment bit is set) — but if a firewall along the path blocks that ICMP message (common misconfiguration), the sender never learns to shrink its packets and connections mysteriously hang exactly at the point payloads get large (TLS handshake certificates, larger HTTP responses) while small requests work fine. This "PMTUD black hole" is a classic, hard-to-diagnose production issue — suspect it when small requests succeed but larger ones hang without a clean RST or error.
- **"Ping works but the app doesn't" is (almost) never a network-layer problem**: ICMP echo (ping) only proves IP-layer reachability — it says nothing about whether the target port is open, whether a firewall allows that specific port/protocol, whether TLS negotiation succeeds, or whether the application itself is healthy. Reflexively blaming "the network" here wastes debugging time; check firewall rules, security groups, and application health/logs first — the network layer already proved itself functional.
- **TLS handshake RTT cost compounds with DNS and TCP setup**: a cold connection to a new host can be DNS (1+ RTT) + TCP handshake (1 RTT) + TLS handshake (1 RTT for TLS 1.3, 2 for TLS 1.2) before a single byte of the actual request is sent — for latency-sensitive services, this is why connection pooling/keep-alive, DNS caching, TLS session resumption, and (where available) HTTP/3's combined transport+TLS handshake are not micro-optimizations but the difference between a request that feels instant and one that visibly lags, especially over higher-RTT mobile or cross-region links.
