# MAP

## Introduction 
The Mesh Anonymity Protocol (MAP) defines an identity-free, decentralized and ephemeral mesh communication system. Its goal is to create a fluctuating network that only consists of currently present participants, without any long-term identities, traceability or central instances. MAP targets environments where classical, server-based or infrastructure-backed networks (Internet, cellular networks, managed Wi-Fi) are not suitable or not desired. It enables anonymous exchange of files and messages between arbitrary devices within the physical range of local radio technologies.

## Goals The main goals of MAP are: 
- Complete absence of persistent identifiers.
- Strict separation of content and origin (no mapping of files to devices or persons).
- Use of existing short-range radio technologies (e.g. WiIFi Direct, WiIFi Aware, Bluetooth Low Energy).
- Ephemeral behavior: the network exists only through currently present devices.
- Minimal and controlled protocol headers to avoid metadata.
- Optional, revocable pseudo-identity (Ephemeral User Identifier, EUI).
- Support for portable content that can be physically moved between different meshes by users.

## Non-Goals MAP explicitly does not aim to: 
- build a global, continuously reachable network;
- provide guaranteed latency or bandwidth;
- provide long-lived, stable identities;
- offer central management, control or censorship components;
- be compatible with classical client-server application protocols by design.

## Terminology 
The following terms are used consistently in this specification:
- Node: A device that implements MAP and participates in a mesh.
- Mesh: A set of nodes that discover each other through MAP messages.
- Content: An abstract, identity-free data object (e.g. file, message, blob).
- Chunk: A fixed-size segment of content.
- ENT (Ephemeral Network Tag): Short-lived, mesh-specific tag that distinguishes mesh contexts.
- EUI (Ephemeral User Identifier): Optional, short-lived pseudo-identifier for users within a mesh.
- MNT (Mesh Name Token): Human-readable, randomly generated mesh name.
- PCE (Portable Content Extraction): Mechanism for local persistence of received content.
- RIP (ReIInject Procedure): Mechanism for re-introducing locally stored content into a mesh.

## Architectural Overview 
MAP is layered and separates transport, mesh presence, content distribution and optional identity. The architecture is structured as follows:
- Transport layer
- Mesh Presence Layer (MPL)
- Content Exchange Layer (CEL)
- Ephemeral Identity Layer (EIL)
- Portable Content Layer (PCL – PCE/RIP)
- Cosmetic presentation layer (e.g. MNT) 

## Transport Layer 
The transport layer comprises all physical and logical mechanisms used to carry MAP frames. MAP itself is transport agnostic and only defines requirements on how broadcasts and short-range links are used.
Example transports: 
- WiIFi Aware / WiIFi Direct
- AdIhoc WiIFi (802.11 without infrastructure)
- Bluetooth Low Energy (Advertising + GATT)
- Other local radio technologies with broadcast or multicast capabilities Implementations SHOULD use multiple transports in parallel, where available, to increase coverage and robustness of the mesh.

## Protocol Frame (MAP Frame Format) 
All MAP messages use a compact, common frame format. Headers are kept as small as possible to minimize metadata.

## Generic MAP Frame 
The generic MAP frame has the following layout (byte-oriented, network byte order):
- Magic (2 bytes): Protocol identifier, ASCII “MA” (0x4D41).
- Version (1 byte): Protocol version, starting at 0x01.
- Type (1 byte): Message type (e.g. Presence, Chunk Transfer).
- ENT (4 bytes): Ephemeral Network Tag.
- Flags (1 byte): Feature bits.
- PayloadLen (2 bytes): Payload length in bytes.
- Payload (0–65535 bytes): Message body according to type.
- CRC16 (2 bytes): Checksum over header + payload.

## Flags Field 
The Flags field is an 8-bit value with the following proposed meaning: 
- Bit 0: File sharing active
- Bit 1:
  Messaging active 
  - Bit 2: EUI present 
  - Bit 3–7: Reserved (MUST be set to 0 until specified).

## Mesh Presence Layer (MPL) 
The Mesh Presence Layer makes the existence of a node within a mesh visible without disclosing any identity. MPL uses periodic, anonymous Presence Announcement frames.

## Presence Announcement (Type = 0x01) 
Presence messages are sent periodically (e.g. every 2–5 seconds) over all available transports. They inform other nodes that a MAP participant is present and which services it supports in principle.
Payload structure: 
- FeatureBits (1 byte): Offered features (file, chat, etc.).
- ApproxStorage (1 byte):
  Rough indication of available temporary storage capacity (0–255, free scale). 
  - Optional: MNT-Length (1 byte). 
  - Optional: MNT-String (0–255 bytes, UTFI8).

## Ephemeral Network Tag (ENT) 
ENT is a 4-byte value that loosely characterizes the mesh group. ENT is not intended to identify individual nodes.
Example generation (non-normative): ENT = 32-bit truncation of SHAI256( random_256bit || local_epoch_byte ) Implementations SHOULD regenerate ENT regularly (e.g. every minute) to make tracking over time harder.

## Content Exchange Layer (CEL) 
The Content Exchange Layer is responsible for identity-free content exchange. Content is split into fixed-size chunks and distributed independently.

## Content and ContentHash 
Content is an anonymous byte sequence without filename, path or metadata. Each content object is identified by a ContentHash.
Recommended definition: ContentHash = 16-byte truncation of SHAI256( full_content ) 

## Chunking 
Content is split into fixed-size chunks (e.g. 2048 bytes). The final chunk may be shorter. Chunks are uniquely identified by (ContentHash, ChunkIndex).

### Content Availability (Type = 0x02) 
Content Availability messages signal which chunks of a specific content are held by a node.
Payload structure: 
- ContentHash (16 bytes)
- ChunkBitmap (variable length): Bit mask where bit i indicates availability of chunk i.

### Chunk Request (Type = 0x03) 
Chunk Request frames are used to request a specific chunk of a content object from any node.
Payload structure: 
- ContentHash (16 bytes)
- ChunkIndex (2 bytes) 

### Chunk Transfer (Type = 0x04) 
Chunk Transfer frames carry individual chunks.
Payload structure: 
- ContentHash (16 bytes)
- ChunkIndex (2 bytes)
- TotalChunks (2 bytes)
- ChunkData (0–2048 bytes)

## Ephemeral Identity Layer (EIL) 
The Ephemeral Identity Layer defines an optional concept that allows users to be temporarily addressable inside a mesh without creating any persistent identity.

## Ephemeral User Identifier (EUI) 
An EUI is a pseudo-random identifier that only has meaning within a mesh context (as defined by ENT).
Example generation (non-normative): EUI = 16-byte truncation of SHAI256( random_256bit || ENT ) Properties: 
- EUI is not globally unique.
- EUI may be regenerated by the user at any time.
- On mesh changes (different ENT) the EUI SHOULD be regenerated.
- The user may completely disable the EUI and remain anonymous.

### Identity Exchange (Type = 0x10) 
Identity Exchange messages can be used to announce EUI and capabilities (e.g. ready to receive chat messages).
Payload structure: 
- EUI (16 bytes)
- Capabilities (1 byte, e.g. chatIcapable, broadcastIonly).

## Portable Content Layer (PCL) 
PCE and RIP The Portable Content Layer describes how content can be stored locally by users and later reintroduced into other meshes without violating anonymity or protocol principles.

## Portable Content Extraction (PCE) 
PCE allows a node to persist received content locally. Only the raw content and its ContentHash are stored. Mesh-specific information (ENT, EUI, MNT, timestamps, origin devices) is not stored.
Recommended minimal storage format: 
- ContentHash (16 bytes)
- RawData (full reconstructed content) 

## ReIInject Procedure (RIP) 
RIP defines how locally stored content is re-introduced into a mesh:
Pseudocode: 
1. Read file from local storage. 
2. Perform sanitization (remove metadata if not already done). 
3. Compute or reuse ContentHash. 
4. Split file into chunks (e.g. 2048 bytes). 
5. Broadcast Content Availability frames for ContentHash with corresponding ChunkBitmap. 
6. Answer Chunk Requests and send chunks.

## Mesh Naming (Mesh Name Token, MNT) 
For human-friendly display, a mesh may be assigned a random cosmetic name. This name is non-binding, non-unique and only serves user orientation.
One possible approach is to combine a random adjective with a mythical creature, e.g. “Sparkling Tatzelwurm” or “Nebulous Leviathan”.

## MNT Generation 
1. Choose an adjective from a predefined list. 
2. Choose a mythical creature from a predefined list. 
3. Concatenate into a string, e.g. “ ”.
MNT may optionally be included in the Presence Announcement. It has no protocol-level significance and may change at any time.

## Security and Privacy Considerations 
MAP is designed to systematically remove potential sources of identifiability. Nevertheless, implementers must take care not to create additional traces outside the protocol.

## Metadata Reduction Implementations MUST ensure that: 
- no MAC addresses, hostnames or IP addresses are transported on the application layer;
- content is stripped of metadata (EXIF, XMP, office metadata etc.) before transmission;
- no permanent protocol or debug logs with content or peer information are stored.

## Traffic Analysis Resistance 
To mitigate traffic analysis, implementations SHOULD: 
- use constant packet sizes where possible;
- introduce jittered send intervals;
- avoid explicit, bidirectional sessions with clear start/stop markers;
- distribute chunk transfers across multiple transports if available.

## Anonymity Level 
The target anonymity level is such that even if people physically see each other exchanging data, the protocol and network traces alone do not allow reliable attribution of participants, roles or authorship.

## Implementation Guidelines 
The following guidelines are intended to help developers create MAP-compliant implementations.

### Minimum Requirements (MUST) 
- ENT MUST be regenerated regularly (e.g. every 60 seconds).
- Headers and payloads MUST be kept as small as practical.
- Content MUST be sanitized before transmission.
- EUI MUST only be used if explicitly enabled by the user.
- No persistent communication logs MAY be kept.

### Recommended Behavior (SHOULD) 
- Implementations SHOULD use multiple transports in parallel.
- Packet sizes SHOULD be uniform where possible. • Users SHOULD be able to switch between “fully anonymous” and “with EUI”.
- User interfaces SHOULD only show abstract information (e.g. number of nodes, number of contents).

### Optional Behavior (MAY) 
- Implementations MAY add additional cryptographic protection (e.g. end-to-end encryption of contents) as long as anonymity is not weakened.
- Implementations MAY support different chunk sizes as long as compatibility is maintained.

## Use Cases 

### Anonymous File Browser 
A MAP client can implement a file browser that only shows: 
- the current mesh name (MNT), e.g. “Flickering Harpy”;
- the estimated number of nodes in the mesh;
- a list of available anonymous contents.
When a content object is copied to local storage, only the sanitized content is stored. Upon entering another mesh, the same content can be reintroduced via RIP.

### Anonymous Messenger 
A MAP-based messenger can use EUI to better address messages within a mesh without introducing identity. Users can exchange their EUI as a QR code for targeted messaging. Once they leave the mesh or regenerate their EUI, the mapping is gone.

## Summary 
The Mesh Anonymity Protocol (MAP) defines a framework for fully anonymous, fluctuating mesh networks without central entities, persistent identities or heavy metadata. The combination of ENT, CEL, EIL, PCL and cosmetic MNT enables a broad range of applications built entirely on local, temporary network structures.
