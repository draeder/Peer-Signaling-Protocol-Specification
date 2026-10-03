# Peer Signaling Protocol Specification

**Version:** 1.0
**Status:** Draft
**Scope:** Peer-to-peer signaling and discovery for WebRTC-based meshes, partial meshes, relay-assisted meshes, and hybrid decentralized overlays.
**Revision:** 2 (additive; envelopes remain `psp_version` `"1.0"`). Revision 2 adds mesh-carried signaling (19.4), relay on demand (19.5), anchors (19.6), peer channel control frames (19.7) and relay federation (19.8). A peer that implements none of them remains conforming.

## 1. Purpose

This protocol defines a transport-agnostic signaling layer for establishing and maintaining WebRTC peer connectivity without requiring a centralized signaling model.

It standardizes how peers:

* identify themselves
* announce presence
* discover candidate peers
* exchange offers, answers, and ICE candidates
* negotiate roles and topology hints
* handle retries, liveness, and teardown
* operate over arbitrary signaling transports

This protocol does **not** define media encoding, application payload semantics, or routing of end-user data after a WebRTC connection is established.

---

## 2. Design Goals

1. **Transport agnostic**
   Must work over WebSocket, HTTP polling, long-polling, Nostr, DHT relays, PubSub, local LAN broadcast gateways, or any other message substrate.

2. **Peer-first**
   No central coordinator is required for correctness, though bootstrap and relay services may be used.

3. **WebRTC-native**
   Must support SDP offer/answer exchange and trickle ICE.

4. **Topology-aware**
   Must support direct meshes, partial meshes, relay-assisted overlays, and role-based peers.

5. **Deterministic enough to interoperate**
   Different implementations should be able to establish sessions without prior coordination beyond this spec.

6. **Extensible**
   New message types and capabilities must be negotiable without breaking older implementations.

7. **Security-conscious**
   Peer identity and message authenticity should be supported, even when signaling flows through untrusted relays.

---

## 3. Terminology

### 3.1 Peer

An endpoint participating in discovery, signaling, and possibly WebRTC data/media transport.

### 3.2 Signaling Transport

Any substrate used to carry signaling messages between peers, including relays.

### 3.3 Session

A signaling exchange between two peers intended to produce a WebRTC connection.

### 3.4 Announcement

A message advertising peer presence, reachability, role, and capabilities.

### 3.5 Role

A topology or function hint such as `peer`, `bootstrap`, `relay`, `super`, `observer`, `publisher`, `subscriber`.

### 3.6 Trickle ICE

Incremental delivery of ICE candidates after offer/answer exchange begins.

### 3.7 Correlation ID

A unique identifier linking all messages associated with one connection attempt.

---

## 4. Protocol Model

The protocol has four layers:

1. **Identity Layer**
   Defines peer IDs and optional signing keys.

2. **Discovery Layer**
   Announces peers and discovers candidate peers.

3. **Negotiation Layer**
   Exchanges offers, answers, ICE candidates, and session control messages.

4. **Control Layer**
   Handles liveness, teardown, re-announcement, redirection, and extensions.

---

## 5. Identity

Each peer MUST have a stable `peer_id` within a logical network namespace.

### 5.1 Peer ID

`peer_id` is an opaque string. Implementations SHOULD derive it from a public key fingerprint or hash.

Example forms:

* SHA-1 hex of a public key
* SHA-256 hex of a public key
* multibase/multihash identifier
* application-defined stable node ID

This spec does not mandate a specific hash algorithm, but all peers in a network namespace MUST agree on the format.

### 5.2 Network Namespace

A `network` field MUST scope peer discovery and signaling. This prevents unrelated overlays from colliding on the same transport.

Examples:

* `example-main`
* `room:abc123`
* `app:chat:prod`
* `mesh:us-east`

### 5.3 Instance ID

A peer process SHOULD generate an `instance_id` each time it starts. This distinguishes restarts from a long-lived stable identity.

---

## 6. Envelope Format

All messages MUST use a common envelope.

```json
{
  "psp_version": "1.0",
  "type": "announce",
  "network": "example-main",
  "from": "peer-a",
  "to": null,
  "session_id": null,
  "message_id": "01HV7Y9M6P4A1KJ8D2XQZ3T9WR",
  "timestamp": 1775030400000,
  "ttl_ms": 30000,
  "body": {},
  "sig": null
}
```

---

## 7. Common Envelope Fields

### 7.1 Required Fields

* `psp_version`: protocol version string
* `type`: message type
* `network`: logical namespace
* `from`: sender peer ID
* `message_id`: unique message ID
* `timestamp`: sender timestamp in milliseconds since Unix epoch
* `body`: message-specific object

### 7.2 Optional Fields

* `to`: recipient peer ID, or `null` for broadcast/discovery
* `session_id`: correlation ID for one signaling exchange
* `ttl_ms`: validity lifetime for the message
* `sig`: detached or embedded signature object
* `hop_count`: relay hop count
* `trace`: optional relay/debug path metadata
* `reply_to`: message ID being answered
* `priority`: advisory delivery priority

### 7.3 Message ID

`message_id` MUST be globally unique with high probability. ULID, UUIDv4, or cryptographically random identifiers are acceptable.

---

## 8. Message Types

This protocol defines the following core message types:

### Discovery

* `announce`
* `withdraw`
* `discover`
* `peer_list`
* `redirect`

### Negotiation

* `connect_request`
* `connect_accept`
* `connect_reject`
* `offer`
* `answer`
* `ice_candidate`
* `ice_end`
* `renegotiate`

### Control

* `ping`
* `pong`
* `bye`
* `error`
* `ack`

### Extension

* `ext`

---

## 9. Discovery Messages

## 9.1 `announce`

Used by a peer to advertise presence and capabilities.

```json
{
  "psp_version": "1.0",
  "type": "announce",
  "network": "example-main",
  "from": "peer-a",
  "message_id": "msg-001",
  "timestamp": 1775030400000,
  "ttl_ms": 30000,
  "body": {
    "instance_id": "inst-01",
    "roles": ["peer"],
    "capabilities": {
      "trickle_ice": true,
      "restart_ice": true,
      "datachannel": true,
      "media": false,
      "sdp_semantics": ["unified-plan"]
    },
    "transports": {
      "signal": ["nostr", "websocket", "http-poll"],
      "rtc": ["webrtc"]
    },
    "hints": {
      "max_peers": 8,
      "connected_peers": 3,
      "wants_peers": true,
      "nat": "unknown",
      "region": "us-east"
    },
    "pubkey": "base64-or-jwk-or-opaque",
    "metadata": {}
  }
}
```

### Rules

* Peers SHOULD periodically re-announce before `ttl_ms` expiry, while they hold a relay connection (see 19.5: a peer that has released its relay connection does not re-announce).
* A relay SHOULD NOT rewrite a registration that still has at least half of its `ttl_ms` remaining; it SHOULD extend the lease only when half or more of it has elapsed.
* Consumers MUST treat expired announcements as stale.
* Relays MAY cache and rebroadcast live announcements within the TTL window.
* `announce` SHOULD be idempotent.

---

## 9.2 `withdraw`

Signals that a peer or instance is intentionally leaving.

```json
{
  "psp_version": "1.0",
  "type": "withdraw",
  "network": "example-main",
  "from": "peer-a",
  "message_id": "msg-002",
  "timestamp": 1775030410000,
  "body": {
    "instance_id": "inst-01",
    "reason": "shutdown"
  }
}
```

### Rules

* Receivers SHOULD mark the sender unavailable immediately.
* If `instance_id` does not match a known live instance, receivers MAY ignore it.

---

## 9.3 `discover`

Requests peers, relays, or bootstrap nodes to provide candidate peers.

```json
{
  "psp_version": "1.0",
  "type": "discover",
  "network": "example-main",
  "from": "peer-a",
  "to": null,
  "message_id": "msg-003",
  "timestamp": 1775030420000,
  "body": {
    "want_roles": ["peer", "bootstrap"],
    "exclude_peers": ["peer-b"],
    "limit": 16,
    "constraints": {
      "max_connected": 8,
      "same_region_preferred": true
    }
  }
}
```

### Rules

* `discover` may be broadcast or sent to a known helper peer.
* Receivers MAY respond with `peer_list`.
* Peers SHOULD avoid returning peers already connected to the requester when known.

---

## 9.4 `peer_list`

Returns candidate peers.

```json
{
  "psp_version": "1.0",
  "type": "peer_list",
  "network": "example-main",
  "from": "bootstrap-1",
  "to": "peer-a",
  "message_id": "msg-004",
  "reply_to": "msg-003",
  "timestamp": 1775030421000,
  "body": {
    "peers": [
      {
        "peer_id": "peer-c",
        "instance_id": "inst-c1",
        "roles": ["peer"],
        "pubkey": "opaque",
        "last_seen": 1775030419000,
        "hints": {
          "connected_peers": 2,
          "max_peers": 8,
          "region": "us-east"
        }
      },
      {
        "peer_id": "peer-d",
        "instance_id": "inst-d1",
        "roles": ["relay", "peer"],
        "pubkey": "opaque",
        "last_seen": 1775030418000,
        "hints": {
          "connected_peers": 5,
          "max_peers": 64,
          "region": "us-central"
        }
      }
    ]
  }
}
```

---

## 9.5 `redirect`

Suggests a better signaling target or alternate route.

```json
{
  "psp_version": "1.0",
  "type": "redirect",
  "network": "example-main",
  "from": "bootstrap-1",
  "to": "peer-a",
  "message_id": "msg-005",
  "timestamp": 1775030422000,
  "body": {
    "kind": "better-peer",
    "target_peer": "peer-d",
    "reason": "lower-load"
  }
}
```

---

## 10. Session Establishment

A signaling exchange for a WebRTC connection MUST be scoped by `session_id`.

A session starts when one peer sends `connect_request`, `offer`, or another negotiation-initiation message with a new `session_id`.

### 10.1 Session ID

`session_id` MUST be unique per connection attempt. A new attempt after failure MUST use a new `session_id`.

---

## 11. Connection Management Messages

## 11.1 `connect_request`

Optional preflight request before SDP exchange.

```json
{
  "psp_version": "1.0",
  "type": "connect_request",
  "network": "example-main",
  "from": "peer-a",
  "to": "peer-c",
  "session_id": "sess-001",
  "message_id": "msg-006",
  "timestamp": 1775030430000,
  "body": {
    "intent": "webrtc-datachannel",
    "initiator_preference": "auto",
    "roles": {
      "from_role": "peer",
      "requested_role": "peer"
    },
    "capabilities": {
      "trickle_ice": true,
      "restart_ice": true,
      "datachannel": true,
      "media": false
    },
    "topology_hints": {
      "desired_link": "direct",
      "priority": 10
    }
  }
}
```

### Purpose

* load shedding
* admission control
* initiator collision reduction
* topology filtering

Peers MAY skip this step and go straight to `offer`.

---

## 11.2 `connect_accept`

```json
{
  "psp_version": "1.0",
  "type": "connect_accept",
  "network": "example-main",
  "from": "peer-c",
  "to": "peer-a",
  "session_id": "sess-001",
  "message_id": "msg-007",
  "reply_to": "msg-006",
  "timestamp": 1775030430100,
  "body": {
    "initiator_decision": "peer-a",
    "expires_in_ms": 10000
  }
}
```

### Rules

* The responder MAY designate which peer should create the offer.
* The requester SHOULD begin SDP exchange within `expires_in_ms`.

---

## 11.3 `connect_reject`

```json
{
  "psp_version": "1.0",
  "type": "connect_reject",
  "network": "example-main",
  "from": "peer-c",
  "to": "peer-a",
  "session_id": "sess-001",
  "message_id": "msg-008",
  "reply_to": "msg-006",
  "timestamp": 1775030430200,
  "body": {
    "code": "capacity_reached",
    "reason": "max peers reached",
    "retry_after_ms": 30000,
    "alternates": ["peer-d", "peer-e"]
  }
}
```

---

## 12. SDP Negotiation

## 12.1 `offer`

```json
{
  "psp_version": "1.0",
  "type": "offer",
  "network": "example-main",
  "from": "peer-a",
  "to": "peer-c",
  "session_id": "sess-001",
  "message_id": "msg-009",
  "timestamp": 1775030431000,
  "body": {
    "sdp": "v=0\r\n...",
    "trickle_ice": true,
    "restart_ice": false,
    "setup_role": "actpass"
  }
}
```

### Rules

* The sender MUST be prepared to receive `ice_candidate` messages immediately after sending an offer.
* The receiver MUST correlate the offer using `session_id`.

---

## 12.2 `answer`

```json
{
  "psp_version": "1.0",
  "type": "answer",
  "network": "example-main",
  "from": "peer-c",
  "to": "peer-a",
  "session_id": "sess-001",
  "message_id": "msg-010",
  "reply_to": "msg-009",
  "timestamp": 1775030432000,
  "body": {
    "sdp": "v=0\r\n...",
    "trickle_ice": true,
    "restart_ice": false
  }
}
```

### Rules

* A peer MUST NOT send multiple active answers for one session unless renegotiation is explicitly underway.
* Receivers SHOULD reject malformed SDP with an `error`.

---

## 12.3 `ice_candidate`

```json
{
  "psp_version": "1.0",
  "type": "ice_candidate",
  "network": "example-main",
  "from": "peer-a",
  "to": "peer-c",
  "session_id": "sess-001",
  "message_id": "msg-011",
  "timestamp": 1775030432500,
  "body": {
    "candidate": {
      "candidate": "candidate:...",
      "sdpMid": "0",
      "sdpMLineIndex": 0,
      "usernameFragment": "abcd"
    }
  }
}
```

### Rules

* Peers MAY send zero or more `ice_candidate` messages.
* Receivers SHOULD queue candidates briefly if remote SDP is not yet applied.
* Candidates without a matching session MUST be ignored or rejected.

---

## 12.4 `ice_end`

Signals end-of-candidates.

```json
{
  "psp_version": "1.0",
  "type": "ice_end",
  "network": "example-main",
  "from": "peer-a",
  "to": "peer-c",
  "session_id": "sess-001",
  "message_id": "msg-012",
  "timestamp": 1775030434000,
  "body": {}
}
```

### Rules

* `ice_end` is OPTIONAL but RECOMMENDED.
* Peers MAY infer end-of-candidates through implementation-specific WebRTC APIs.

---

## 12.5 `renegotiate`

Requests fresh offer/answer exchange on an existing peer connection.

```json
{
  "psp_version": "1.0",
  "type": "renegotiate",
  "network": "example-main",
  "from": "peer-c",
  "to": "peer-a",
  "session_id": "sess-001",
  "message_id": "msg-013",
  "timestamp": 1775030500000,
  "body": {
    "reason": "ice_restart",
    "ice_restart": true
  }
}
```

---

## 13. Control Messages

## 13.1 `ping`

```json
{
  "psp_version": "1.0",
  "type": "ping",
  "network": "example-main",
  "from": "peer-a",
  "to": "peer-c",
  "message_id": "msg-014",
  "timestamp": 1775030600000,
  "body": {
    "nonce": "r4nd0m"
  }
}
```

## 13.2 `pong`

```json
{
  "psp_version": "1.0",
  "type": "pong",
  "network": "example-main",
  "from": "peer-c",
  "to": "peer-a",
  "message_id": "msg-015",
  "reply_to": "msg-014",
  "timestamp": 1775030600050,
  "body": {
    "nonce": "r4nd0m"
  }
}
```

### Use

* signaling transport liveness
* relay liveness
* pre-connect reachability testing

---

## 13.3 `bye`

Graceful session termination.

```json
{
  "psp_version": "1.0",
  "type": "bye",
  "network": "example-main",
  "from": "peer-a",
  "to": "peer-c",
  "session_id": "sess-001",
  "message_id": "msg-016",
  "timestamp": 1775030700000,
  "body": {
    "reason": "disconnect"
  }
}
```

### Rules

* `bye` may refer to signaling session teardown, WebRTC teardown, or both.
* After `bye`, both peers SHOULD discard pending signaling state for that `session_id`.

---

## 13.4 `error`

```json
{
  "psp_version": "1.0",
  "type": "error",
  "network": "example-main",
  "from": "peer-c",
  "to": "peer-a",
  "session_id": "sess-001",
  "message_id": "msg-017",
  "reply_to": "msg-009",
  "timestamp": 1775030431500,
  "body": {
    "code": "invalid_sdp",
    "reason": "failed to parse offer",
    "fatal": true
  }
}
```

### Standard Error Codes

* `invalid_message`
* `unsupported_version`
* `unsupported_type`
* `invalid_signature`
* `wrong_network`
* `unknown_peer`
* `unknown_session`
* `expired_message`
* `capacity_reached`
* `forbidden`
* `invalid_sdp`
* `invalid_candidate`
* `timeout`
* `conflict`
* `internal_error`

---

## 13.5 `ack`

Optional acknowledgment for transports that do not guarantee delivery.

An `ack` confirms delivery of one message on one signaling transport. It is not a liveness signal for peers that are connected to each other: the liveness of an established WebRTC edge MUST be judged on that edge (see 19.7), never on relay acknowledgements. A peer that has released its relay connection on purpose (19.5) receives none and MUST NOT treat their absence as a fault.

```json
{
  "psp_version": "1.0",
  "type": "ack",
  "network": "example-main",
  "from": "peer-c",
  "to": "peer-a",
  "message_id": "msg-018",
  "reply_to": "msg-011",
  "timestamp": 1775030432510,
  "body": {}
}
```

---

## 14. Extension Messages

## 14.1 `ext`

Carries implementation-specific extensions.

```json
{
  "psp_version": "1.0",
  "type": "ext",
  "network": "example-main",
  "from": "peer-a",
  "to": "peer-c",
  "session_id": "sess-001",
  "message_id": "msg-019",
  "timestamp": 1775030800000,
  "body": {
    "namespace": "com.example.mesh",
    "name": "role_announce",
    "payload": {
      "role": "super"
    }
  }
}
```

### Rules

* Receivers MUST ignore unknown extensions unless marked required by prior capability negotiation.
* Extension namespaces SHOULD be collision-resistant.

---

## 15. State Machine

## 15.1 Discovery State

A peer is either:

* `unknown`
* `discovered`
* `announced`
* `stale`
* `withdrawn`

## 15.2 Session State

Recommended signaling session states:

* `new`
* `requesting`
* `accepted`
* `offered`
* `answered`
* `connecting`
* `connected`
* `failed`
* `closed`

### Typical Outbound Flow

1. discover peer
2. send `connect_request`
3. receive `connect_accept`
4. send `offer`
5. exchange `answer`
6. exchange `ice_candidate`
7. mark `connected` when WebRTC connects
8. send `bye` on close

### Fast Path

1. discover peer
2. send `offer`
3. receive `answer`
4. exchange candidates

---

## 16. Collision Handling

Offer glare can occur when two peers simultaneously initiate.

### Recommended Rule

Each peer computes a deterministic winner for offerer role.

Example:

* Compare `peer_id`
* Lower lexical value becomes initiator
* Higher lexical value waits for offer

Alternative:

* Use `connect_request` / `connect_accept`
* Use random tie-breakers in `body.tie_breaker`

### Requirement

Implementations MUST have a deterministic glare strategy or use `connect_request`.

---

## 17. Reliability and Retries

Because the signaling transport may be lossy:

* Important messages SHOULD be retried until acknowledged, expired, or superseded.
* Retries MUST preserve `session_id` but SHOULD use a new `message_id`.
* Receivers SHOULD deduplicate by `(from, message_id)`.
* Duplicate offers/answers/candidates MUST be handled safely.

### Suggested Retry Strategy

* initial resend after 1 second
* exponential backoff
* max 5 attempts
* stop on terminal `error`, `connect_reject`, `bye`, or expiry

---

## 18. Security

## 18.1 Signing

Messages SHOULD be signed when signaling flows through untrusted intermediaries.

Example `sig` object:

```json
{
  "alg": "ed25519",
  "kid": "peer-a-key-1",
  "value": "base64-signature"
}
```

### Signed Payload

The signature SHOULD cover all envelope fields except `sig`.

## 18.2 Key Advertisement

Peers MAY advertise public keys in `announce.body.pubkey`.

## 18.3 Replay Protection

Receivers SHOULD reject:

* expired messages
* duplicate `message_id`s inside a replay window
* messages with stale timestamps outside local policy

## 18.4 Network Isolation

Receivers MUST discard messages for the wrong `network`.

## 18.5 Authorization

This spec allows, but does not require, application-specific access control. Implementations MAY reject unknown peers, unsigned peers, or peers lacking required capabilities.

---

## 19. Transport Binding

This protocol is transport agnostic. A transport binding MUST preserve the envelope faithfully.

### 19.1 Required Transport Properties

A signaling transport SHOULD support:

* peer-addressable delivery or topic-based fanout
* binary-safe or UTF-8-safe payload delivery
* messages up to at least 64 KB
* moderate latency suitable for negotiation

### 19.2 Encoding

Default encoding is UTF-8 JSON.

Alternative encodings such as CBOR or MessagePack MAY be used if all participating peers agree.

### 19.3 Relay Behavior

Relays SHOULD:

* preserve message bodies without mutation
* honor TTL where practical
* avoid duplicating identical messages excessively
* not assume trust

### 19.4 Mesh-Carried Signaling

A relay is for meeting the mesh. Once two peers are connected, signaling for a third peer, and for the pair itself, can travel on the connections they already have.

A peer that holds an established data channel to a neighbour MAY carry a signaling envelope over that channel instead of the relay.

* Eligible types are the negotiation messages: `connect_request`, `connect_accept`, `connect_reject`, `offer`, `answer`, `ice_candidate`, `ice_end`, `renegotiate`, and `bye`. Discovery messages and control messages (`announce`, `withdraw`, `discover`, `peer_list`, `ping`, `pong`, `ack`) address a relay and are not carried this way.
* A sender SHOULD try the mesh first for an eligible message addressed to one peer, and use the relay only when the mesh cannot carry it. A sender MAY send on both while the addressee has not yet answered over the mesh, and SHOULD stop sending the relay copy after the first mesh-carried message from that peer.
* The envelope is carried unmodified, as the `envelope` member of a peer channel control frame (19.7):

```json
{ "type": "~signal", "envelope": { "psp_version": "1.0", "type": "offer", "network": "example-main", "from": "peer-a", "to": "peer-b", "session_id": "room-1", "message_id": "msg-021", "timestamp": 1775030440000, "ttl_ms": 30000, "body": {} } }
```

* The receiver MUST apply the same checks as for an envelope delivered by a relay: addressed to this peer, same `network`, same `session_id`, and `from` is not this peer. Anything else MUST be dropped.
* When the frame arrives on a channel and no forwarding is in use, `from` MUST equal the peer on the other end of that channel. A peer MAY forward a frame for another peer only without changing the envelope, with a bounded hop count, and with deduplication by `(from, message_id)`; because forwarders are untrusted, signing (18.1) is RECOMMENDED when forwarding is used.
* A mesh-carried frame MUST NOT, by itself, cause a released relay connection to be reopened (19.5).

### 19.5 Relay On Demand

A peer with a healthy mesh does not need a relay connection, and holding one costs the relay work on every re-announcement and acknowledgement. A peer that can carry signaling over the mesh (19.4) MAY release its relay connection while the mesh is healthy, and MUST reopen it when it needs one. A peer that cannot carry signaling over the mesh MUST keep its relay connection.

A **neighbour** is a peer with an open data channel whose liveness has been confirmed within the keepalive window (19.7).

A peer SHOULD release when all of these have held continuously for the settle period (default 30 s):

1. it has at least `min_mesh_peers` neighbours (default 2);
2. it is not an anchor (19.6);
3. no negotiation is in flight, because the answer to one may still be coming by relay.

To release, a peer:

1. sends `withdraw` with `body.reason` set to `relay_idle`;
2. closes the signaling connection with a normal closure;
3. keeps every data channel open, and does not reconnect the signaling connection while the conditions above hold.

A relay that receives `withdraw` with reason `relay_idle` SHOULD drop the registration at once. Other peers learn of the peer through the mesh, not through the relay, and MUST NOT treat the withdrawal as the peer leaving the mesh. The closing of a signaling connection that was released on purpose MUST NOT tear down data channels, start recovery, or rotate to another relay.

A released peer MUST reopen the relay connection and announce again when:

* it has fewer than `min_mesh_peers` neighbours;
* it must signal a peer the mesh cannot reach, which includes the first message of a new negotiation;
* it needs discovery (`discover`) or an application asks it to;
* it has become an anchor.

After reopening, a peer SHOULD stay connected for at least the linger period (default 60 s) before releasing again. A message that caused the reopening need not be queued if the sender retries as in section 17.

A host that enforces a relay acknowledgement deadline MUST suspend it while the connection is released on purpose.

### 19.6 Anchors

If every peer in a room released its relay connection, a newcomer would find nobody. A few peers therefore stay registered: the **anchors**.

* The anchors of a room are the `anchors` peers (default 3) with the smallest distance to the room.
* The distance of a peer is `SHA-256(peer_id) XOR SHA-256(network + ":" + session_id)`, each digest read as a 256-bit unsigned big-endian integer and each input encoded as UTF-8. Ties are broken by lexicographic order of `peer_id`.
* A peer computes the anchors among itself, its neighbours, and the peers its neighbours have vouched for (19.7). It is an anchor when it is among them. Peers with the same view reach the same answer, and a peer closer to the room than the others it knows of learns of it through its neighbours.
* A peer that is not an anchor but has fewer than `min_mesh_peers` neighbours keeps its relay connection, so a small or damaged mesh is never left without one.
* Anchors are an availability measure, not an authority: a neighbour that lies about who is close can make a peer release early or hold on too long. A peer recovers from the first because its own neighbour count falling below `min_mesh_peers` reopens the relay.

### 19.7 Peer Channel Control Frames

A data channel between peers can carry control frames that belong to this protocol rather than to the application. A control frame is a JSON object whose `type` begins with `~`. Frame types that begin with `~` are reserved: an implementation MUST NOT deliver them to the application, and MUST ignore any it does not recognise.

Defined frames:

* `~signal` carries an envelope (19.4).
* `~anchors` tells a neighbour which peers the sender vouches for:

```json
{ "type": "~anchors", "ids": ["peer-a", "peer-c", "peer-d"] }
```

* The sender vouches for itself and for its own live neighbours only, never for a peer it has merely heard of. A peer that disappeared therefore stops being repeated as soon as the last peer connected to it notices, and no stale peer can keep itself alive by being passed around.
* A sender SHOULD send the frame when its set changes, and otherwise at least every half of the gossip TTL (default 20 s).
* A receiver keeps the set for the gossip TTL and then discards it. It MUST accept at most 16 ids of at most 256 characters each, and MUST NOT forward the frame.

Liveness of an edge is the peers' own concern and is confirmed on the channel, for example by `ping` and `pong` sent as data channel messages, not through a relay.

### 19.8 Relay Federation

No relay is required to be a hub. A relay MAY be configured with one or more seed relays; a relay joins the overlay through one seed at a time and learns the others from it, so any relay can be a seed. A relay MAY forward an addressed message toward the relay that holds the addressee. A relay that does not hold the addressee MUST say so with an `error` whose code is `unknown_peer` rather than drop the message silently, and a relay MUST NOT forward a message again once it has been forwarded, so that a forwarded message cannot loop.

---

## 20. Capability Negotiation

Capabilities are advisory unless explicitly required by an extension.

Common capability keys:

* `trickle_ice`
* `restart_ice`
* `datachannel`
* `media`
* `sdp_semantics`
* `can_relay_signal`
* `can_forward_data`
* `can_act_bootstrap`
* `mesh_signal` (the peer can carry negotiation messages over data channels, 19.4)
* `relay_on_demand` (the peer may release its relay connection, 19.5)
* `supports_signed_messages`
* `encoding`

Peers SHOULD intersect capabilities conservatively.

---

## 21. Topology Hints

Topology hints guide peer selection but do not change core interoperability.

Example hint object:

```json
{
  "max_peers": 8,
  "connected_peers": 3,
  "wants_peers": true,
  "preferred_degree": 6,
  "can_accept_inbound": true,
  "can_relay_signal": false,
  "peer_class": "edge"
}
```

Possible peer classes:

* `edge`
* `super`
* `relay`
* `bootstrap`
* `observer`

Possible desired link types:

* `direct`
* `relay-assisted`
* `backup`
* `control-only`

---

## 22. Minimal Interoperable Profile

A conforming minimal implementation MUST support:

* JSON envelope
* `announce`
* `discover` or equivalent discovery intake
* `offer`
* `answer`
* `ice_candidate`
* `bye`
* `error`
* `network`
* `peer_id`
* `session_id`
* trickle ICE or explicit no-trickle behavior

A minimal implementation MAY omit:

* signatures
* `connect_request`
* `peer_list`
* `redirect`
* `ack`
* extensions

---

## 23. Full Interoperable Profile

A full implementation SHOULD support:

* all core discovery messages
* all negotiation messages
* message signing
* retry and deduplication
* glare handling
* liveness with `ping`/`pong`
* re-announcement and expiry
* topology hints
* redirect and alternate peer suggestions
* mesh-carried signaling (19.4), relay on demand with anchors (19.5, 19.6) and peer channel control frames (19.7)

---

## 24. Example Full Handshake

## 24.1 Discovery

`peer-a` announces.

```json
{
  "psp_version": "1.0",
  "type": "announce",
  "network": "room-7",
  "from": "peer-a",
  "message_id": "m1",
  "timestamp": 1775030000000,
  "ttl_ms": 30000,
  "body": {
    "instance_id": "a1",
    "roles": ["peer"],
    "capabilities": {
      "trickle_ice": true,
      "datachannel": true
    },
    "hints": {
      "max_peers": 6,
      "connected_peers": 2,
      "wants_peers": true
    }
  }
}
```

## 24.2 Candidate Discovery

`peer-a` asks bootstrap for peers.

```json
{
  "psp_version": "1.0",
  "type": "discover",
  "network": "room-7",
  "from": "peer-a",
  "to": "bootstrap-1",
  "message_id": "m2",
  "timestamp": 1775030000100,
  "body": {
    "limit": 8
  }
}
```

Bootstrap returns `peer-c`.

## 24.3 Preflight

`peer-a` asks to connect.

```json
{
  "psp_version": "1.0",
  "type": "connect_request",
  "network": "room-7",
  "from": "peer-a",
  "to": "peer-c",
  "session_id": "s1",
  "message_id": "m3",
  "timestamp": 1775030000200,
  "body": {
    "intent": "webrtc-datachannel",
    "initiator_preference": "auto"
  }
}
```

`peer-c` accepts.

## 24.4 Offer / Answer

`peer-a` sends `offer`, `peer-c` sends `answer`.

## 24.5 Trickle

Both sides exchange `ice_candidate` and optionally `ice_end`.

## 24.6 Connected

The signaling session is considered successful when the underlying WebRTC transport reports connected.

## 24.7 Close

Either side sends `bye`.

---

## 25. Conformance Requirements

### MUST

* support the base envelope
* correlate negotiation messages with `session_id`
* reject wrong-network messages
* safely ignore unknown optional fields
* provide deterministic glare handling
* process offer/answer/candidate messages correctly

A peer that releases its relay connection (19.5) MUST also:

* be able to carry negotiation messages over the mesh and accept them (19.4)
* keep its data channels open across the release, and not recover or rotate relays because of it
* reopen the relay connection in each case 19.5 lists
* ignore `~`-prefixed control frames it does not recognise, and never deliver them to the application (19.7)

### SHOULD

* support announcements with TTL
* retry important messages on lossy transports
* deduplicate duplicates
* support signing where relays are untrusted
* expose topology hints

### MAY

* use binary encodings
* use app-specific extensions
* implement alternate discovery mechanisms

---

## 26. Recommended Default Values

* `ttl_ms` for announce: `30000`
* announce refresh interval: `10000` to `15000`
* message replay window: `60000`
* connection request expiry: `10000`
* retry base delay: `1000`
* max retries: `5`
* peer discovery limit default: `16`
* relay on demand: `min_mesh_peers` `2`, `anchors` `3`, settle `30000`, linger `60000`
* anchor evaluation interval: `5000`
* anchor gossip TTL: `20000`

---

## 27. Implementation Notes

1. **Do not trust signaling transport ordering.**
   Candidates may arrive before offer/answer processing completes.

2. **Do not assume one peer process equals one stable instance.**
   Use `instance_id`.

3. **Expect duplicates.**
   Especially on gossip or relay fanout substrates.

4. **Expect stale announcements.**
   TTL enforcement matters.

5. **Keep signaling separate from app messaging.**
   Once WebRTC is up, app traffic should move off the signaling substrate if possible.

6. **Keep the relay out of an established mesh.**
   Re-announcements, acknowledgements and liveness checks against a relay are costs that a healthy mesh does not need. Carry negotiation over the mesh (19.4), judge edges on the edge, and release the relay connection when the mesh can stand without it (19.5).

---

## 28. Future Extension Areas

Reserved areas include:

* NAT classification hints
* bandwidth class hints
* cryptographic suite negotiation
* end-to-end encrypted signaling payloads
* DHT advertisement compression
* multiplexed multi-session negotiation
* delegated relay sponsorship
* peer reputation and trust scoring

---

# Compact Message Schema Summary

## Base Envelope

```json
{
  "psp_version": "1.0",
  "type": "string",
  "network": "string",
  "from": "string",
  "to": "string|null",
  "session_id": "string|null",
  "message_id": "string",
  "reply_to": "string|null",
  "timestamp": 0,
  "ttl_ms": 0,
  "body": {},
  "sig": null
}
```

---

# Recommended Initial Message Set for a Real Implementation

If you want this to be simple enough to ship first, implement only:

* `announce`
* `discover`
* `peer_list`
* `offer`
* `answer`
* `ice_candidate`
* `bye`
* `error`
