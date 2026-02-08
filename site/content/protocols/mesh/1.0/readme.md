---
title: Mesh
publisher: vinaysingh8866
license: MIT
piuri: https://didcomm.org/mesh/1.0
status: Proposed
summary: A DIDComm v2 protocol for decentralized peer-to-peer mesh networking over BLE and LoRa transports with Ed25519-signed announcements, TTL-based relay flooding, topology-aware routing, and packet deduplication.
tags: [mesh, ble, lora, p2p, relay, offline]
authors:
  - name: Vinay Singh
    email: vinay@ajna.inc

---

## Summary

The Mesh Networking Protocol enables DIDComm agents to form ad-hoc mesh networks over short-range transports (BLE, LoRa) without internet connectivity. Nodes discover peers via Ed25519-signed announcements, build a topology graph, and relay encrypted DIDComm messages across multiple hops.

Key capabilities:
- Ed25519-signed peer announcements with routing ID binding
- TTL-based controlled flooding with degree-adaptive jitter
- Topology-aware shortest-path routing for directed messages
- Broadcast and directed (unicast) messaging
- Packet deduplication via LRU cache
- Graceful leave notifications
- Fragment support for MTU-constrained transports

---

## Goals

- Enable DIDComm messaging between agents with no internet connectivity
- Provide multi-hop relay so agents beyond direct radio range can communicate
- Prevent topology spoofing through Ed25519 signature verification and routing ID binding
- Minimize broadcast storms via TTL capping, deduplication, and degree-adaptive relay jitter
- Support heterogeneous transports (BLE + LoRa) in the same mesh

---

## Roles

- `node`: Any agent participating in the mesh. All nodes are equal peers with no central coordinator. Every node can act as a relay.

In practice, nodes take on transient behavioral roles:
- **announcer**: Broadcasting its presence and neighbor list
- **relay**: Forwarding packets on behalf of others
- **sender**: Originating a directed or broadcast message
- **recipient**: Final destination of a directed message

---

## Identifiers

### Routing ID

Each node is identified by an 8-byte **Routing ID** derived from its Ed25519 public key:

```
routing_id = SHA256(ed25519_public_key)[0..8]
```

Routing IDs are compact identifiers used in packet headers and topology tracking. They are cryptographically bound to the signing key, verified during announcement processing.

### DID

The mesh layer does NOT include or require DIDs. Identity exchange and connection establishment are handled separately via the [Out-of-Band](https://didcomm.org/out-of-band/2.0) protocol. DIDs are never broadcast over the mesh.

---

## Transport

The mesh protocol is transport-agnostic at the protocol level.

### BLE (Bluetooth Low Energy)

Nodes expose a GATT service with the following characteristics:

| Characteristic | UUID | Purpose |
|---------------|------|---------|
| Mesh TX | `0000A3A1-0000-1000-8000-00805F9B34FB` | Inbound data (write) |
| Mesh RX | `0000A3A2-0000-1000-8000-00805F9B34FB` | Outbound data (notify) |
| Routing ID | `0000A3A3-0000-1000-8000-00805F9B34FB` | Node's 8-byte routing ID (read) |

Service UUID: `0000A3A0-0000-1000-8000-00805F9B34FB`

### LoRa (via Meshtastic)

Messages are serialized and transmitted over Meshtastic radio channels. MTU constraints require fragmentation for messages exceeding ~200 bytes.

---

## States

```
UNINITIALIZED -> INITIALIZING -> ACTIVE <-> ANNOUNCING
                                        <-> RELAYING
                              -> SHUTTING_DOWN -> UNINITIALIZED
```

| State | Description |
|-------|-------------|
| `uninitialized` | Mesh module not started |
| `initializing` | Generating keys, setting up transport |
| `active` | Listening for announcements, accepting packets |
| `announcing` | Broadcasting our announcement (periodic, every 10s default) |
| `relaying` | Forwarding a received packet to neighbors |
| `shutting_down` | Broadcasting leave notification before shutdown |

---

## Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `default_ttl` | 7 | TTL assigned to new packets |
| `max_hops` | 10 | Maximum route length for directed messages |
| `announce_interval` | 10s | Time between periodic announcements |
| `route_freshness` | 60s | How long topology entries remain valid |
| `dedup_capacity` | 1000 | Size of deduplication LRU cache |
| `dedup_ttl` | 5min | How long dedup entries persist |
| `fragment_ttl_cap` | 5 | Maximum TTL for fragment packets |
| `high_degree_threshold` | 6 | Peer count above which broadcast TTL is capped |

---

## Basic Walkthrough

1. **Node A** starts mesh, generates Ed25519 keypair, derives routing ID from public key
2. **Node A** begins BLE advertising with its service UUID
3. **Node B** discovers Node A via BLE scan, connects, reads routing ID characteristic
4. **Node B** registers Node A as a peer
5. Both nodes begin periodic `announce` broadcasts containing their routing ID, signing key, and direct neighbor list
6. Upon receiving an announcement, each node verifies the Ed25519 signature and routing ID binding, then updates its topology graph
7. **Node A** sends a DIDComm message to **Node C** (not directly connected). The mesh handler computes a route (A -> B -> C), wraps the packed DIDComm message in a `relay` message with `dest_id=C`, `ttl=7`, and sends to B
8. **Node B** receives the relay, decrements TTL, computes next hop for C, and forwards
9. **Node C** receives the relay, sees `dest_id` matches its routing ID, extracts the DIDComm payload, and processes it
10. When **Node A** shuts down, it broadcasts a `leave` message so peers immediately remove it from their topology

---

## Relay Policy

1. **TTL <= 1 or sender is self**: Do not relay
2. **Handshake messages**: Always relay with minimal jitter (10-35ms)
3. **Directed messages**: Always relay with moderate jitter (20-60ms)
4. **Fragments**: Relay with TTL capped at `fragment_ttl_cap`
5. **Broadcasts/Announcements**: Relay with degree-adaptive jitter and TTL capping

### Degree-Adaptive Jitter

Relay delay scales with peer count to reduce broadcast storms in dense networks:

| Peer Count | Delay Range | Classification |
|------------|-------------|----------------|
| 0-2 | 10-40ms | Sparse |
| 3-5 | 60-150ms | Moderate |
| 6-9 | 80-180ms | Dense |
| 10+ | 100-220ms | Very Dense |

Dense networks (>= `high_degree_threshold` peers) also cap broadcast TTL to 5.

---

## Security

- **Announcement authentication**: Every announcement is Ed25519-signed. Nodes MUST verify the signature before updating topology
- **Routing ID binding**: `routing_id == SHA256(signing_key)[0..8]`. Nodes MUST verify this binding to prevent ID spoofing
- **Stale announcement rejection**: Announcements older than `dedup_ttl` are silently dropped
- **Deduplication**: LRU cache tracks seen packet/announcement IDs to prevent replay and flooding
- **Payload encryption**: The mesh only transports already-packed (encrypted) DIDComm messages. Relay payloads are opaque to intermediate nodes
- **No DID leakage**: Only 8-byte routing IDs appear in headers, never DIDs

---

## Message Reference

All mesh messages use the namespace: `https://didcomm.org/mesh/1.0/<message-name>`

---

### announce

Periodic broadcast to advertise presence and share topology.

Message Type URI: `https://didcomm.org/mesh/1.0/announce`

```json
{
    "type": "https://didcomm.org/mesh/1.0/announce",
    "id": "announce-a1b2c3",
    "body": {
        "routing_id": "<8 bytes, base64>",
        "signing_key": "<32 bytes Ed25519 public key, base64>",
        "direct_neighbors": [
            "<8 bytes routing_id>",
            "<8 bytes routing_id>"
        ],
        "timestamp": 1706266800000,
        "signature": "<64 bytes Ed25519 signature, base64>"
    }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `routing_id` | bytes | Yes | 8-byte routing ID derived from `signing_key` |
| `signing_key` | bytes | Yes | 32-byte Ed25519 public key |
| `direct_neighbors` | bytes[] | Yes | Routing IDs of directly connected peers |
| `timestamp` | number | Yes | Milliseconds since Unix epoch |
| `signature` | bytes | Yes | Ed25519 signature over `routing_id + signing_key + neighbors + timestamp` |

**Verification:**
1. Verify `signature` against `signing_key`
2. Verify `routing_id == SHA256(signing_key)[0..8]`
3. Reject if `timestamp` is stale (older than `dedup_ttl`)
4. Reject if already in deduplication cache (key: `routing_id + timestamp`)

---

### relay

Wraps a packed DIDComm message for multi-hop mesh delivery.

Message Type URI: `https://didcomm.org/mesh/1.0/relay`

```json
{
    "type": "https://didcomm.org/mesh/1.0/relay",
    "id": "relay-d4e5f6",
    "body": {
        "id": "pkt-7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d",
        "sender_id": "<8 bytes>",
        "dest_id": "<8 bytes or null>",
        "next_hop": "<8 bytes or null>",
        "ttl": 5,
        "timestamp": 1706266810000,
        "flags": {
            "is_handshake": false,
            "is_directed": true,
            "is_fragment": false,
            "requires_ack": false
        },
        "payload": "<packed DIDComm message>"
    }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | Yes | Unique packet ID (32 hex chars) for deduplication |
| `sender_id` | bytes | Yes | Originator's 8-byte routing ID |
| `dest_id` | bytes | No | Final destination routing ID. `null` for broadcasts |
| `next_hop` | bytes | No | Next relay hop routing ID. Set by relay nodes for routing optimization |
| `ttl` | number | Yes | Time-to-live, decremented on each relay hop |
| `timestamp` | number | Yes | Packet creation time (ms since epoch) |
| `flags` | object | Yes | Packet handling flags (see below) |
| `payload` | string | Yes | The packed (encrypted) DIDComm message |

**Flags:**

| Flag | Description |
|------|-------------|
| `is_handshake` | Session-critical message (always relayed, minimal jitter) |
| `is_directed` | Unicast message with a specific destination (mirrors `dest_id` presence) |
| `is_fragment` | Fragment of a larger message (TTL capped at `fragment_ttl_cap`) |
| `requires_ack` | Sender expects acknowledgment |

**Processing:**
1. Check dedup cache for `id + timestamp` — drop if seen
2. If `dest_id` matches our routing ID: extract `payload`, deliver to agent
3. If `sender_id` is ours: drop (don't relay our own packets)
4. Apply relay policy (see Relay Policy)
5. Decrement TTL; for directed messages, compute next hop from topology; broadcast or forward

---

### leave

Graceful departure notification.

Message Type URI: `https://didcomm.org/mesh/1.0/leave`

```json
{
    "type": "https://didcomm.org/mesh/1.0/leave",
    "id": "leave-g7h8i9",
    "body": {
        "routing_id": "<8 bytes>",
        "timestamp": 1706267000000,
        "signature": "<64 bytes Ed25519 signature>"
    }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `routing_id` | bytes | Yes | 8-byte routing ID of the departing node |
| `timestamp` | number | Yes | Departure time (ms since epoch) |
| `signature` | bytes | Yes | Ed25519 signature over `routing_id + timestamp` |

**Processing:**
1. Remove peer from topology graph
2. Emit peer-lost event

---

## Topology

Each node maintains a distributed topology graph tracking:
- **Direct neighbors**: Peers within direct BLE/LoRa range
- **Announced neighbors**: Neighbor lists received via announcements

Routes are computed on-demand using BFS over the topology graph, limited to `max_hops`. Only **bidirectionally confirmed** edges are used for routing — both peers must claim each other as neighbors.

Topology entries expire after `route_freshness` and are pruned periodically.

---

## Fragmentation

For messages exceeding the transport MTU (typically 512 bytes for BLE, ~200 bytes for LoRa):

1. Sender splits the packed message into fragments
2. Each fragment is sent as a `relay` message with `is_fragment: true`
3. Fragment TTL is capped at `fragment_ttl_cap` to prevent flood amplification
4. Receiver assembles fragments using the packet ID and sequence numbers
5. Assembly times out after 30s; maximum 128 concurrent assemblies

---

## Composition

| Protocol | Composition |
|----------|-------------|
| DID Exchange | Connection establishment messages transported over mesh relay |
| BasicMessage | Chat messages between mesh peers |
| Trust Ping | Liveness checks over mesh |
| Coordinate Mediation | Not required — mesh replaces mediators for local networks |
| Routing 2.0 | Mesh provides its own routing; DIDComm routing used for internet-bound messages |

---

## Privacy

- Intermediate relay nodes cannot read message payloads (DIDComm encryption)
- Only compact 8-byte routing IDs appear in packet headers, not DIDs
- Announcements reveal routing IDs, signing keys, and neighbor topology (inherent to mesh discovery)
- Identity and connection establishment are handled via OOB protocol, keeping DIDs out of the broadcast mesh layer

---

## Implementations
- Ajna Browser

---

## Endnotes

### Future Considerations

- LoRa transport integration via Meshtastic
- Encrypted announcements (hide topology from passive observers)
- Gossip-based topology synchronization as alternative to full-flood announcements
- Mesh-to-internet gateway nodes bridging to DIDComm mediators
- Quality-of-service routing (latency-aware, bandwidth-aware path selection)
- Group messaging with mesh multicast
- Mesh network partitioning detection and recovery
