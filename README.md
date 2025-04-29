# AnonNetwork
A theoretical secure system aiming for high-assurance anonymity and confidentiality.

**"Trust-No-Kernel" Anonymous Text Messaging System** combines a hardened hardware appliance at each endpoint with an enhanced global mix-onion network and an unlinkable messaging protocol. Even a fully compromised host kernel or a sophisticated global adversary should face significant, quantifiable difficulty in recovering plaintext, metadata, or identities.

## 1. Threat Model & Goals

1.  **Adversaries**
    * **Host-Kernel Malware**: Full control of the user's OS and kernel (mitigated by appliance I/O and host interface isolation).
    * **Global Passive/Active Network Observer (GPA)**: Can see, modify, inject, drop packets on any link; actively attempts traffic correlation.
    * **Malicious Node Operators & Collusion**: Significant fraction of mix/onion nodes may be adversarial and collude. Capable of timing attacks, selective dropping, Sybil attacks.
    * **Side-Channels**: Power, timing, RF emanations (mitigated by appliance design).
    * **Supply Chain Attacker**: Attempts to tamper with hardware/firmware before delivery (mitigated by design transparency, multi-sourcing, attestation).
    * **Censorship Infrastructure**: Attempts to block access to the network or directory information.

2.  **Security Goals**
    * **End-to-End Confidentiality**: Only sender's and recipient's appliances see plaintext.
    * **Metadata Protection (Against GPA & Collusion)**: Robustly hide who's talking to whom, when, how much, and in what pattern, resistant to sophisticated correlation.
    * **Pseudonymity & Unlinkability**: Messages cannot be tied back to real identities or to each other across contexts.
    * **Kernel-Isolation**: Host OS cannot access cryptographic keys, plaintext, or sensitive metadata.
    * **Censorship & Fingerprinting Resistance**: Employ techniques to bypass DPI and network blocks.

## 2. Endpoint Appliance ("Trust-No-Kernel" Box)

1.  **Hardware & Boot**
    * **Immutable Boot ROM** → verifies a signed microkernel (e.g., seL4) on each power-up. Rooted in **Physically Unclonable Function (PUF)** if feasible.
    * **Secure Enclave / HSM** → holds long-term keys; enforces “keys never leave.” Selected based on public security evaluations (e.g., FIPS 140-2/3, Common Criteria).
    * **Component Sourcing:** Utilize common, multi-sourced sub-components (CPU, RAM, Flash) where possible within the custom PCB. **Open Source Hardware Design** for transparency.
    * **Manufacturing & Auditing:** Partnership with multiple, auditable manufacturers. Implement random batch destructive testing for implant detection.

2.  **I/O Isolation**
    * **User I/O** (keyboard, display) plugs directly into the appliance, never the host PC.
    * **Host Interface** only a USB “network gadget” with end-to-end encrypted and authenticated tunnels (e.g., Noise Protocol); host sees opaque, fixed-size packets.

3.  **Internal Software**
    * **Minimal Microkernel (seL4)** → isolates services (network, UI driver, crypto agent, messaging agent) in separate, minimal VMs/processes.

4.  **Host-Appliance Protocol**
    * **Encapsulated Frames:** Every frame sent over USB uses a secure channel protocol (e.g., Noise\_IKpsk2\_Ed25519\_AESGCM\_SHA256) authenticated with keys derived during initial pairing/attestation. Payload is the fixed-size Onion-cell (e.g., 512 B).
    * **No Host Storage**: All message state lives in the appliance's sealed flash/RAM.

## 3. Anonymous Overlay Network

### A. Node Types & Roles

| Node Type             | Function                                                                                                     | Notes                                                                                                                                                   |
| :-------------------- | :----------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Directory Authorities** | Mutually attestable servers running verifiable logs (e.g., Trillian / Certificate Transparency-like) of node lists & metadata. Provide signed snapshots. | Geographically diverse, independent entities. Multiple sources required by client.                                                                     |
| **Entry Guard Nodes** | Stable, vetted first hops. Clients select a small set and rotate slowly.                                      | High bandwidth, uptime requirements. Subject to stricter monitoring/vetting. Helps resist GPA correlation starting at the client.                            |
| **Mix-Relay Nodes** | Apply **Poisson or Loopix-style mixing** (adds random delay from exponential distribution), batching, reordering. Padding to fixed cell size. | Stateless where possible. Can implement cryptographic Proof-of-Mixing if needed. May require moderate **stake/bond** to disincentivize misbehavior. |
| **Onion-Forwarders** | Apply onion decryption (PQ KEM based) and forward cells. Includes per-hop transport anti-replay nonces.          | Lower latency focus than Mix-Relays.                                                                                                                    |
| **Rendezvous Servers (PIR-enabled)** | Store encrypted message chunks using **Private Information Retrieval (PIR)** schemes. Query requires client computation but hides accessed ID from server. | Sharded, ephemeral storage (e.g., time-based expiry). Constant-rate polling enforced by client appliance.                                               |
| **Token Authorities (Threshold)** | Distributed entities jointly issuing blind tokens using **Threshold Blind Signatures** (e.g., 3-of-5). Optionally require small Proof-of-Work for requests. | Independent operation crucial. Compromise of < threshold number of authorities does not break the system.                                            |

### B. Path Construction

1.  **Fetch & Verify Directory**: Client appliance fetches signed directory snapshots from *multiple* Directory Authorities, cross-validates. Uses DoH/ECH or pluggable transport.
2.  **Select Guard**: Select/confirm Entry Guard node(s) from the verified list based on stability and policy.
3.  **Select Random Path**: Construct path of ≥ 5-7 intermediate nodes (Mix/Onion types) after the Guard, considering:
    * **Location/AS Diversity:** Avoid nodes in same AS/country where possible using directory metadata.
    * **Node Reputation (Optional/Experimental):** Factor in reputation scores from a decentralized system (use with caution due to manipulation risks).
    * **Type Mixing:** Ensure path includes sufficient Mix-Relay nodes for timing obfuscation.
4.  **Acquire Blind Token**: Obtain token from Threshold Token Authorities, potentially including PoW.
5.  **Circuit Setup**: Authenticate to Guard using blind token. Establish circuit using PQ KEM handshake with each hop. Keys sealed in appliance HSM. Per-hop links use authenticated transport with anti-replay.

### C. Traffic Handling

* **Mixing Strategy**: Primarily rely on **Poisson/Loopix-style mixing** in Mix-Relay nodes to provide resistance against GPA timing analysis. Define standard parameters for delay distributions.
* **Cover Traffic**:
    * **Constant Rate:** Appliances send constant-rate traffic (real or cover cells) over circuits.
    * **Loop Cover Traffic:** Appliances generate and send Sphinx "loop" packets that traverse the mixnet and return, mimicking real traffic patterns.
    * **Dummy Messages/Polls:** Appliances send occasional dummy messages to plausible (but unused) Rendezvous IDs and perform PIR polls at a constant rate, masking real activity.
* **Padding:** All cells padded to a uniform size (e.g., 512 B or 1 KiB).
* **Pluggable Transports**: As before (HTTP/3, QUIC wrappers, etc.), rotated periodically. Use ECH (Encrypted Client Hello) where possible.

## 4. Text Messaging Protocol


1.  **Message Formatting**
    * **Double-Layer Encryption:** As before (Outer session, Inner ratchet key).
    * **AEAD Protection:** Inner layer (AES-GCM) MUST include a **strictly ordered sequence number** within the authenticated additional data (AAD) for each chunk of a single message, preventing reordering/replay *within* a message by the drop server or network.
    * **Chunking & Padding:** As before (fixed size, padded).

2.  **Recipient Addressing & Rendezvous**
    * **Stealth Addresses:** As before. Master public key used for derivation.
    * **Ephemeral Rendezvous IDs:** Use **single-use** Rendezvous IDs derived from stealth address mechanism + fresh nonce for each message or small batch of chunks. Rendezvous ID = `HMAC(master_pub, one_time_dh_pub, nonce)`.
    * **PIR Interaction:** Rendezvous IDs are used as keys/tags for storage on Rendezvous Servers. Retrieval *must* use a **PIR scheme** (e.g., SealPIR, Spiral) allowing the client to fetch data for its ID(s) without revealing the ID(s) to the server.

3.  **Delivery Mechanism**
    * **Dropboxes (PIR Enabled):**
        1.  Sender encrypts chunk (with sequence # in AAD) → labels with single-use Rendezvous ID → pushes to appropriate Rendezvous Server shard (padding with dummy pushes optional).
        2.  Recipient appliance periodically polls Rendezvous Server(s) using **PIR** for its calculated set of potential current Rendezvous IDs (includes real IDs and plausible dummy IDs) at a **constant rate**.
    * **Ack & Retransmit:** Implicit via ratchet state and verified sequence numbers within messages. Missing chunks require higher-level re-request (handled by messaging agent, potentially generating new IDs).

4.  **Contact Discovery**
    * **Primary Method: Out-of-Band Exchange:** Users exchange public identifiers (`pseudonym@anonnetwork_domain`, derived from master public key) via a secure external channel (in-person QR code on appliance screen, PGP email, Signal). This is the recommended secure method.
    * **Optional/Experimental: Decentralized Intro:** Allow opt-in protocols for introductions via existing contacts, clearly warning about potential metadata linkage.
    * **NO Centralized Discovery:** Avoid searchable central directories for user identities.

5.  **Forward Secrecy & Post-Compromise Recovery**
    * Asynchronous Ratchet (X3DH + Double Ratchet). Key Erasure.

## 5. Hardware & Kernel Security


1.  **Separation**
2.  **Immutable Firmware / Verified Boot** *(Rooted in PUF/HSM)*
3.  **Remote Attestation**
    * Appliance uses HSM/Secure Enclave (potentially rooted in PUF) to generate **signed attestations** covering firmware hashes, configuration, and potentially hardware state.
    * Verifier (other appliances, directory servers) can challenge appliance for fresh attestation before communication or listing. Protocol designed to resist replay.
4.  **Side-Channel Mitigations**

## 6. Operational & Governance Model


1.  **Node Hosting**: As before (NGOs, universities etc.), emphasis on diversity. **Node operators may need to stake/bond collateral.**
2.  **Token Authority**: **Threshold-based** operation mandatory. Run by highly reputable, independent, audited entities.
3.  **Directory Authority**: Run verifiable logs; operated by independent, audited entities distinct from Token Authorities and node operators where possible.
4.  **Audits & Bug Bounties**: Crucial, ongoing process for hardware, firmware, protocols, and node software.
5.  **User Training**: As before, emphasize OPSEC.

## 7. End-to-End Flow Example


1.  **Power On**: Appliance boots, verifies microkernel via ROM/PUF, performs self-checks.
2.  **Connect & Attest**: Connects I/O. Attests its integrity (if challenged by directory/peer). Fetches verified directory info from multiple authorities. Selects/confirms Entry Guard(s).
3.  **Token Refresh**: Acquires new threshold blind signature token (potentially with PoW).
4.  **Circuit Setup**: Builds circuit via Guard through mix/onion nodes.
5.  **Compose**: User writes message on appliance keyboard/screen.
6.  **Encrypt & Send**: Appliance agent E2E encrypts (ratchet + sequence #), derives *single-use* Rendezvous ID, pushes PIR-encoded chunk(s) to Rendezvous Server via circuit. Sends loop cover traffic.
7.  **Mix-Onion Delivery**: Cells traverse network with Poisson mixing, delays, constant rate padding.
8.  **Receive**: Recipient appliance performs constant-rate **PIR polling** for potential Rendezvous IDs. Retrieves chunk(s), verifies sequence #, ratchets keys, decrypts, purges old state, displays plaintext on its own screen. Sends loop cover traffic.

### Why No One Can Spy

* **Network Observer (GPA)**: Sees fixed-size cells flowing at roughly constant rates. **Poisson mixing and loop cover traffic significantly increase the difficulty and cost of traffic correlation.** PIR obscures rendezvous lookups.
* **Compromised Host Kernel**: Sees only encrypted/authenticated traffic over USB to/from the appliance. No access to keys, plaintext, or internal state.
* **Malicious Node/Collusion**: Single nodes see little. Collusion effectiveness is reduced by Guard nodes, path selection strategies, potential node staking, and threshold trust for tokens/directory. End-to-end timing correlation remains the hardest challenge but is significantly obscured.
* **Rendezvous Server**: Cannot determine *which* message IDs are being requested due to **PIR**. Cannot read message content.
