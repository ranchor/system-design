# OSI Model

The OSI (Open Systems Interconnection) model is a conceptual model that describes how networked systems communicate. It splits a communication system into **seven abstract layers**. Each layer serves the layer above it and is served by the layer below.

It acts as a shared vocabulary for networking. The internet actually runs on the simpler **TCP/IP model**, but engineers still describe things in OSI terms: "L4 load balancer", "L7 firewall", "L2 switch".

## Why does the OSI model matter?

- **Common terminology** for design discussions, docs and vendor products.
- **Troubleshooting:** isolate a problem layer by layer (cable → link → IP → port → TLS → HTTP).
- **Security:** threats and defenses map to layers (e.g. SYN flood at L4, SQL injection at L7).
- **Interoperability:** vendors build products that work together by conforming to a layer's interface.
- **Separation of concerns:** a complex function is broken into simpler, replaceable pieces.

## Layers

The seven layers, from top to bottom:

![osi-model](https://raw.githubusercontent.com/karanpratapsingh/portfolio/master/public/static/courses/system-design/chapter-I/osi-model/osi-model.png)

**Mnemonic** (top → bottom, 7 → 1): **A**ll **P**eople **S**eem **T**o **N**eed **D**ata **P**rocessing.
(Bottom → top: **P**lease **D**o **N**ot **T**hrow **S**ausage **P**izza **A**way.)

### Quick reference

| #  | Layer        | Data unit (PDU)  | Responsibility                                        | Protocols / examples                         | Devices / components                       |
|----|--------------|------------------|-------------------------------------------------------|----------------------------------------------|--------------------------------------------|
| 7  | Application  | Data / message   | Network services for applications                     | HTTP/HTTPS, gRPC, DNS, SMTP, FTP, SSH, WebSocket, MQTT | L7 load balancer, API gateway, WAF, reverse proxy |
| 6  | Presentation | Data             | Translation, encoding, encryption, compression        | TLS/SSL\*, JSON, XML, Protobuf, JPEG, gzip, ASCII/UTF-8 | —                                          |
| 5  | Session      | Data             | Open, maintain and close sessions; checkpoints        | RPC, NetBIOS, SOCKS, TLS session resumption  | —                                          |
| 4  | Transport    | Segment (TCP) / Datagram (UDP) | End-to-end delivery, ports, reliability, flow & congestion control | TCP, UDP, QUIC\*                    | L4 load balancer, stateful firewall, NAT\* |
| 3  | Network      | Packet           | Logical addressing (IP) and routing between networks  | IPv4, IPv6, ICMP, IPsec, BGP, OSPF           | Router, L3 switch                          |
| 2  | Data Link    | Frame            | Node-to-node delivery on the same network, MAC addressing, error detection | Ethernet, Wi-Fi (802.11), ARP, VLAN (802.1Q), PPP | Switch, bridge, NIC                        |
| 1  | Physical     | Bit              | Transmit raw bits over a medium                       | Copper, fiber, radio, DSL, USB, Bluetooth PHY | Hub, repeater, cable, modem                |

\* Some protocols don't fit cleanly. TLS sits between L4 and L7 (usually called L6 or "L4.5"). QUIC is a transport protocol built on UDP that also includes TLS. NAT rewrites L3 and L4 headers.

### Application (Layer 7)

The only layer that directly interacts with data from the user. Software such as web browsers and email clients relies on it to start communication. The client applications themselves are **not** part of this layer. The layer is the set of protocols and data handling those applications rely on, such as HTTP, SMTP and DNS.

### Presentation (Layer 6)

Also called the **translation layer**. It formats data so the receiver can understand it. Its jobs are **translation** (character encoding, serialization such as JSON or Protobuf), **encryption/decryption** (TLS) and **compression**.

### Session (Layer 5)

Opens, maintains and closes communication between two devices. The time between opening and closing is the **session**. This layer keeps the session open long enough to transfer all the data, then closes it so resources aren't wasted. It also synchronizes transfer with **checkpoints**, so a long transfer can resume from the last checkpoint after a failure.

### Transport (Layer 4)

Handles **end-to-end communication between processes**, identified by **ports**. It breaks session-layer data into **segments** before handing them to the network layer and reassembles them on the receiver. It also provides **flow control** (don't overwhelm the receiver) and **error control** (detect loss and retransmit).

| TCP                                               | UDP                                               |
|---------------------------------------------------|---------------------------------------------------|
| Connection-oriented (3-way handshake: SYN, SYN-ACK, ACK) | Connectionless                                    |
| Reliable, ordered delivery with retransmissions   | Best effort; packets may be lost, duplicated or reordered |
| Flow and congestion control                       | None (the application handles it if needed)      |
| Higher latency and a heavier header (20+ bytes)   | Low latency, 8-byte header                       |
| HTTP/1.1, HTTP/2, SMTP, SSH, databases            | DNS, video/voice streaming, gaming, QUIC/HTTP/3   |

### Network (Layer 3)

Moves data **between different networks**. It splits transport segments into **packets**, adds logical (IP) addresses, and finds the best path to the destination. This is called **routing**. If both devices are on the same network, routing isn't needed; delivery happens at layer 2.

### Data Link (Layer 2)

Like the network layer, but for transfer between two devices **on the same network**. It wraps packets into **frames** with source and destination **MAC addresses**, and detects errors with a checksum (CRC). It is commonly split into two sublayers: **LLC** (logical link control) and **MAC** (media access control). **ARP** maps IP addresses to MAC addresses.

### Physical (Layer 1)

The physical equipment involved in the transfer, such as cables, fiber, radio, hubs and connectors. Here data becomes a **bit stream** of 1s and 0s. Both devices must agree on signaling (voltage, light pulses, frequency, bit rate) so that 1s can be told apart from 0s.

## Encapsulation and Decapsulation

When sending, each layer adds its own **header** (and the data link layer adds a **trailer**) to the data from the layer above. This is **encapsulation**. The receiver removes the headers in reverse order, which is **decapsulation**.

```
L7-5  [            Data            ]
L4    [TCP hdr][      Data         ]                  -> Segment
L3    [IP hdr][TCP hdr][   Data    ]                  -> Packet
L2    [Eth hdr][IP hdr][TCP hdr][Data][Eth trailer]   -> Frame
L1    0101100111010...                                -> Bits
```

Typical overhead: Ethernet 14 B header + 4 B FCS, IPv4 20 B, TCP 20 B, UDP 8 B. The Ethernet **MTU is 1500 B**, so the max TCP payload (MSS) is about **1460 B**.

## OSI vs TCP/IP Model

| OSI (7 layers)                        | TCP/IP (4 layers)       | Examples                    |
|---------------------------------------|-------------------------|-----------------------------|
| Application, Presentation, Session    | Application             | HTTP, DNS, TLS, gRPC        |
| Transport                             | Transport               | TCP, UDP, QUIC              |
| Network                               | Internet                | IP, ICMP                    |
| Data Link, Physical                   | Network Access / Link   | Ethernet, Wi-Fi             |

OSI is a reference model, defined before its protocols and largely theoretical. TCP/IP is the practical model the internet runs on, where the protocols came first. Some descriptions use a 5-layer hybrid that keeps Physical and Data Link separate.

## What happens when you type a URL (walking the layers)

1. **L7:** The browser parses the URL. **DNS** (usually over UDP:53) resolves the hostname to an IP, then the browser builds an HTTP request.
2. **L6:** **TLS** handshake, then the request is encrypted and possibly compressed.
3. **L5:** The session is set up (TLS session or HTTP keep-alive or HTTP/2 connection reuse).
4. **L4:** TCP 3-way handshake to port 443. Data is split into segments.
5. **L3:** IP packets are routed hop by hop across the internet to the server's IP.
6. **L2:** On each hop, frames are addressed to the next hop's MAC (found by ARP).
7. **L1:** Bits travel over copper, fiber or radio.

The server decapsulates from L1 up to L7 and sends the response back down the stack.

## Relevance to System Design

| Concept                    | Layer | Why it matters                                                                                  |
|----------------------------|-------|-------------------------------------------------------------------------------------------------|
| **L4 load balancer**       | 4     | Routes by IP and port (TCP/UDP) without reading content. Very fast with low overhead and can do TLS passthrough. Examples: AWS NLB, HAProxy in TCP mode, LVS. |
| **L7 load balancer**       | 7     | Reads HTTP (path, headers, cookies) to route, terminate TLS, cache, rewrite, rate limit or A/B test. More CPU per request. Examples: AWS ALB, Nginx, Envoy. |
| **TLS termination**        | 6/7   | Offloading encryption at the LB or edge saves backend CPU and enables L7 routing.              |
| **CDN / reverse proxy**    | 7     | Caches HTTP responses close to users.                                                          |
| **Firewalls**              | 3/4/7 | Packet filter (IP and port) vs. stateful (connection tracking) vs. WAF (inspects HTTP payload). |
| **DDoS types**             | 3/4/7 | Volumetric or UDP floods (L3/L4), SYN floods (L4), HTTP floods and slowloris (L7).             |
| **VPC / subnets / routing**| 3     | CIDR blocks, route tables and security groups operate on IPs and ports.                        |
| **VLANs / switches**       | 2     | Segment traffic within a datacenter.                                                           |
| **HTTP/3 (QUIC)**          | 4/7   | Runs over UDP to avoid TCP head-of-line blocking and makes connection setup faster (0-RTT or 1-RTT). |

### Interview quick hits
- **"What layer is a load balancer?"** It depends: an L4 LB works on TCP/UDP and an L7 LB works on HTTP. Explain the trade-off (speed vs. content-aware routing).
- **Switch vs. router:** a switch forwards **frames** by MAC within a network (L2). A router forwards **packets** by IP between networks (L3).
- **Hub vs. switch:** a hub (L1) repeats bits to every port. A switch (L2) learns MACs and forwards selectively.
- **Ports live at L4**, IP addresses at L3 and MAC addresses at L2.
- **Ping/ICMP** works at L3, so a successful ping proves L1–L3 work but says nothing about whether port 443 is open (L4) or the app is healthy (L7).

# References
- https://www.cloudflare.com/learning/ddos/glossary/open-systems-interconnection-model-osi/
- https://github.com/karanpratapsingh/system-design#osi-model
- https://en.wikipedia.org/wiki/OSI_model
- https://www.imperva.com/learn/application-security/osi-model/
