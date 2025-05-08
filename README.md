# AnonNetwork

A theoretical secure system aiming for high-assurance anonymity and confidentiality.

This constant-rate, multi-entry mix network is designed to provide strong anonymity and security against traffic analysis, through built-in multi-packet mixing, uniform packet sizes, bidirectional tunnels, and adaptive cover traffic—all without relying on directories or bootstrap nodes.

## 1. Threat Model & Goals

1. **Adversaries**
   * **Kernel-Level Malware**: Full control of the user's OS and kernel (mitigated by appliance I/O isolation and host interface separation).
   * **Global Passive/Active Observer (GPA)**: Can monitor, modify, inject, or drop packets on any link; attempts sophisticated traffic correlation.
   * **Malicious Mix Operators & Collusion**: All nodes may be compromised; capable of timing, dropping, Sybil, and replay attacks.
   * **Side-Channels**: Power, timing, electromagnetic emanations.
   * **Supply-Chain Attacks**: Hardware/firmware tampering (mitigated via design transparency, multi-sourcing, and attestation).
   * **Censorship Systems**: Deep packet inspection (DPI) and network blocking.

2. **Security Goals**
   * **End-to-End Confidentiality**: Only sender’s and receiver’s appliances decrypt plaintext.
   * **Metadata Protection**: Hide participant identities, message volumes, and timing patterns.
   * **Pseudonymity & Unlinkability**: Prevent linking multiple messages or sessions to one user.
   * **Host Isolation**: User OS cannot access cryptographic keys or metadata.
   * **Censorship Resistance**: Bypass DPI and blocklisting via pluggable transports.

## 2. Secure Peer Discovery without Bootstrap

To eliminate the risk of a compromised bootstrap node, AnonNetwork uses a fully decentralized, multi-layered peer discovery system that ensures no single point of failure. Each node independently discovers the network via a combination of cryptographically verifiable techniques:

1. **Verifiable Random Walks (VRWs)**
   * Nodes initiate random walks across the known network graph, collecting node descriptors at each hop.
   * Each walk includes cryptographic proofs to detect tampering or censorship.

2. **Decentralized Hash Sampling**
   * Nodes derive sampling targets from hash-based computations over time-variant values, ensuring uniform global coverage without centralized seeding.

3. **Proof-of-Uptime Directory Fragments**
   * Network descriptors are gossiped and validated via uptime-signed statements from diverse peers.
   * The use of threshold cryptography prevents forged inclusion of malicious descriptors.

4. **Chaotic Probing & Distributed Hash Crawls**
   * Each node occasionally probes random IP segments based on distributed hash ranges.
   * Results are vetted for liveliness, cryptographic validity, and consensus inclusion.

5. **History-Based Pruning**
   * Nodes track descriptor history including reliability, uptime, and correct mixing.
   * Descriptors with poor reputations are down-weighted and excluded over time.

This discovery system is resistant to Sybil, eclipse, and partitioning attacks, offering full network visibility to honest nodes even under adversarial conditions—without any bootstrap or directory server.

## 3. Core Design Features

* **Entry**
   * **Multi-Entry Splitting**: Each message is split into uniform 2 KB packets and sent over three independent entry mixes simultaneously.
   * **Merging**: Before entering the first mix node, three 2 KB packets are merged into a 6 KB super-packet (in each entry node, with other messages or noise).
* **Synchronous Constant-Rate**: Nodes emit exactly 1 6KB super-packet per second per tunnel (to possibly 30-80 tunnels max), replacing dummies with real packets as needed.
* **Uniform Packet Size & Padding**: Fixed 2 KB packets (6KB super-packet) prevent size-based correlation.
* **Mesh Mixing & Switching**: At each node, batches of incoming 6 KB super-packets (containing real and dummy 2 KB packets) are pooled, unpacked into 2 KB packets, shuffled, and re-packed into new 6 KB super-packets, which are then sent along random outgoing paths.
* **Bidirectional, Rotating Tunnels**: No dedicated exit nodes; middle nodes decrypt one layer, mix, then forward—also acting as egress.
* **Cover Traffic & Loop Backs**: Encrypted dummies and self-addressed loops detect drops and maintain constant flow.
* **Re-Shufling**: At each node, the pool of 2KB packages (super-packet) is re-shuffled with new packages, creating a new super-packet.

## 4. Network Architecture & Packet Workflow

1. **Segmentation & Padding**: Sender appliance divides the message into chunks of less than 2 KB, encrypts each, and pads until it reached 2 KB.
2. **Peer Sampling**: Client uses secure decentralized discovery to sample a vetted pool of mix nodes and selects three entry mixes via weighted random selection.
3. **Packet Injection**: Every second, each tunnel sends exactly one packet—real data if available; otherwise, an encrypted dummy.
4. **Mixing Mesh**:
   * **Batch Collection**: A node collects all incoming packets across its tunnels at the 1 s interval.
   * **Random Delay & Reordering**: Each batch is collectively delayed by a Poisson-distributed offset and then shuffled.
   * **Switching**: Packets are reassigned to outgoing tunnels at random, breaking per-stream correlations.
5. **Layered Encryption**: Each packet contains multiple encryption layers, peeled one layer per hop.
6. **Bidirectional Replies**: Recipient constructs fresh inbound tunnels using the same peer-discovery process; response packets follow the same mixed fashion back.

## 5. Performance Evaluation

| Metric     | Value                                                           |
| ---------- | --------------------------------------------------------------- |
| Throughput | ≈0.2-0.5 Mbps effective (6 KB/s cover overhead)                 |
| Latency    | 2-20 s end-to-end (3-5 hops, 1 s base interval + Poisson delay) |
| Bandwidth  | ≥6 KB/s per tunnel for cover + real traffic                     |
| Storage    | Buffer for 1 s batches per node                                 |

## 6. Security & Anonymity Measures

* **Batch Mixing**: Pools multiple streams, preventing linkability of individual packets.
* **Switching**: Random reassignment of packets to outgoing tunnels.
* **Cover Traffic**: Constant emission of dummy packets; loop-back dummies verify integrity.
* **Layered (Onion/Garlic) Encryption**: Ensures confidentiality and integrity per hop.
* **Path Rotation**: New tunnels per message direction thwart end-to-end path tracing.
* **Adaptive Delays**: Poisson-based random delays counter timing analysis.
* **Threshold Flushing**: Optionally wait for k packets before release to enforce minimum anonymity sets.
* **Decentralized Discovery**: Peer discovery avoids bootstrap and ensures adversarial resilience.
* **Reputation System**: Local scoring limits influence of Sybil or malicious nodes.

## 7. Anonymity & Security Assessment

| Aspect                       | Score (1-10) | Rationale                                                             |
| ---------------------------- | ------------ | --------------------------------------------------------------------- |
| Anonymity                    | 9            | Multi-entry, batch mixing, switching, cover traffic.                  |
| Confidentiality              | 9            | Layered encryption; only endpoints see plaintext.                     |
| Resistance to GPA            | 8            | Constant-rate, adaptive delays, threshold mixes mitigate timing.      |
| Resistance to Node Collusion | 9            | Decentralized discovery eliminates bootstrap; strong local filtering. |
| Operational Complexity       | 4            | High: discovery logic, buffer sync, and p er-hop crypto.               |

## 8. Trade-offs & Limitations

* **Latency vs. Anonymity**: Higher mixing intervals and batch sizes increase anonymity but also delay.
* **Bandwidth vs. Security**: Constant cover traffic ensures uniform flow, at the cost of extra bandwidth.
* **Scalability**: Large-scale decentralized discovery and validation can tax node resources.
* **Sybil Resistance**: No central trust anchor; reliant on probabilistic trust and exclusion mechanisms.

## 9. Potential Enhancements

* **Verifiable Shuffles**: Cryptographically prove correct mixing without revealing batch contents.
* **Differential Privacy for Timing**: Calibrate delays to meet formal privacy budgets.
* **Hardware Acceleration**: Offload encryption and mixing to dedicated hardware.
* **Secure Peer Exchange Ledger**: Gossip descriptors into multi-ledger systems to increase fault tolerance.

## 10. Conclusion

AnonNetwork eliminates central discovery points using a secure, fully decentralized mechanism. With multi-entry mixing, cover traffic, switching, and batch operations, it resists powerful adversaries while offering practical, high-assurance anonymity for secure communication.
