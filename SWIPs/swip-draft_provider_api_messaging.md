---
SWIP: <to be assigned>
title: Swarm Provider API — Messaging Extension (PSS & GSOC)
author: Florian Glatz (@heckerhut)
discussions-to: https://discord.com/channels/799027393297514537/1239813439136993280
status: Draft
type: Standards Track
category: Interface
created: 2026-07-13
requires: <Swarm Provider API SWIP>
---

## Simple Summary

An extension to the [Swarm Provider API (`window.swarm`)](./swip-draft_provider_api.md)
that adds **real-time messaging**: encrypted point-to-point delivery (PSS) and
topic broadcast (GSOC), including a **subscription primitive** for receiving
pushed messages — the first streaming surface in the provider API, which is
otherwise entirely request/response.

## Abstract

The core Swarm Provider API specifies request/response RPC methods and two
lifecycle events (`connect`, `disconnect`). It has no way for a web
application to *receive* asynchronous data: every method resolves once, and the
only push signals concern the connection itself. Real-time applications —
collaborative editors, chat, presence, live dashboards — need a channel over
which the node delivers incoming messages as they arrive.

This extension adds that channel. It specifies:

- **`swarm_subscribe` / `swarm_unsubscribe`** — open and close a long-lived
  subscription to a GSOC address (topic broadcast) or a PSS topic
  (point-to-point inbox). Delivery is via a new **`message`** event, borrowing
  the subscription-id + event pattern of
  [EIP-1193](https://eips.ethereum.org/EIPS/eip-1193)'s `eth_subscribe` (with
  a flatter envelope; see [Events](#events)).
- **`swarm_sendPss`** — send an encrypted point-to-point message to a recipient
  identified by their PSS public key.
- **`swarm_sendGsoc`** — broadcast a message on a topic that any subscriber to
  the derived GSOC address receives.
- **`swarm_getMessagingIdentity`** — disclose the PSS public key and routing
  target that peers use to address messages to this origin.

PSS and GSOC are existing Swarm/Bee primitives; this extension does not define
them, only their exposure at the provider boundary with consent and
origin-scoping consistent with the core specification. Messages are best-effort,
unordered, and may be duplicated; applications requiring ordering or exactly-once
semantics MUST layer that themselves (a CRDT such as Yjs satisfies this
naturally).

## Motivation

The core API's request/response shape is a deliberate, minimal foundation, but
it structurally cannot express "tell me when something arrives." Applications
work around this by polling — repeatedly reading a feed or SOC on a timer —
which trades latency for request volume and never reaches interactive speed. A
collaborative document synced by polling feeds every few seconds is usable for
asynchronous editing but cannot show live cursors or keystroke-level updates.

Swarm already has the transport primitives for real-time messaging:

- **PSS** (Postal Service over Swarm): trojan chunks encrypted to a recipient's
  public key and routed toward a neighborhood — encrypted, sender-anonymous,
  point-to-point.
- **GSOC** (Graffiti Single Owner Chunk): a Single Owner Chunk at a
  deterministically-derivable address that many parties can write to and
  subscribe to — a broadcast "mailbox at a well-known address."

Historically these required a full node to *receive* (Bee has no push delivery
to light nodes). Light-node implementations can now receive by pulling from the
target neighborhood and decrypting locally, making send **and** receive viable
on the kind of light/ultra-light node a browser embeds. This extension exposes
that capability to dApps so the decentralized web gains real-time collaboration
without a coordination server.

### Design Goals

Identical to the core specification, plus:

- **One new interaction pattern, minimally.** Add exactly one streaming shape
  (subscription-id + `message` event), reusing EIP-1193 precedent, rather than
  several bespoke callback styles.
- **Hide protocol mechanics.** DApps address a *topic* or a *recipient key*; the
  provider handles trojan construction, neighborhood targeting, GSOC owner
  mining, nonce mining, encryption, and the receive pipeline.
- **Tolerate an unreliable substrate honestly.** The API surfaces best-effort
  delivery — messages may be lost, reordered or duplicated — as such and does
  not pretend otherwise.

## Specification

This document is an **extension**. It adds methods, one event, error reasons,
capability fields, a permission tier and a permission manifest capability to
the core Swarm Provider API. It changes none of the existing method semantics.
An implementation MAY conform to the core specification without this
extension; an implementation of this extension MUST also conform to the core
specification.

### Feature Detection

Because this extension is optional, dApps MUST NOT assume its presence from the
core `specVersion` alone. An implementation of this extension MUST include the
string `"messaging"` in the core `swarm_getCapabilities().features` array:

```javascript
const caps = await window.swarm.getCapabilities()
if (caps.features?.includes('messaging')) {
  // PSS/GSOC methods are available
}
```

`swarm_getCapabilities` is additionally extended with messaging limits (see
[Limits](#limits)). Both the feature string and the limit fields MUST be
available before `swarm_requestAccess` — the core spec makes
`swarm_getCapabilities` permission-free, and these are static capability facts,
not user data.

### Convenience Methods

Implementations of this extension MUST expose these convenience wrappers, in
addition to the core set:

```javascript
window.swarm.getMessagingIdentity()                               // swarm_getMessagingIdentity
window.swarm.subscribe({ kind, topic | address, history })        // swarm_subscribe
window.swarm.unsubscribe({ subscriptionId })                      // swarm_unsubscribe
window.swarm.sendPss({ topic, recipient, targets, data })         // swarm_sendPss
window.swarm.sendGsoc({ topic, data })                            // swarm_sendGsoc
```

Each MUST be equivalent to calling `window.swarm.request()` with the
corresponding method name and params.

### Events

This extension adds one standard event:

| Event | Data | Description |
|---|---|---|
| `message` | `SubscriptionMessage` (below) | Emitted for each message delivered on any active subscription owned by the calling page. |

**`SubscriptionMessage`:**

```javascript
{
  type: "swarm_subscription",   // constant; disambiguates from other future event payloads
  subscription: string,          // the subscriptionId this message belongs to
  result: {
    kind: "gsoc" | "pss",       // matches the subscription kind
    key: string,                 // GSOC: the 64-hex SOC address; PSS: the 64-hex topic
    data: string,                // base64-encoded decrypted payload
    encoding: "base64",
    receivedAt: number           // ms unix timestamp the provider received it
  }
}
```

The `message` event MUST fire only for subscriptions the receiving page
created (origin-scoped, same isolation as feeds and tags). An implementation
MUST NOT deliver a message to a page of a different origin than the one that
opened the corresponding subscription, including after that tab has navigated
elsewhere.

The envelope borrows EIP-1193's subscription pattern but not its exact shape:
EIP-1193 nests `subscription` and `result` under `data`, while here they sit at
the top level next to `type`. Handlers written for EIP-1193 subscriptions need
a small adapter.

Consistent with EIP-1193, `swarm.on('message', handler)`,
`swarm.removeListener('message', handler)`, and `swarm.removeAllListeners('message')`
MUST be supported.

### Error Format

Unchanged from the core specification. This extension reuses the standard error
codes and adds the following `data.reason` values.

**`4100`:**

| Reason | Meaning |
|---|---|
| `messaging_not_granted` | A messaging-tier method without the messaging grant, where no prompt could acquire it. |

**`4900`** (all retryable):

| Reason | Applicable Methods | Meaning |
|---|---|---|
| `node_subscription_limit` | `swarm_subscribe` | The node's aggregate receive capacity is exhausted, possibly by other origins (see below). |
| `establish_timeout` | `swarm_subscribe` | The subscription did not establish within the implementation's bound. |
| `subscription_cancelled` | `swarm_subscribe` | The subscription was torn down (navigation, tab close, grant revoked) before it was established. |

`swarm_sendPss` and `swarm_sendGsoc` also report the core setup reasons
(`node-stopped`, `node-not-ready`, `ultra-light-mode`, `no-usable-stamps`), as
any write does.

**`-32602`:**

| Reason | Applicable Methods | Meaning |
|---|---|---|
| `invalid_params` | All messaging methods | Generic shape error, e.g. both or neither of `topic` and `address`, or an empty `subscriptionId`. |
| `invalid_topic` | `swarm_subscribe`, `swarm_sendPss`, `swarm_sendGsoc` | Topic is missing, empty, too long, or contains control characters. |
| `invalid_address` | `swarm_subscribe`, `swarm_sendGsoc` | An explicit GSOC `address` is not a valid 64-char hex (subscribe), or was supplied at all (send; see `swarm_sendGsoc`). |
| `invalid_recipient` | `swarm_sendPss` | Recipient public key is missing or not a valid 33-byte compressed secp256k1 key (66-char hex). |
| `invalid_target` | `swarm_sendPss` | `targets` is missing, not valid hex, or outside 2–`maxTargetDepth` whole bytes (a 1-byte target is too shallow to land in the recipient's neighborhood — see [PSS mining depth](#pss-mining-depth-prefix-length)). |
| `invalid_kind` | `swarm_subscribe` | `kind` is not `"gsoc"` or `"pss"`. |
| `payload_too_large` | `swarm_sendPss`, `swarm_sendGsoc` | Payload exceeds `maxMessageBytes`. |
| `invalid_payload` | `swarm_sendGsoc` | Payload is empty. A GSOC update is a single-owner chunk and bee's chunk layer rejects an empty SOC payload, so an empty broadcast cannot be expressed. (`swarm_sendPss` **does** accept an empty payload — the PSS trojan framing carries an explicit length.) |
| `too_many_subscriptions` | `swarm_subscribe` | The origin has reached `maxSubscriptions`. `data.limit` gives the cap. |
| `rate_limited` | `swarm_subscribe`, `swarm_sendGsoc` | The origin has started too many new GSOC topics recently (see [Limits](#limits)). `data.retryAfterMs` says when to retry. |
| `unsupported_option` | `swarm_subscribe`, messaging methods accepting `options` | An unrecognized option field was supplied, or `history: true` on an implementation without the mailbox. `data.option` names it. |

**`-32603`:**

| Reason | Applicable Methods | Meaning |
|---|---|---|
| `gsoc_mining_timeout` | `swarm_subscribe`, `swarm_sendGsoc` | Deriving the GSOC signer for a topic ran past the implementation's bound. |
| `gsoc_mining_failed` | `swarm_subscribe`, `swarm_sendGsoc` | Deriving the GSOC signer failed. |

**Failures after establishment.** A subscription whose underlying receive
pipeline fails after being established (e.g. the node restarts or the
connection to it drops) has no per-subscription error signal in this revision —
the `SubscriptionMessage` schema deliberately has no error variant, and
normative mid-stream error signalling is deferred to a future revision (see
[Future Work](#future-work)). Implementations SHOULD re-establish the pipeline
automatically, with backoff, for as long as the subscription is open; messages
sent while it is down MAY be lost. An implementation that instead ends such a
subscription MUST document that. Where the failure is node-wide, the core spec's
`disconnect` event does not apply (the origin's grants are unchanged); pages
that need liveness guarantees SHOULD pair messaging with feed reads (see
[Delivery semantics](#delivery-semantics-normative)).

---

### Methods

#### `swarm_getMessagingIdentity`

Return the messaging identity peers use to reach this origin — the PSS public
key they encrypt to, and the neighborhood-prefix target they route toward.

Operates under the **messaging permission tier** (see [Permission Model](#permission-model)):
the returned public key is a stable identifier, so disclosure requires consent,
mirroring `swarm_getSigningIdentity` in the core spec.

**Params:** None.

**Result:**

```javascript
{
  pssPublicKey: string,   // 66-char hex, 33-byte compressed secp256k1 public key
  pssTarget: string,      // hex neighborhood prefix (2..maxTargetDepth bytes); peers pass it as `targets` in swarm_sendPss
  identityMode: string    // "app-scoped" if the key is this origin's own, "bee-wallet" if it is the node's
}
```

**Errors:** `4001` if the user rejects the messaging-permission prompt. `4100`
if the origin lacks basic connection or a prompt cannot be surfaced. `4900` if
the node is unavailable.

**Behavior:**
- If messaging permission has not been granted, the implementation MUST prompt
  (same tier as the first `swarm_subscribe`/`swarm_send*` call) and return on
  approval. Once granted, the method does not prompt again.
- The `pssPublicKey` MUST be stable for the lifetime of the origin's messaging
  grant.
- **Per-origin or node key.** Implementations SHOULD give each origin its own
  PSS key (`identityMode: "app-scoped"`), so one dApp cannot present, or
  receive, another's messages. Bee and current light-node implementations
  decrypt PSS only with the node's own key, though, so an implementation MAY
  return the node's key (`identityMode: "bee-wallet"`), which is then the same
  for every origin. The consequences are serious enough that such an
  implementation MUST tell the user when it asks for the messaging grant; see
  [Shared PSS key](#shared-pss-key).
- Whether the PSS key is bound to the same key as `swarm_getSigningIdentity`'s
  `owner` is implementation-defined; dApps MUST NOT assume equality.
- `pssTarget` MUST be a truncated neighborhood prefix, **at least 2 and at most
  `maxTargetDepth` bytes**, NOT the node's full overlay address. Two bounds, two
  reasons:
  - *Upper* (privacy): the full 64-hex overlay is a node-global value, identical
    for every origin, that would function as a cross-origin correlation handle
    and disclose the node's exact network position (see
    [Security Considerations](#security-considerations)).
  - *Lower* (deliverability): the prefix length is the number of leading bits a
    sender's trojan chunk is mined to match, and it must be **deep enough that
    storers in the recipient's neighborhood keep the chunk** — a prefix
    shallower than the network's storage depth produces chunks that are stored
    in some other neighborhood, where the recipient does not look, so the
    message is, in all likelihood, undeliverable. It also governs the receiver's pull cost. See
    [PSS mining depth](#pss-mining-depth-prefix-length).

---

#### `swarm_subscribe`

Open a long-lived subscription. Incoming messages are delivered via the
[`message` event](#events), tagged with the returned `subscriptionId`.

**Params:**

| Field | Type | Required | Description |
|---|---|---|---|
| `kind` | `"gsoc" \| "pss"` | Yes | Subscription type. |
| `topic` | `string` | Conditional | GSOC: a topic string the provider deterministically maps to a GSOC address. PSS: the topic to receive on. Required unless a raw `address` is given for GSOC. |
| `address` | `string` | Conditional | GSOC only. A raw 64-char hex GSOC/SOC address, for interoperating with GSOCs whose address was derived elsewhere. Mutually exclusive with `topic`. Refused for PSS. |
| `history` | `boolean` | No | PSS only, OPTIONAL feature. When `true`, on subscribe the provider attempts to deliver messages that arrived *before* the subscription — a **mailbox** — recovering offline messages, subject to storer retention and an implementation-defined bound (see [Delivery semantics](#delivery-semantics-normative)). Default `false` (live traffic only). Ignored for GSOC (a SOC has a single latest value, not a message backlog). |
| `options` | `object` | No | Reserved. Unknown fields rejected with `unsupported_option`. |

**Result:**

```javascript
{
  subscriptionId: string,   // opaque id, unique within this origin's session
  kind: "gsoc" | "pss",
  key: string               // resolved GSOC address (gsoc) or topic hash (pss), 64-char hex
}
```

**Errors:** `4001` if the user rejects the messaging-permission prompt this call
triggered. `4100` if the origin lacks basic connection or a prompt cannot be
surfaced. `4900` if the node is unavailable, **or if the node's aggregate
subscription capacity is exhausted** (`node_subscription_limit`, see below), or
the subscription did not establish (`establish_timeout`,
`subscription_cancelled`). `-32602` for `invalid_kind`, `invalid_topic`,
`invalid_address`, `invalid_params`, `too_many_subscriptions`, `rate_limited`,
`unsupported_option`. `-32603` for `gsoc_mining_timeout`, `gsoc_mining_failed`.

**Per-origin cap vs. node capacity — two distinct failures.**
`too_many_subscriptions` (`-32602`) means *this origin* has reached
`maxSubscriptions`: the dApp caused it and can fix it by unsubscribing. But a
receive pipeline is a **node-wide** resource — an implementation typically
bounds the number of *distinct neighborhoods* it will pull concurrently across
**all** origins (the reference light-node implementation allows 8), and that
pool can be exhausted by *other* origins' subscriptions. That case MUST be
reported as `4900` (`node_subscription_limit`), NOT as
`too_many_subscriptions`: it is not a parameter error, the calling dApp did
nothing wrong, and it is **retryable** — capacity frees when other
subscriptions close. DApps SHOULD treat it like any other transient node
condition (back off and retry, or degrade to feed polling) rather than as a
permanent refusal.

**Behavior:**
- Subscribing establishes a receive pipeline for the target neighborhood. The
  implementation is responsible for pulling and decrypting matching chunks and
  delivering decoded payloads via the `message` event. The call resolves once
  the node has accepted the subscription; implementations MAY bound that wait
  and fail with `establish_timeout`.
- **History.** The mailbox is OPTIONAL. An implementation that does not offer
  it MUST reject `history: true` with `-32602`, `data.reason =
  "unsupported_option"` and `data.option = "history"`, so a page can tell it
  will not receive missed messages; `history: false` or absent is always
  accepted.
- **GSOC topic → address derivation MUST be deterministic within an
  implementation**: any two origins passing the same `topic` to the same
  provider MUST resolve to the same address, whether subscribing or sending.
  The derivation MUST be documented by the implementation. See
  [GSOC topic derivation](#gsoc-topic-derivation) for an example.
- **Cross-implementation convergence is NOT guaranteed in this revision.**
  Determinism alone does not pin the address: implementations must additionally
  agree on the identifier derivation (topic → 32-byte SOC identifier), the
  target-neighborhood derivation (topic → overlay prefix), the mining algorithm
  and its search order (e.g. bee-js `gsocMine`), and the required proximity —
  a difference in any one yields a different address for the same topic. Until
  a normative derivation profile exists (see [Future Work](#future-work)),
  applications interoperating across different providers MUST exchange the
  resolved GSOC `address` (returned by both `swarm_subscribe` and
  `swarm_sendGsoc`) out of band and use the raw `address` parameter — for
  **receiving only**. Sending requires the mined owner key, which only the
  topic derivation yields, so cross-provider *senders* must share the topic
  itself (see `swarm_sendGsoc` Behavior).
- For PSS, only messages the node can decrypt with the key
  `swarm_getMessagingIdentity` reports for this origin are delivered.
  Undecryptable traffic in the neighborhood MUST NOT be delivered. Where that
  key is the node's, this includes messages other origins' peers sent to the
  node on the same topic (see [Shared PSS key](#shared-pss-key)).
- **Duplicates.** The same message MAY be delivered
  more than once, also within one subscription. Implementations MAY suppress a
  redelivery of the same chunk, but MUST NOT suppress a message only because
  its payload equals an earlier one's: a genuine repeat (a second "ok", an
  empty keep-alive) and a redelivery look the same at that level, and dropping
  one would silently swallow the other. Applications that need exactly-once
  semantics MUST carry their own message id.
- Subscriptions are **origin-scoped** and **page-scoped**. They MUST be torn
  down automatically when the page's document is unloaded or replaced
  (navigation to a new document, tab close) and when the origin's messaging
  grant or connection is revoked. Same-document navigations (fragment changes,
  `history.pushState`) MUST NOT end them. Implementations are NOT REQUIRED to
  persist subscriptions across page reloads.
- Multiple subscriptions to the same key — by one origin or several — MAY be
  coalesced internally onto one node connection, but each MUST receive
  messages and MUST be independently addressable by `swarm_unsubscribe`.
- Subscriptions still being established count toward `maxSubscriptions`.

<a id="delivery-semantics-normative"></a>**Delivery semantics (normative):** Messages are **best-effort**: a message
MAY be lost (for example while a pipeline is re-establishing), MAY arrive out
of send order, and MAY arrive more than once. Without `history`, a subscriber
also misses messages published while it was not subscribed. Applications MUST tolerate this.
Ordering/causality and gap recovery are application concerns; the durable
Swarm feed layer (core spec) is the appropriate substrate for authoritative
state and catch-up, with messaging used for live propagation.

**With `history: true` (mailbox):** an implementation that supports the mailbox
additionally delivers, on subscribe, messages that arrived before the
subscription — offline delivery. This is **bounded and best-effort**, not a
durable queue: recovery is limited by (a) storer **retention** — a message
whose postage stamp has expired, or that has been evicted from neighborhood
reserves, is unrecoverable; and (b) an **implementation-defined lookback
bound** the provider MAY impose to keep the recovery sweep cheap. Recovered
messages carry the same `message`-event shape as live ones and are subject to
the same best-effort, may-duplicate semantics; `receivedAt` reflects when the
*provider* recovered the message, not when it was originally sent (a
sender-supplied timestamp, if wanted, is application payload). Whether
historical messages are ordered relative to each other is
implementation-defined; applications needing send-order MUST carry their own
sequence. How far back the mailbox reaches is implementation-defined and, at
the default 2-byte prefix, is a **recent-history window** rather than the
complete backlog (a light-node receiver sweeps a busy neighborhood bin, so a
bounded lookback covers minutes-to-hours, not all of history). A complete
backlog sweep would require the trojan concentrated into a sparse deep bin
pulled by a deeply-resident receiver (see
[PSS mining depth](#pss-mining-depth-prefix-length) for why that trade is not
the default).

---

#### `swarm_unsubscribe`

Close a subscription.

**Params:**

| Field | Type | Required | Description |
|---|---|---|---|
| `subscriptionId` | `string` | Yes | An id returned by `swarm_subscribe` for this origin. |

**Result:**

```javascript
{ unsubscribed: true }
```

**Errors:** `4100` if the origin is not connected. `-32602` (`invalid_params`)
if `subscriptionId` is missing or not a non-empty string.

**Behavior:**
- Needs the connection grant only; never prompts.
- **Idempotent.** The method MUST resolve `{ unsubscribed: true }` for any
  well-formed id, including one that is already closed, never existed, or
  belongs to another origin — and MUST answer all of these the same way, so a
  page cannot probe for other origins' subscription ids.
- After a successful unsubscribe, the `message` event MUST NOT fire for that
  `subscriptionId`.
- The provider SHOULD release the underlying receive pipeline when its last
  subscriber for a neighborhood unsubscribes.

---

#### `swarm_sendPss`

Send an encrypted point-to-point message to a recipient identified by their PSS
public key. The message is a trojan chunk encrypted to the recipient and routed
toward the neighborhood implied by `targets`.

Operates under the **messaging tier**, with per-send consent like publish; each
send consumes postage.

**Params:**

| Field | Type | Required | Description |
|---|---|---|---|
| `topic` | `string` | Yes | Topic the recipient subscribes on. Hashed/normalized by the provider as in Bee PSS. |
| `recipient` | `string` | Yes | Recipient PSS public key: 66-char hex, 33-byte compressed secp256k1 (as returned by `swarm_getMessagingIdentity`). MAY carry a `0x` prefix. |
| `targets` | `string` | Yes | Neighborhood prefix as hex (**2**–`maxTargetDepth` bytes) locating the recipient — the value the recipient's `swarm_getMessagingIdentity` returned as `pssTarget`, shared out of band alongside their public key. The sender's provider cannot derive this from the public key alone. The prefix length sets the trojan's mining depth (see [PSS mining depth](#pss-mining-depth-prefix-length)): it MUST be deep enough that neighborhood storers keep the chunk, and deeper targets cost more to mine but less to receive. |
| `data` | `string \| Uint8Array \| ArrayBuffer` | Yes | Payload. Strings encoded UTF-8. MUST be ≤ `maxMessageBytes`. MAY be empty — the PSS trojan framing carries an explicit length, so a zero-byte message is well-formed (useful for pings/signals). |
| `options` | `object` | No | Reserved. Unknown fields rejected with `unsupported_option`. |

**Result:**

```javascript
{ sent: true }
```

**Errors:** `4001` if the user rejects a per-send prompt. `4100` if not
connected. `4900` if node unavailable, not ready to write, or without a usable
postage stamp (core setup reasons). `-32602` for `invalid_topic`,
`invalid_recipient`, `invalid_target`, `payload_too_large`,
`unsupported_option`.

**Behavior:**
- The provider constructs the trojan chunk (encryption to `recipient`, nonce
  mining, targeting) and uploads it with an automatically-selected postage
  batch. DApps do not manage batches or mining.
- PSS is **fire-and-forget**: a successful result means the chunk was accepted
  for upload, NOT that the recipient received it. There is no delivery receipt.
- Payload confidentiality is provided by PSS encryption to the recipient key.
  Sender anonymity properties are those of the underlying PSS primitive and are
  not strengthened or weakened here.

---

#### `swarm_sendGsoc`

Broadcast a message on a topic. Any origin subscribed (via `swarm_subscribe`
with `kind: "gsoc"`) to the address the topic derives to receives it.

Operates under the **messaging tier**, with per-send consent like publish.

**Params:**

| Field | Type | Required | Description |
|---|---|---|---|
| `topic` | `string` | Yes | Topic the provider deterministically maps to a GSOC address (same derivation as `swarm_subscribe`). Unlike `swarm_subscribe`, there is no raw `address` alternative — see Behavior. |
| `data` | `string \| Uint8Array \| ArrayBuffer` | Yes | Payload. Strings encoded UTF-8. MUST be **1**..`maxMessageBytes` bytes: an empty payload is rejected with `invalid_payload` (a GSOC update is a single-owner chunk, and bee's chunk layer refuses an empty SOC payload — the provider must not sign a chunk the network will not accept). |
| `options` | `object` | No | Reserved. Unknown fields rejected with `unsupported_option`. |

**Result:**

```javascript
{
  sent: true,
  address: string   // 64-char hex GSOC address written to
}
```

**Errors:** `4001` if the user rejects a per-send prompt. `4100` if not
connected. `4900` if node unavailable, not ready to write, or without a usable
postage stamp. `-32602` for `invalid_topic`, `invalid_address`,
`invalid_payload`, `payload_too_large`, `rate_limited`, `unsupported_option`.
`-32603` for `gsoc_mining_timeout`, `gsoc_mining_failed`.

**Behavior:**
- GSOC send writes a Single Owner Chunk at the derived address. The provider
  performs any required owner mining so the chunk lands in the correct
  neighborhood, signs it, and uploads it with an automatically-selected batch.
  The first use of a topic mines its owner key, which can take a noticeable
  fraction of a second; implementations MAY cache derivations, MAY bound the
  mining time, and MAY limit how many new topics an origin starts (see
  [Limits](#limits)).
- **There is deliberately no raw `address` variant** (asymmetric with
  `swarm_subscribe`, which accepts one). An `address` param MUST be rejected
  with `invalid_address`. Writing a GSOC requires signing with the mined owner
  key, and that key is recovered by re-running the derivation from the topic; a
  GSOC address is a hash output (`keccak256(identifier ‖ owner)`) from which
  neither the identifier nor the owner key can be recovered. An address alone
  therefore carries nothing to sign with — watching an address is passive,
  authoring at one is not. Senders interoperating across providers MUST share
  the topic (and, until a normative derivation profile exists, use the same
  provider implementation or agree on the derivation out of band).
- Re-sending on the same topic writes a new update at the same address (this
  is how a broadcast channel advances). Because delivery is best-effort,
  subscribers MUST NOT assume they observe every update — only that they tend
  to see recent ones. A GSOC topic is therefore a good *notification* channel
  and a poor log. For a stream of independent updates where every value
  matters (e.g. CRDT deltas), applications SHOULD keep the authoritative
  updates in feeds, and use GSOC on a fixed topic (or PSS) only to carry them
  live or to signal that new entries exist, with the feeds as the catch-up
  path. PSS is best-effort too, so it is no substitute for durable storage
  unless the application adds its own retransmission. Changing the topic per
  message does not help: each topic
  is a different address, and a subscriber receives only the addresses it
  subscribed to, so it would have to know every future topic in advance.
  Implementations SHOULD document their GSOC addressing so applications can
  reason about this.
- Unlike the core `swarm_writeSingleOwnerChunk` (signed by the origin's
  publisher identity), GSOC uses a mined owner key derived for neighborhood
  placement; this is why GSOC send is a distinct method rather than a use of
  the core SOC write.

---

### PSS mining depth (prefix length)

A PSS message is a *trojan chunk* whose content-address is **mined** to share a
number of leading bits — the **prefix length**, `L` bits = `targets`/`pssTarget`
byte-count × 8 — with the recipient's overlay, so the network routes it into the
recipient's neighborhood. `L` is not a free knob; it is bounded on both sides,
which is why this spec constrains `targets`/`pssTarget` to **2–`maxTargetDepth`
bytes** rather than the underlying primitive's 1-byte floor.

**Lower bound — landing where the recipient looks.** Every chunk is stored by
the neighborhood its address falls in: the storers whose overlay shares at
least the storage depth `d` leading bits with it (`d` is the proximity order at
which reserves are maintained). Mining guarantees only `L` shared bits with the
recipient's overlay, so a trojan mined to `L < d` lands, with probability about
`1 − 2^(L−d)`, in some *other* neighborhood. It is stored there, but the
recipient pulls its own neighborhood and never sees it: the send returns
success (the chunk was accepted for upload), yet the message is, in all
likelihood, undeliverable. On the
public network `d` is on the order of a dozen bits and rises with network growth;
a 1-byte (`L=8`) target is below it, which is why 1-byte targets are rejected. A
2-byte (`L=16`) target is comfortably above `d` today and is the adopted default.

**Upper bound — mining cost.** Mining is `~2^L` hashes, so each extra byte is
~256× the sender work: a 2-byte mine is sub-second, a 3-byte mine is seconds on a
constrained (mobile) sender and can exceed a send timeout. `maxTargetDepth` caps
the deep end.

**Reception cost — and why deeper does not help a light node.** A receiver pulls
candidate chunks from the covering peers nearest the target and tries to decrypt
them. It is tempting to think a deeper `L` cuts this: on a depth-`d` storer the
chunks at proximity `≥ L` number roughly `2^(reserve_bits − (L − d))`, so
concentrating the trojan deeper shrinks that set. But a receiver only realizes
that if it pulls the deep bin — which requires a covering peer at proximity
`≥ L`, i.e. the receiver itself being **deeply resident** in the target
neighborhood. A light node's covering peers sit shallower than `L`, so it pulls
the *same* bins and downloads the *same* candidates regardless of `L` — measured
identical at `L=16` and `L=24` on a reference light node. For the light-node
deployments this extension targets, a deeper prefix therefore adds sender cost
without reducing reception cost, and the mailbox is a recent-history window at
any `L`. (A deeply-resident receiver — a near-full node in the neighborhood —
*would* benefit, and could privately agree a deeper `L` with its senders.)

**Convention — `L = 16` (2-byte targets), with a `2..maxTargetDepth` range.**
Interoperating sender and receiver implementations MUST agree on `L`: a receiver
pulls the bins a message at prefix `L` can occupy, so a sender mining to a
different `L` may land where the receiver is not looking. This SWIP adopts
**16 bits (2 bytes)** as the interoperability default:

- Senders SHOULD mine `targets` to 2 bytes; receivers assume `L = 16`.
- 2 bytes is also the hard `invalid_target` floor (below the network storage
  depth a chunk does not land where the recipient looks); 1 byte is always
  rejected.
- Implementations MAY mine deeper (up to `maxTargetDepth`) where sender and
  receiver privately agree, but a conforming default receiver assumes 16.

**Why not deeper.** A deeper prefix concentrates a trojan into a smaller slice of
the reserve, which *in principle* cuts a receiver's candidate traffic and shrinks
the recoverable mailbox backlog. But that benefit accrues only to a receiver
**deeply resident** in the target neighborhood (covering peers at proximity
`≥ L`); a light node's covering peers sit shallower, so it pulls the same bins
and downloads the same candidates regardless of `L` — measured identical at
`L = 16` and `L = 24` on a reference light node. Meanwhile mining is `~2^L`
hashes, so each extra byte is ~256× the sender work: `L = 24` is seconds on a
constrained (mobile) sender and can exceed a send timeout. Since PSS already
carries an economic anti-spam gate — every send burns a **postage stamp** — the
extra proof-of-work buys a light-node deployment little and costs its senders a
lot. Hence the 2-byte default.

**Open question (protocol-level).** Whether the network *should* mandate a deeper
prefix as a deliberate anti-spam / anti-free-riding **proof-of-work** on senders
— distinct from postage's economic gate — is a live protocol-incentive question
raised by the Swarm protocol designer and still under discussion. It governs
storage and retrieval for *every* PSS participant, not just this provider
surface, so its normative home is arguably the core Swarm protocol
documentation, not this extension; this SWIP fixes the provider default at 16 and
will track any core-level decision.

---

### GSOC topic derivation

This section is **informative**. As explained under `swarm_subscribe`, this
revision does not fix how a topic maps to a GSOC address; each implementation
documents its own. As an example, Freedom Browser derives it as follows:

```
identifier    = keccak256(utf8(topic))
targetOverlay = keccak256(utf8("freedom-gsoc-v1:" + topic))
signer        = gsocMine(targetOverlay, identifier, 12)   // bee-js; deterministic sequential search
address       = keccak256(identifier ‖ signer.address)
```

`gsocMine` searches keys in a fixed order until the SOC address shares 12
leading bits with `targetOverlay`, so every participant that runs the same
derivation on the same topic converges on the same signer and address. Any
change to a constant here moves every existing room to a new address, which is
why an implementation treats its derivation as frozen once shipped, and why a
future normative profile will have to pick one derivation for everyone.

---

### Limits

`swarm_getCapabilities().limits` is extended with:

| Limit | Recommended Default | Notes |
|---|---|---|
| `maxMessageBytes` | 4000 | Max PSS/GSOC payload. A Swarm chunk is 4096 bytes; PSS/GSOC framing and encryption overhead reduce the usable payload (the reference light-node implementation reserves 96 bytes). Implementations MUST advertise the true usable maximum. |
| `maxTargetDepth` | 3 | Max neighborhood-prefix length in bytes for PSS `targets`/`pssTarget`. The minimum is fixed at **2 bytes** (below the network storage depth a chunk does not land in the recipient's neighborhood — see [PSS mining depth](#pss-mining-depth-prefix-length)); this limit caps the deep end (sender mining cost). |
| `maxSubscriptions` | 32 | Max concurrent active subscriptions **per origin**, including ones still being established. The node additionally bounds its *aggregate* receive pipelines across all origins (implementation-defined; not advertised here because it is shared state, not a per-origin promise) — exhaustion of that pool surfaces as a retryable `4900`, see `swarm_subscribe`. |

Not reported there:

- **Topics** are non-empty strings without control characters. Implementations
  MUST accept topics of up to 256 UTF-8 bytes, and MAY reject longer ones with
  `invalid_topic`.
- **New GSOC topics.** Deriving a topic's GSOC signer costs CPU (it is a
  mining search), so implementations MAY limit how many *new* topics — ones
  that have to be mined — an origin starts per time window, across
  `swarm_subscribe` and `swarm_sendGsoc`. Topics already derived MUST NOT
  count. Exceeding the limit is `-32602 rate_limited` with `retryAfterMs`. (One
  implementation allows 16 new topics per origin per 60 seconds.)

Applications needing to move more than `maxMessageBytes` in one logical message
MUST chunk at the application layer, or publish the bulk via `swarm_publishData`
and send only the resulting reference over messaging.

### Permission Model

This extension adds one tier to the core permission model:

| Tier | Methods | Grant needed | Prompt |
|---|---|---|---|
| **Messaging** | `swarm_getMessagingIdentity`, `swarm_subscribe` | connection + messaging grant | the first messaging-tier call acquires the grant; then never |
| | `swarm_sendPss`, `swarm_sendGsoc` | connection + messaging grant | the first call acquires the grant; then per send, unless auto-approve (messaging) |
| | `swarm_unsubscribe` | connection | never |

- **Messaging grant.** Separate from the publish and feed grants. Rationale:
  messaging discloses a stable public identity, opens a resource-consuming
  receive pipeline (the node pulls from a neighborhood on the origin's behalf),
  and sends consume postage. Any messaging method other than
  `swarm_unsubscribe` MAY trigger the grant prompt on first call; on rejection
  the method MUST reject with `4001`.
- **Send consent.** `swarm_sendPss`/`swarm_sendGsoc` MAY additionally require
  per-operation approval (like `swarm_publishData`), or fall under an origin
  auto-approve the user has enabled. Implementations MUST still enforce size
  limits and stamp checks under auto-approve, and SHOULD additionally apply a
  per-origin send rate cap there — messaging sends are cheap to issue in a
  loop and each consumes postage.
- **Permission manifests.** The core permission manifest's `messaging`
  capability grants the connection, the messaging grant and send auto-approval
  in one step.
- **Subscription resource control.** Subscriptions consume bandwidth for as long
  as they are open. Implementations MUST cap concurrent subscriptions per origin
  (`maxSubscriptions`) and SHOULD apply per-origin bandwidth accounting to the
  receive pipeline.
- **Revocation.** On disconnect, or when the user revokes only the messaging
  grant, the provider MUST tear down the origin's subscriptions and stop
  emitting `message` events for it.

### Encoding

Payloads cross the process/IPC/`postMessage` boundary as in the core spec, so
inbound payloads in the `message` event are **base64** (`encoding: "base64"`),
decoded exactly as `swarm_readFeedEntry` results are. Outbound `data` accepts
`string | Uint8Array | ArrayBuffer` as elsewhere.

## Rationale

### Why a subscription primitive instead of polling helpers?

The core API can already be polled (`swarm_readFeedEntry`, `swarm_readSingleOwnerChunk`).
Polling is sufficient for asynchronous state and is the right tool for
authoritative catch-up. It cannot reach interactive latency without pathological
request rates, and it cannot express PSS's push-only, encrypted, point-to-point
delivery at all. A subscription is the minimal addition that unlocks the class
of applications the core API cannot serve.

### Why an EIP-1193-style subscription?

Web3 developers already know `eth_subscribe` → subscription id → `message`
events. Borrowing that pattern (with `type: "swarm_subscription"`) means
existing mental models transfer, and keeps the provider to a single streaming
idiom rather than inventing per-method callbacks. The envelope is flatter than
EIP-1193's — `subscription` and `result` sit at the top level instead of under
`data` — so code written for EIP-1193 subscriptions needs a small adapter.
Aligning the envelope would break pages written against the deployed shape, so
this revision documents that shape.

### Why is GSOC send a new method rather than `swarm_writeSingleOwnerChunk`?

The core SOC write signs with the origin's publisher identity at a
caller-chosen identifier. GSOC requires an owner key *mined* so the resulting
SOC address falls in a chosen neighborhood — a different owner, chosen by the
provider, not the origin identity. Overloading the core method with a mining
mode would complicate its clean "sign with your identity" contract; a dedicated
method keeps both honest.

### Why does `swarm_sendGsoc` accept a topic but not an address?

An earlier draft mirrored `swarm_subscribe` and allowed a raw `address` on
send. That was unimplementable: authoring a GSOC means producing a signature
from the mined owner key, and the address — `keccak256(identifier ‖ owner)` —
is a one-way hash of the very material a sender needs. Every conforming
provider would have had to reject the parameter at runtime; better for the
interface to make the impossible state unrepresentable. The asymmetry is
principled: subscribing to an address is passive observation, sending to one
is authorship, and authorship needs the key that only the topic derivation
(or out-of-band agreement on it) can reproduce.

### Why expose a PSS public key separately from the signing identity?

Receiving PSS requires peers to encrypt to a key the node holds the private half
of. That key's role (encryption) and possibly its derivation differ from the
feed/SOC signing key. Exposing it explicitly via `swarm_getMessagingIdentity`
avoids conflating the two and lets implementations choose their key management.

### Why allow the node's own PSS key?

Per-origin PSS keys are the right design, and the specification asks for them.
But PSS decryption happens in the node, and today's nodes decrypt only with
their own key. A provider that wants per-origin keys has to receive the raw
trojan chunks and decrypt them itself, which current light-node interfaces do
not offer. Forbidding the node key would leave no conforming implementation of
PSS receive; allowing it, with a mandatory disclosure to the user and advice to
encrypt end to end, lets applications ship now and tells them exactly what
they are trading.

### Why may duplicates reach the page?

An earlier draft required providers to deduplicate deliveries. At the level a
provider usually sees — the decrypted payload — a network redelivery and a
genuine repeat are identical, so payload-based deduplication silently drops
legitimate messages (a second "ok", a keep-alive ping). Applications that need
exactly-once delivery need their own message ids anyway, because messages can
also arrive after a resubscribe or out of order; with ids, duplicates cost
nothing. Suppressing a redelivery of the *same chunk* remains allowed.

### Why surface best-effort delivery so prominently?

Because building on a false reliability assumption is the most likely way to
misuse this API. Stating the true semantics steers applications toward
CRDT/idempotent designs (which compose perfectly with these guarantees) and
toward using the durable feed layer for authoritative state — the intended
architecture: **feeds for the source of truth, messaging for live propagation.**

### Why must unsubscribe be idempotent?

A page that unsubscribes during teardown cannot always know whether the
provider already closed the subscription (navigation, revocation, a failed
pipeline). An error there is noise. Answering every well-formed id the same
way also keeps one origin from learning which subscription ids exist for
another.

## Backwards Compatibility

Purely additive. Implementations without this extension omit the messaging
methods and the `"messaging"` feature string; dApps detect absence via
`swarm_getCapabilities().features` and fall back to the feed/polling approach.
No core method changes.

## Security Considerations

Inherits the core specification's origin trust model, key-custody rules
(messaging private keys MUST NOT reach the page context; encryption/signing
occur in the trusted context), and resource-exhaustion posture. Additional
considerations:

- **Receive pipeline as an amplification/DoS surface.** A subscription makes the
  user's node pull from a neighborhood on the origin's behalf. Unbounded
  subscriptions or very shallow targets could be abused to consume bandwidth.
  Implementations MUST cap concurrent subscriptions (`maxSubscriptions`) and
  SHOULD bound per-origin receive bandwidth.
- **Cross-origin identity correlation.** Any node-global value returned to
  multiple origins is a correlation handle. This is why
  `swarm_getMessagingIdentity` returns a truncated `pssTarget` prefix rather
  than the node's full overlay address: the full overlay is identical for every
  origin, uniquely identifies the user's node, and discloses its exact network
  position. Even the truncated prefix leaks up to `maxTargetDepth` bytes of
  neighborhood information and is the same across origins; implementations
  SHOULD keep `maxTargetDepth` small. A node-global PSS key (below) is itself
  such a handle.
- <a id="shared-pss-key"></a>**Shared PSS key.** When `swarm_getMessagingIdentity` returns the node's
  key, every origin with the messaging grant shares one PSS identity, so:
  - any such origin that subscribes to a PSS topic receives every message the
    node decrypts on that topic, including messages a peer meant for another
    site, and topics are plain strings that any app can know or guess;
  - two sites the user uses can be linked through the key.

  An implementation that uses the node's key MUST say so to the user when it
  asks for the messaging grant. Applications SHOULD encrypt PSS payloads end
  to end (for example with keys from the
  [Encryption Key](./swip-draft_provider_api_encryption.md) extension) and use
  topics that cannot be guessed.
- **Metadata exposure.** PSS hides payloads but the existence and neighborhood
  of traffic is observable to the network. Subscribing to a topic reveals
  interest in that neighborhood to peers the node pulls from. Applications
  handling sensitive relationships SHOULD treat topics as potentially
  observable and derive them so as not to leak identifiers.
- **Send is unauthenticated at the transport layer.** GSOC broadcasts can be
  written by anyone who derives the address; PSS messages can be sent by anyone
  who knows the recipient key and topic. Applications MUST authenticate message
  *content* themselves (e.g. signed payloads) and MUST NOT trust a message's
  origin from the transport alone. This mirrors the core spec's stance that the
  provider secures access to the node, not the semantics of arbitrary data.
- **Payload confidentiality.** GSOC payloads are plaintext on Swarm unless the
  application encrypts them; only PSS provides transport encryption. For a
  private collaborative session over GSOC, the application MUST encrypt payloads
  (e.g. with a per-session key shared out of band, such as a key carried in a
  capability link).
- **Duplicate/stale delivery.** Duplicated, unordered delivery means a
  malicious or lagging peer could replay old messages. Application-layer
  sequence/version handling (again, native to CRDTs) is required; state MUST NOT
  advance irreversibly on receipt of a single unverified message.

## Test Cases

### Feature Detection

```javascript
const caps = await window.swarm.getCapabilities()
assert(Array.isArray(caps.features))
const hasMessaging = caps.features.includes('messaging')
if (hasMessaging) {
  assert(typeof caps.limits.maxMessageBytes === 'number')
  assert(typeof caps.limits.maxSubscriptions === 'number')
}
```

### Messaging Identity

```javascript
const { maxTargetDepth } = (await window.swarm.getCapabilities()).limits
const id = await window.swarm.getMessagingIdentity()
assert(/^[0-9a-f]{66}$/.test(id.pssPublicKey))   // 33-byte compressed key
assert(/^([0-9a-f]{2})+$/.test(id.pssTarget))    // whole bytes, hex
assert(id.pssTarget.length / 2 >= 2)              // deep enough to be stored
assert(id.pssTarget.length / 2 <= maxTargetDepth) // truncated prefix, never the full overlay
assert(['app-scoped', 'bee-wallet'].includes(id.identityMode))
```

### GSOC Broadcast Round-Trip (two peers, same topic, same provider implementation)

```javascript
// Peer B subscribes
const sub = await window.swarm.subscribe({ kind: 'gsoc', topic: 'room:doc-42' })
assert(typeof sub.subscriptionId === 'string')
assert(/^[0-9a-f]{64}$/.test(sub.key))   // resolved GSOC address

const received = new Promise((resolve) => {
  window.swarm.on('message', (m) => {
    if (m.subscription === sub.subscriptionId) {
      const bytes = Uint8Array.from(atob(m.result.data), c => c.charCodeAt(0))
      resolve(new TextDecoder().decode(bytes))
    }
  })
})

// Peer A sends on the same topic (address derives identically)
const sent = await window.swarm.sendGsoc({ topic: 'room:doc-42', data: 'hello room' })
assert(sent.sent === true)
assert(sent.address === sub.key)   // same-provider topic→address determinism
                                   // (across different providers, exchange `address` instead)

assert(await received === 'hello room')
```

### PSS Point-to-Point

```javascript
// Recipient publishes its messaging identity out of band
const me = await window.swarm.getMessagingIdentity()
const sub = await window.swarm.subscribe({ kind: 'pss', topic: 'dm:alice' })

// Sender (knowing the recipient's pssPublicKey, pssTarget, and topic,
// shared out of band)
await window.swarm.sendPss({
  topic: 'dm:alice',
  recipient: recipientPssPublicKey,   // 66-char hex
  targets: recipientPssTarget,        // recipient's getMessagingIdentity().pssTarget
  data: 'private hello',
})
// recipient's 'message' event fires with kind:'pss', decrypted payload
```

### Mailbox (offline delivery, where supported)

```javascript
// A sender posts to a room while nobody is subscribed.
await window.swarm.sendPss({ topic: 'room:42', recipient, targets, data: 'sent while you were away' })

// Later, a client subscribes WITH history — and receives the earlier message.
const seen = []
window.swarm.on('message', (m) => seen.push(m))
let sub
try {
  sub = await window.swarm.subscribe({ kind: 'pss', topic: 'room:42', history: true })
} catch (err) {
  // An implementation without the mailbox says so instead of ignoring it.
  assert(err.code === -32602)
  assert(err.data.reason === 'unsupported_option')
  assert(err.data.option === 'history')
  sub = await window.swarm.subscribe({ kind: 'pss', topic: 'room:42' }) // live only
}
// With the mailbox, `seen` comes to include messages for sub.subscriptionId
// that were sent before this subscription existed (bounded by storer
// retention; see Delivery semantics).
```

### Unsubscribe Stops Delivery and Is Idempotent

```javascript
const sub = await window.swarm.subscribe({ kind: 'gsoc', topic: 't' })
const res = await window.swarm.unsubscribe({ subscriptionId: sub.subscriptionId })
assert(res.unsubscribed === true)
// No further 'message' events fire for sub.subscriptionId

// Closing it again, or closing an id that never existed, also succeeds:
assert((await window.swarm.unsubscribe({ subscriptionId: sub.subscriptionId })).unsubscribed === true)
assert((await window.swarm.unsubscribe({ subscriptionId: 'nonexistent' })).unsubscribed === true)
```

### Payload Too Large

```javascript
const { maxMessageBytes } = (await window.swarm.getCapabilities()).limits
try {
  await window.swarm.sendGsoc({ topic: 't', data: 'x'.repeat(maxMessageBytes + 1) })
  assert.fail('should reject')
} catch (err) {
  assert(err.code === -32602)
  assert(err.data.reason === 'payload_too_large')
}
```

### Payload Bounds (empty)

```javascript
// GSOC cannot express an empty broadcast (bee rejects an empty SOC payload):
try {
  await window.swarm.sendGsoc({ topic: 't', data: '' })
  assert.fail('should reject')
} catch (err) {
  assert(err.code === -32602)
  assert(err.data.reason === 'invalid_payload')
}
// PSS carries an explicit length, so a zero-byte message is valid:
const res = await window.swarm.sendPss({
  topic: 't', recipient: peerKey, targets: peerTarget, data: '',
})
assert(res.sent === true)
```

### No Raw Address on Send

```javascript
try {
  await window.swarm.sendGsoc({ address: '00'.repeat(32), data: 'hi' })
  assert.fail('should reject')
} catch (err) {
  assert(err.code === -32602)
  assert(err.data.reason === 'invalid_address')
}
```

### Permission Gating

```javascript
// Before the messaging grant, a messaging method prompts; on rejection:
try {
  await window.swarm.subscribe({ kind: 'gsoc', topic: 't' })
  assert.fail('should reject when user denies messaging permission')
} catch (err) {
  assert(err.code === 4001)   // User Rejected
}
```

## Future Work

- **Normative GSOC derivation profile.** A pinned topic → (identifier,
  target neighborhood, proximity) derivation plus mining algorithm and search
  order, so the same topic converges on the same address across *independent*
  provider implementations, not only within one. Until then, cross-provider
  interop goes through the raw `address` parameter. The informative
  [example](#gsoc-topic-derivation) is a candidate starting point, minus its
  vendor-specific context string.
- **Per-origin PSS keys** everywhere, once nodes can decrypt PSS with keys other
  than their own, or hand providers the raw trojan chunks.
- **Mid-stream error signalling.** A normative way to report that an established
  subscription's pipeline has degraded/failed (a dedicated event or a
  `message` error variant), rather than the current SHOULD-reconnect approach.
- **Delivery/ack extensions for PSS.** Optional application-visible send
  acknowledgement where the underlying primitive can support it.
- **Presence/awareness helpers.** A higher-level convention for ephemeral
  presence (cursors, typing) layered on GSOC, if a common pattern emerges.
- **Backpressure signalling.** Surfacing to the dApp when the provider is
  shedding inbound messages under load, so applications can adapt send rates.

## Reference Endpoints (informative)

This extension is designed to sit atop light-node PSS/GSOC support exposing
gateway endpoints equivalent to:

- `POST /pss/send/{topic}/{targets}` (+ recipient key) — backs `swarm_sendPss`
- `POST /soc/{owner}/{id}` with mined owner/id — backs `swarm_sendGsoc`
- `GET /gsoc/subscribe/{address}` (streaming) — backs `swarm_subscribe` (gsoc)
- `GET /pss/subscribe/{topic}` (streaming) — backs `swarm_subscribe` (pss)
- `GET /addresses` reporting `pssPublicKey`/overlay — backs `swarm_getMessagingIdentity`
  (the provider truncates the overlay to the `pssTarget` prefix; the full
  overlay never crosses the page boundary)

The provider is responsible for translating the ergonomic, consent-gated,
origin-scoped method surface above onto whatever node interface it drives; these
endpoints are named only to ground the mapping.

## Implementation

[Freedom Browser](https://github.com/solardev-xyz/freedom-browser) implements
this extension on the [Ant](https://github.com/freedom-hq/ant) light node; see
[`docs/swarm-provider-api.md`](https://github.com/solardev-xyz/freedom-browser/blob/main/docs/swarm-provider-api.md)
and [`src/main/swarm/messaging-service.js`](https://github.com/solardev-xyz/freedom-browser/blob/main/src/main/swarm/messaging-service.js).
It uses the node's PSS key (`identityMode: "bee-wallet"`) and says so in its
messaging prompt; the resulting exposure is tracked as
[#1156](https://github.com/solardev-xyz/freedom-browser/issues/1156). Making
`swarm_unsubscribe` idempotent and refusing `history` until the mailbox ships
are tracked as [#1157](https://github.com/solardev-xyz/freedom-browser/issues/1157).

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
