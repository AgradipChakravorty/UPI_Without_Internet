# UPI Offline Mesh Payment Engine

[![Java](https://img.shields.io/badge/Java-17%2B-orange.svg)](https://www.oracle.com/java/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.x-green.svg)](https://spring.io/projects/spring-boot)
[![Database](https://img.shields.io/badge/Database-H2%20In--Memory-blue.svg)](https://www.h2database.com/)

A Spring Boot backend engineered to process offline UPI payments routed through a Bluetooth-style peer-to-peer mesh network. Designed for zero-connectivity environments (such as basements or remote locations), the system encrypts local payment intents on the sender's device and gossips the payload across neighboring peer nodes until reaching an internet-connected bridge device, which uploads the packet to the central backend for atomic settlement.

Includes an in-memory software simulator of the mesh network and an interactive web dashboard to demonstrate end-to-end multi-hop routing, cryptographic verification, and idempotent ledger processing.

---

## Technical Architecture

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                         SENDER PHONE (offline)                          │
│  PaymentInstruction { sender, receiver, amount, pinHash, nonce, time }  │
│              │                                                          │
│              ▼ encrypt with server's RSA public key                     │
│   MeshPacket { packetId, ttl, createdAt, ciphertext }                   │
└──────────────────────────────────────┬──────────────────────────────────┘
                                       │ Bluetooth gossip
                                       ▼
        ┌─────────┐  hop   ┌─────────┐  hop   ┌─────────┐
        │stranger1│ ─────▶ │stranger2│ ─────▶ │ bridge  │ ◀── walks outside
        └─────────┘        └─────────┘        └────┬────┘     gets 4G
                                                   │
                                                   ▼ HTTPS POST
┌─────────────────────────────────────────────────────────────────────────┐
│                     SPRING BOOT BACKEND                                 │
│                                                                         │
│  /api/bridge/ingest                                                     │
│       │                                                                 │
│       ▼                                                                 │
│  [1] hash ciphertext (SHA-256)                                          │
│       │                                                                 │
│       ▼                                                                 │
│  [2] IdempotencyService.claim(hash)  ◀── atomic putIfAbsent (≈ Redis    │
│       │                                  SETNX). Duplicates rejected    │
│       │                                  here, before any work.         │
│       ▼                                                                 │
│  [3] HybridCryptoService.decrypt(ciphertext)                            │
│       │       (RSA-OAEP unwraps AES key, AES-GCM decrypts payload       │
│       │        AND verifies the auth tag — tampering = exception)       │
│       ▼                                                                 │
│  [4] Freshness check: signedAt within last 24h                          │
│       │                                                                 │
│       ▼                                                                 │
│  [5] SettlementService.settle()                                         │
│       @Transactional: debit sender, credit receiver, write ledger       │
│       @Version on Account = optimistic locking (defense in depth)       │
└─────────────────────────────────────────────────────────────────────────┘
```
---

## What This Project Proves

* **Untrusted Intermediary Security:** Payments traverse untrusted peer nodes without exposing sensitive data or allowing payload tampering (Hybrid RSA-OAEP + AES-GCM).
* **Backend Idempotency:** Simultaneous packet uploads across multiple bridge nodes settle exactly once (Atomic compare-and-set on SHA-256 ciphertext hashes).
* **Replay & Tamper Resistance:** Replayed or altered packets are caught and dropped before touching the database ledger.

---

## Core Engineering Challenges Solved

### **Problem 1: Untrusted Intermediaries (Data Privacy & Integrity)**
Peer devices carry encrypted transactions. To prevent reading or altering payloads:
* **Hybrid Cryptography:** Payment data is encrypted using AES-256-GCM. The ephemeral AES key is wrapped using the server's RSA-2048 public key.
* **Authenticated Encryption:** AES-GCM attaches a 16-byte authentication tag. Any single-bit modification by an intermediate node invalidates the tag, triggering immediate rejection.

### **Problem 2: The Duplicate-Storm (Preventing Double-Spending)**
When multiple bridge devices hold identical packets and reconnect to 4G simultaneously:
* **Atomic SHA-256 Deduplication:** Computes `SHA-256(ciphertext)` before running expensive RSA decryption operations.
* **ConcurrentHashMap `putIfAbsent` (SETNX Pattern):** Guarantees that only the first thread claiming a hash proceeds to settlement; concurrent duplicates return `DUPLICATE_DROPPED`.
* **Database Defense-in-Depth:** A `UNIQUE` index constraint on `transactions.packet_hash` prevents duplicate ledger entries.

### **Problem 3: Replay Attacks**
* **Freshness Constraint:** Payloads enforce a `signedAt` timestamp check; packets older than 24 hours are rejected.
* **Unique Nonce Generation:** Distinct UUID nonces guarantee that separate transfers generate unique ciphertexts and distinct hashes.

---

## Setup & Running Locally

### **Prerequisites**
* JDK 17 or newer

### **1. Launch Application**
Execute the Maven wrapper from the root directory:
```bash
./mvnw clean spring-boot:run
```
### **2. Open Interactive Dashboard**

Navigate to the web interface in your browser:

```text
http://localhost:8080/dashboard.html:
```
##  Interactive Simulation Workflow 

Inject Payment: Select a sender (alice@demo) and receiver (bob@demo), specify an amount, and click Inject into Mesh to generate an encrypted packet.

Gossip Propagation: Click Run Gossip Round 2–3 times to simulate peer-to-peer packet transmission across offline devices.

Bridge Settlement: Click Bridges Upload to Backend to simulate a 4G connection uploading buffered packets to the server for ledger updates.

Idempotency Verification: Click Bridges Upload to Backend again to verify that duplicate packet submissions are rejected without double-debiting balances.

## Test Suite Execution

Run multi-threaded concurrency and cryptographic tests:

```bash
./mvnw test
```
* **`encryptDecryptRoundTrip`**: Validates end-to-end hybrid cryptographic operations.
* **`tamperedCiphertextIsRejected`**: Verifies that tampered payloads fail GCM authentication tag verification.
* **`singlePacketDeliveredByThreeBridgesSettlesExactlyOnce`**: Simulates 3 concurrent threads uploading an identical packet simultaneously to verify atomic deduplication.

## Production Architecture Mapping

| Component | Architecture Choice | Production Equivalent |
| :--- | :--- | :--- |
| **Database** | H2 In-Memory Database | PostgreSQL / MySQL with Read-Replicas |
| **Idempotency** | ConcurrentHashMap (`putIfAbsent`) | Distributed Redis Cluster (`SET NX EX`) |
| **Key Management** | Server Key Pair Generation | Hardware Security Module (HSM) / AWS KMS |
| **Transport Layer** | Simulated Gossip Mesh | Native Android/iOS BLE GATT & Wi-Fi Direct |

## Technical Considerations

* **Deferred Settlement Risk:** Until a packet reaches the backend, an offline transfer represents ledger intent. In commercial systems (e.g., UPI Lite), offline double-spending is mitigated using hardware-backed, pre-funded local wallets.
* **BLE Constraints:** Background BLE broad-scanning is subject to OS-level power optimizations on iOS and Android.

---

## Database Inspection Console

Access live database tables during execution:
* **URL:** `http://localhost:8080/h2-console`
* **JDBC URL:** `jdbc:h2:mem:upimesh`
* **Username:** `sa`
* **Password:** *(leave blank)*
