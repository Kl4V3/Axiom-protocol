# Axiom: Decentralized and Censorship-Resistant Communication Protocol

## 🚀 Live Deployment Details

- **Network:** Arbitrum One (Mainnet L2)
- **Chain ID:** `42161`
- **RPC Endpoint:** `https://arb1.arbitrum.io/rpc` (or any custom Alchemy/Infura endpoint)
- **Axiom Proxy Contract Address:** `0xc11CFf8111e8b1F055eba095Efb679a38Abe6b63`

*(Note: Axiom uses a UUPS Upgradeable Proxy architecture. All client interactions must **always** be directed towards this Proxy address, never the underlying implementation contract).*

## 📖 How to Read Data (Indexer / Clients)

Clients should **never** attempt to read protocol posts directly from the smart contract state variables (as preserving gas is a priority, content is not stored in state). Instead, clients must **index the blockchain events**.

## ✍️ How to Publish Data (Client Tx Submission)

To publish data to the Protocol, clients must submit an on-chain transaction calling the `publishAxiom` function on the Proxy contract.

---

The goal of Axiom is to provide a completely anonymous, decentralized, and censorship-resistant social media platform.

To make this possible, the architecture is strictly divided: the foundation (the protocol) and the clients (the software). This repository defines that foundation—a smart contract and a standardized data structure on an Ethereum Layer 2 network.

## Project Scope

The protocol establishes the foundation of the platform:

- **Standard Naming Convention:** A clear structure defining how payloads are sent to the smart contract and how clients read them.
- **Immutability:** The blockchain serves as a fail-safe and tamper-proof database.
- **Microblogging Focus:** The protocol is not designed for large amounts of on-chain data, but rather follows the traditional microblogging concept (short posts). Media files are not natively stored on the chain; instead, they are embedded via external links when needed.
- **Spam Protection:** Transaction fees and a one-time, low entry fee for a wallet's first post prevent state-bloat attacks by botnets.
- **Security Routing:** The protocol enforces the OPSEC separation of messages based on specific security requirements.

---

## Protocol Security Levels

Axiom is designed to enable true freedom of speech for users in countries where communication is restricted. The protocol distinguishes between three security levels. It assumes that users know which security level is appropriate for their specific situation.

- **Level 1: Open**
    Communication occurs openly in plaintext on the blockchain. All clients can read and process the entire traffic. Embedding links (e.g., for images via third-party providers) is permitted at this level. The expectation is that standard social media communication takes place here—including funny cat pictures. This generates important noise within the network. It is less secure regarding pure IP tracking, but the posts remain entirely un-censorable.
- **Level 2: Closed**
    This level is intended for strict security. It exclusively supports plain text messages. The protocol prohibits media links here to technically rule out any IP leaks when clients load external content. Users must independently ensure they acquire the cryptocurrency used for gas fees anonymously.
- **Level 3: Encrypted**
    Built for maximum privacy. The messages themselves are encrypted with AES-256-GCM before being sent. Only metadata, the initialization vector (IV), and the ciphertext are stored on the blockchain. Only clients of users who possess the correct cryptographic key can decrypt and read these messages.

### Network Security & IP Tracking (Hard Rule)

For **all levels**, Onion Routing (e.g., Tor) is strictly mandatory. Communicating with commercial RPC providers (like Infura or Alchemy) leaks the sender's IP address in plaintext. To close potentially life-threatening OPSEC vulnerabilities for dissidents, clients are required to route transactions to the RPC nodes exclusively through the Tor network.

---

## Cryptography Standards (For Level 3)

For messages on Level 3, all clients must strictly adhere to the following cryptographic standards to ensure interoperability and avoid compromising security.

1. **Encryption Algorithm: AES-256-GCM**
    All Level 3 payloads must be symmetrically encrypted using AES in GCM mode with a 256-bit key length. The initialization vector (IV/Nonce) must be randomly regenerated for every single message and is written to the blockchain as plaintext metadata. This prevents pattern recognition by external observers.
2. **Key Derivation: Argon2id**
    Users enter human-readable passwords into their clients. These must never be used directly as AES keys. Clients are strictly required to use the Argon2id hashing algorithm. *(Note: Developers must define fixed parameters for iterations and memory usage within the client so that all generate the exact same key).*
3. **Key Exchange: Out-of-Band**
    Axiom does not handle on-chain key exchanges. The protocol stores no public keys. Exchanging the password (shared secret) for a specific channel is the responsibility of the users and must occur outside the network (e.g., in person).
4. **Data Integrity**
    AES-GCM generates an Authentication Tag. Clients must validate this tag. If validation fails, the client must silently discard the message (drop).

---

## Data Structure, Payload Delivery, and Indexing

Axiom utilizes a hybrid payload delivery (ABI Split). To prevent the smart contract from having to unpack expensive data formats, the data is separated prior to transmission:

1. **Logic Variables:** The level (`uint8 _level`) and the initialization vector (`bytes _iv`) are passed as direct parameters to the smart contract, as it requires them to enforce its security rules.
2. **Opaque Data:** The actual message content is constructed internally by the client as JSON and compressed into **CBOR (Concise Binary Object Representation)**. The contract handles this CBOR package "blindly" and forwards it directly to the event log.

### Payload Keys (CBOR Structure)

Axiom uses single letters as keys to save bytes. The author (`msg.sender`) and timestamp (`block.timestamp`) are omitted, as the smart contract extracts these values in a tamper-proof manner anyway.

- `t` (Type): Integer. The type of action.
- `c` (Content): String/Bytes. The text, name, or ciphertext.
- `h` (Hashtags/Tags): Array. Optional. Used for categorization (subchannels).
- `m` (Message Hint): Bytes (length of 2). **Level 3 only.** A 2-byte HMAC hash used for fuzzy bucketing.
- `r` (Reply-To): Bytes. Optional. The transaction hash of a referenced post.

### Action Types (`t` Field)

- `0` = Profile Update (Links the wallet address to a readable name in field `c`)
- `1` = Post (Standard message)
- `2` = Reply (`r` requires the hash of the original post)
- `3` = Like (`r` requires the hash of the post)
- `4` = Unlike (Reverts Type 3)
- `5` = Retweet / Repost (`r` requires the hash of the post)
- `6` = Un-Retweet (Reverts Type 5)

### Subchannels and Dark Routing (`h` Field)

- **Level 1 & 2:** Tags are passed in plaintext.
- **Level 3 (Encrypted):** Passing tags in plaintext is strictly prohibited at the protocol level, as this leaks metadata. Tags must be encrypted exactly like the content (`c`). For external observers, the tags are therefore completely invisible (Dark Routing).

### Level 3: Fuzzy Bucketing (`m` Field)

Because tags are encrypted on Level 3, clients theoretically have to attempt to decrypt every single message (Trial Decryption). To prevent CPU overloads, Axiom uses Message Hints:

- The sender calculates `HMAC-SHA256(AES_Key, IV)` and places the first **2 bytes** as field `m` into the CBOR payload.
- Receivers calculate this hint for their locally stored passwords. The expensive decryption process is only executed if there is a match. This filters out 99.99% of irrelevant traffic without leaking metadata.

---

## Identity: Global Profiles vs. Private Aliases

Axiom handles identity with total transparency: **The L2 wallet address (`msg.sender`) is the sole social and financial identity.** The protocol transfers the responsibility for financial OPSEC entirely to the user (e.g., the use of mixers and bridges to anonymously procure gas tokens).

The profile update action (`t: 0`) behaves differently depending on the chosen security level:

1. **Global Identity (Level 1 & 2)**
    If a wallet sends an unencrypted profile update, it serves as a global declaration. The wallet becomes known network-wide under this name. Everyone sees this name (e.g., building a public reputation as `@Dissident99`).
2. **Private Aliases & Nicknames (Level 3)**
    If a wallet sends a profile update within an encrypted Level 3 payload, it creates an isolated, private alias. This alias is only visible within the decrypted subchannel to users who know the password. This allows for pseudonymous role distributions in closed groups without altering the global identity. *Rendering Priority:* For Level 3, the frontend must always check if a local alias exists before falling back to the global name.

---

## Smart Contract Architecture & OPSEC Enforcements

Axiom relies on on-chain validation with O(1) complexity. To keep gas costs at an absolute minimum, the smart contract only performs basic cryptographic checks. All resource-intensive content validations are offloaded to the clients (Layer 2).

### 1. On-Chain Validation (The Smart Contract)

The contract acts as an incorruptible bouncer. If a payload does not adhere to the strict rules, the transaction is reverted.

```Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";
import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";

contract Axiom is Initializable, UUPSUpgradeable, OwnableUpgradeable {
    uint256 public entryFee;
    mapping(address => uint8) public walletPath; // 0=New, 1=PathA(Level1), 2=PathB(Level2/3)

    event AxiomPost(address indexed sender, uint8 level, bytes iv, bytes cbor, uint256 timestamp);

    /// @custom:oz-upgrades-unsafe-allow constructor
    constructor() {
        _disableInitializers();
    }

    function initialize() initializer public {
        __Ownable_init(msg.sender);
        entryFee = 0.0001 ether;
    }

    function _authorizeUpgrade(address newImplementation) internal override onlyOwner {}

    function publishAxiom(
        uint8 _level,
        bytes calldata _iv,
        bytes calldata _cbor
    ) external payable {
        require(_level >= 1 && _level <= 3, "Invalid level");

        uint8 requiredPath = (_level == 1) ? 1 : 2;
        uint8 currentPath = walletPath[msg.sender];

        if (currentPath == 0) {
            require(msg.value >= entryFee, "Anti-Sybil: Insufficient entry fee");
            walletPath[msg.sender] = requiredPath;
        } else {
            require(msg.value == 0, "Fee already paid");
            require(currentPath == requiredPath, "OPSEC Violation: Wallet is tainted");
        }

        if (_level == 3) {
            require(_iv.length == 12, "Level 3 strictly requires a 12-byte IV");
        } else {
            require(_iv.length == 0, "Level 1 and 2 require strictly empty IV");
        }

        emit AxiomPost(msg.sender, _level, _iv, _cbor, block.timestamp);
    }

    function withdraw() external onlyOwner {
        payable(owner()).transfer(address(this).balance);
    }
}
```

- **Bi-directional Wallet Taint (Forced Isolation):** If a wallet posts on Level 1 for the first time, it is permanently blocked from Level 2/3. If it posts on Level 2 or 3 first, Level 1 is blocked.
- **Must-Have Parameters:** For Level 3, the contract strictly enforces a 12-byte IV.

### 2. Off-Chain Validation (By the Clients)

If a post violates protocol rules, the client must discard it silently (Local Drop).

- **The Level 2 Link Blocker:** Clients scan the plaintext (`c`) of Level 2 messages. If URLs, IP addresses, or typical media tags are detected, the post is completely blocked.
- **Self-Cleaning:** If an attacker spams the network with links on Level 2, they will pay the gas fees, but no valid Axiom client will ever render those messages.

---

## Infrastructure & Recommended Client Architecture

Axiom is deployed on an **Ethereum Layer 2 (L2) network** (e.g., Arbitrum Nova).

To avoid overloading mobile devices (battery life, storage limitations, WebAssembly limits for Argon2id), Axiom enforces a highly performant client architecture:

- **Axiom Core (Self-Hosted Node):** A server/Docker container (e.g., running on a NAS) that reads the blockchain via RPC, indexes events, and natively executes the resource-intensive cryptography.
- **Axiom UI (Thin Client):** A mobile app or Web UI that solely communicates with the user's own Axiom Core via an API.

### Data Retrieval and EIP-4444

Clients do not download the entire blockchain state. They filter for the smart contract's `AxiomPost` event, which contains all necessary data in plaintext (Sender, Level, IV, CBOR, Timestamp).

Because Ethereum nodes will eventually discard historical data (events older than 365 days) according to EIP-4444, the protocol recommends that local *Axiom Cores* serve as decentralized archives, storing the databases permanently.

---

## Example Workflow: A Level 3 Post

To illustrate how the architecture works in practice, here is a complete lifecycle run-through.

**Scenario:** Alice wants to post the message "Meeting at 8 PM" into the subchannel "AxiomDev". The group previously agreed offline on the password "Secret123".

### Step 1: Local Preparation and Encryption (Axiom Core)

Alice's Axiom Core handles the computational heavy lifting:

1. **Key Derivation:** It converts the password into a 256-bit AES key using Argon2id.
2. **IV Generation:** A random 12-byte initialization vector is generated (e.g., `0x12ab34cd56ef789012ab34cd`).
3. **Encryption:** The content and the tag ("AxiomDev") are encrypted using AES-GCM.
4. **Hint Generation:** The 2-byte HMAC hash for fuzzy bucketing is calculated (`m: "0xa1b2"`).

### Step 2: Payload Construction (CBOR Serialization)

Since the level and the IV are passed directly to the contract, they are excluded from the CBOR object.

**Internal JSON Representation:**

```JSON
{
  "t": 1,
  "c": "0x8a4f...",
  "h": ["0x9b5e..."],
  "m": "0xa1b2"
}
```

This JSON is compressed into a raw CBOR byte array (`0xa3617401...`) to save gas.

### Step 3: The Smart Contract Call (Blockchain Interaction)

Alice triggers the contract function. *Important: The call is strictly routed through Tor!*

*(Examples for L3 and L1:)*

```JavaScript
// Example 1: Client calling the Smart Contract for an encrypted Level 3 post
await axiomContract.publishAxiom(
    3,                                      // _level: 3
    "0x12ab34cd56ef789012ab34cd",           // _iv: 12 bytes hex string required
    "0xa3617401616358208a4f..."             // _cbor: packed CBOR hex string
);

// Example 2: Client calling the Smart Contract for a public Level 1 post
await axiomContract.publishAxiom(
    1,                                      // _level: 1
    "0x",                                   // _iv: strictly empty byte array
    "0xa361740161634c48656c6c6f204178..."   // _cbor: packed CBOR hex string
);
```

The smart contract then emits the event:

`Event: AxiomPost(Sender: 0xAlice..., Level: 3, IV: 0x12ab..., CBOR: 0xa361..., Timestamp: 1710425890)`

### Step 4: Indexing and Trial Decryption (Receiver)

Bob's Axiom Core is listening to the blockchain and receives the event.

1. **Local Storage & Recognition:** The event is written to the local database. The Core detects `Level 3` and unpacks the CBOR to access the content, tags, and the hint `m`.
2. **Trial Decryption:** The Core checks the 2-byte hint `m` against Bob's stored passwords.
3. **Match & Forwarding:** The hint matches "Secret123". The GCM tag confirms that the payload has not been tampered with. The data is decrypted in RAM and sent to Bob's smartphone via the local API. The message "Meeting at 8 PM" appears in the "#AxiomDev" feed.
4. **Unknown Backlog:** For users without the correct password, the hint check fails. This unreadable data noise is automatically deleted after a rolling buffer expires (e.g., 30 days).
