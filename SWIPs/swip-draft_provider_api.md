---
SWIP: <to be assigned>
title: Swarm Provider API (`window.swarm`)
author: Florian Glatz (@heckerhut)
discussions-to: https://discord.com/channels/799027393297514537/1239813439136993280
status: Draft
type: Standards Track
category: Interface
created: 2026-04-03
---

## Simple Summary

A standard JavaScript API (`window.swarm`) that lets web pages use the user's own Swarm node: publish data and files, manage mutable feeds, read and write indexed feed entries, read verified blobs and chunks, write low-level chunk primitives, and sign with a per-site publisher identity. Every operation that spends the user's resources or speaks for the user needs the user's consent, and permissions are scoped to the page's origin.

## Abstract

This SWIP defines a browser-injected JavaScript provider object (`window.swarm`) that allows web applications to interact with a user's Swarm node. The API follows the request/response pattern established by [EIP-1193](https://eips.ethereum.org/EIPS/eip-1193) for Ethereum providers, adapted for Swarm's publishing, feed, and chunk primitives.

It specifies sixteen RPC methods covering connection, capability discovery, data and file publishing, upload tracking, mutable feeds, indexed feed entries, origin-scoped feed introspection, verified blob reads, low-level chunk primitives (content-addressed and single-owner), and signing-identity disclosure. The provider has a permission model with explicit user consent, origin isolation, per-site publisher identities, and size limits. A site served from Swarm can declare the capabilities it needs up front in a **permission manifest**, so the user answers one consent sheet instead of a prompt per capability. Read-only methods on public Swarm data need no grant but MUST be rate-limited per origin.

Two companion extensions build on this specification: [Messaging](./swip-draft_provider_api_messaging.md) (PSS and GSOC, with subscriptions) and [Encryption Key](./swip-draft_provider_api_encryption.md) (a per-site key for key agreement).

## Motivation

Today, publishing to Swarm from a web application requires either:

1. Direct HTTP calls to a locally running Bee node API (exposing the full API surface, including administrative endpoints, to any web page that can reach `localhost`), or
2. Server-side publishing infrastructure (defeating the purpose of decentralized hosting).

Neither approach is suitable for a user-sovereign web. Users need a way to grant web applications controlled, permissioned access to Swarm publishing capabilities — similar to how `window.ethereum` ([EIP-1193](https://eips.ethereum.org/EIPS/eip-1193)) lets web applications interact with a user's Ethereum wallet without exposing private keys.

Without a standard, each Swarm-enabled browser or extension will invent its own API. This fragments the ecosystem: dApp developers must write implementation-specific code, and users can't switch clients without breaking functionality. A standard `window.swarm` provider ensures interoperability across implementations.

### Design Goals

- **User consent first.** No write, signing, or other operation that consumes user resources or speaks for the user executes without consent or an explicit standing grant. Read-only operations on public Swarm data are ungated but rate-limited per origin.
- **Minimal surface.** Only the data plane — not administrative, debug, or staking endpoints.
- **Origin isolation.** Each web origin gets isolated permissions and its own publisher identities.
- **Recoverable identities.** Keys that speak for a site follow from the user's recovery phrase, so restoring the phrase restores them, in any conforming implementation.
- **Implementation freedom.** The spec defines the interface, not how the host manages nodes, key storage, or consent UI.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

### Provider Object

Conforming implementations MUST inject a `window.swarm` object into the JavaScript context of web pages. The object MUST be available before the page's `DOMContentLoaded` event fires, and SHOULD be available before any of the page's own scripts run, so that classic, deferred and module scripts can all use it while the page parses.

Implementations SHOULD expose the provider only in top-level browsing contexts (see [Iframe and Nested Context Behavior](#iframe-and-nested-context-behavior)). They MAY omit it from private browsing contexts and from their own internal pages.

An implementation whose user has turned the provider off MAY still inject `window.swarm`; every call, including permission-free ones, then rejects with `4900`.

#### Detection

```javascript
if (typeof window.swarm !== 'undefined') {
  // Swarm provider is available
}
```

#### Properties

Implementations MAY include additional boolean properties identifying themselves (e.g., `isFreedomBrowser`). These are NOT part of the standard interface and MUST NOT be relied upon for feature detection. Use `swarm_getCapabilities` for capability discovery.

> **Note:** Future versions of this specification may define a standard `isSwarmProvider` property.

### Request Interface

The provider MUST expose a `request` method:

```javascript
window.swarm.request({ method: string, params?: object }): Promise<any>
```

- `method` — (Required) A string identifying the RPC method to invoke.
- `params` — (Optional) A plain object containing method-specific parameters. An omitted `params` MUST be treated as `{}`.

The method MUST return a `Promise` that resolves with the method's result, or rejects with an error object (see [Error Format](#error-format)).

#### Encoding Conventions

- **Hex strings** (references, topics, identifiers, keys, signatures) are case-insensitive on input and carry no `0x` prefix, unless a method says otherwise. Providers return them in lowercase.
- **Ethereum addresses** (`owner`) MAY carry a `0x` prefix on input. Providers return them checksummed ([EIP-55](https://eips.ethereum.org/EIPS/eip-55)) with `0x`.
- **A reference** is 64 hex characters (32 bytes).
- **Binary input** (`data`, `bytes`) accepts `string` (encoded as UTF-8), `Uint8Array`, or `ArrayBuffer`.
- **Binary output** is a base64 string with `encoding: "base64"` (see [Why base64 for read responses?](#why-base64-for-read-responses)).

### Convenience Methods

Implementations MUST expose convenience wrappers for each standard method:

```javascript
window.swarm.requestAccess()                               // swarm_requestAccess
window.swarm.getCapabilities()                             // swarm_getCapabilities
window.swarm.publishData({ data, contentType, name })      // swarm_publishData
window.swarm.publishFiles({ files, indexDocument })        // swarm_publishFiles
window.swarm.getUploadStatus({ tagUid })                   // swarm_getUploadStatus
window.swarm.createFeed({ name })                          // swarm_createFeed
window.swarm.updateFeed({ feedId, reference })             // swarm_updateFeed
window.swarm.writeFeedEntry({ name, data, index })         // swarm_writeFeedEntry
window.swarm.readFeedEntry({ name | topic, owner, index }) // swarm_readFeedEntry
window.swarm.listFeeds()                                   // swarm_listFeeds
window.swarm.publishChunk({ data, span })                  // swarm_publishChunk
window.swarm.readChunk({ reference })                      // swarm_readChunk
window.swarm.readData({ reference })                       // swarm_readData
window.swarm.writeSingleOwnerChunk({ identifier, data })   // swarm_writeSingleOwnerChunk
window.swarm.readSingleOwnerChunk({ owner, identifier })   // swarm_readSingleOwnerChunk
window.swarm.getSigningIdentity()                          // swarm_getSigningIdentity
```

Each convenience method MUST be equivalent to calling `window.swarm.request()` with the corresponding method name and parameters. Extensions add their own wrappers under the same rule: the method name without the `swarm_` prefix.

### Events

The provider MUST support event subscription:

```javascript
window.swarm.on(event: string, handler: Function): this
window.swarm.removeListener(event: string, handler: Function): this
window.swarm.removeAllListeners(event?: string): this
```

Implementations MAY also expose `addListener` as an alias of `on`.

#### Standard Events

| Event | Data | Description |
|---|---|---|
| `connect` | `{ origin: string }` | The origin became connected (see below). |
| `disconnect` | `{ origin: string }` | The origin's connection was revoked, or the provider disconnected. |

- **`connect`** MUST be emitted to the page whose `swarm_requestAccess` call established a new connection, whichever consent UI established it (the connect prompt or a [permission manifest](#permission-manifests) sheet). It is NOT emitted when the origin was already connected.
- **`disconnect`** MUST be emitted to every page of that origin that is currently loaded, not only the one in front. Revoking a single tier or a single auto-approval is not a disconnect: no event is emitted, and the page learns of it from a `4100` on its next call.

Events are a convenience. The authoritative signal is the method result: a call that rejects with `4100` means the grant is gone, whether or not an event arrived.

### Timeouts

Implementations MAY reject a request that has not settled within an implementation-defined time, with `-32603`. A timeout SHOULD leave the user enough time to read and answer any prompt the request waits on, and enough time for the node writes it starts.

A request that timed out on the page MAY still complete: the node write may land, and a consent decision made after the timeout still takes effect. Pages SHOULD re-read state (for example with `swarm_requestAccess` or `swarm_getCapabilities`) rather than assume failure. A response that settles after the page has navigated to a new document MUST NOT be delivered to the new document.

### Error Format

Rejections MUST carry the fields of a [JSON-RPC 2.0 error object](https://www.jsonrpc.org/specification#error_object). Implementations SHOULD reject with `Error` instances carrying these fields:

```javascript
{
  code: number,
  message: string,
  data?: any
}
```

#### Standard Error Codes

| Code | Name | Description |
|---|---|---|
| 4001 | User Rejected | The user denied the request. |
| 4100 | Unauthorized | The origin lacks a grant the method needs, or required consent cannot be obtained in this context. `data.reason` says which. |
| 4200 | Unsupported Method | The requested method is not recognized. |
| 4900 | Node Unavailable | The Swarm node is not running, not ready, or lacks usable postage; or the provider is turned off. `data.reason` says which. |
| -32002 | Resource Unavailable | OPTIONAL. The provider cannot show a prompt now because a conflicting user interaction is in progress (for example a hardware-wallet confirmation). Retryable. |
| -32602 | Invalid Params | Missing or invalid method parameters, a limit exceeded, or a permission-free read over budget. `data.reason` says which. |
| -32603 | Internal Error | An unexpected error in the provider or the node, or a timeout. |

Error codes 4001, 4100, 4200, and 4900 are aligned with the [EIP-1193](https://eips.ethereum.org/EIPS/eip-1193) and [EIP-1474](https://eips.ethereum.org/EIPS/eip-1474) error code ranges; -32002 with EIP-1474's "resource unavailable".

#### Structured Error Reasons

Implementations SHOULD include a `data.reason` string on every `4100` and `-32602`, and on `4900` where a reason below applies. Implementations MAY define further reasons of their own. Pages MUST handle a reason they do not recognize by its `code`.

**`4100` reasons:**

| Reason | Meaning |
|---|---|
| `not_connected` | The origin has no connection grant (`swarm_requestAccess`). |
| `feed_not_granted` | A [feeds or signing tier](#permission-tiers) method without the feed grant. |
| `tag_ownership_mismatch` | `swarm_getUploadStatus` for a tag this origin did not create, or one the provider has forgotten. |

**`4900` reasons.** The first four are the same codes `swarm_getCapabilities` reports as `reason`:

| Reason | Meaning |
|---|---|
| `node-stopped` | The node is not running or does not answer. |
| `node-not-ready` | The node is running but not ready to write (still syncing or connecting, or a postage batch is not yet usable). |
| `ultra-light-mode` | The node runs in ultra-light mode and cannot write. |
| `no-usable-stamps` | No postage batch has room for the write. |

Permission-free reads only ever report `node-stopped`. A `4900` without `data.reason` means the provider is turned off, or a permission manifest the origin depends on could not be read (see [Permission Manifests](#permission-manifests)).

**`-32602` reasons:**

| Reason | Applicable Methods | Meaning |
|---|---|---|
| `invalid_params` | All | Generic shape error not covered by a more specific reason. |
| `unsupported_option` | All methods accepting `options` | An option field was supplied that the implementation does not recognize. `data.option` names it. |
| `payload_too_large` | `swarm_publishData`, `swarm_publishFiles`, `swarm_writeFeedEntry`, `swarm_publishChunk`, `swarm_writeSingleOwnerChunk` | Payload (or total size) exceeds the advertised limit. |
| `missing_content_type` | `swarm_publishData` | `contentType` is missing or empty. |
| `empty_files` | `swarm_publishFiles` | `files` is empty. |
| `too_many_files` | `swarm_publishFiles` | More files than `maxFileCount`. |
| `invalid_path` | `swarm_publishFiles` | A path breaks the [path rules](#swarm_publishfiles). |
| `duplicate_path` | `swarm_publishFiles` | Two files share a path. `data.path` names it. |
| `invalid_index_document` | `swarm_publishFiles` | `indexDocument` is not one of the paths. |
| `invalid_feed_name` | Feed methods | Feed name is empty, too long, or contains `/` or control characters. |
| `invalid_topic` | `swarm_readFeedEntry` | Topic is not a valid 64-character hex string. |
| `invalid_owner` | `swarm_readFeedEntry`, `swarm_readSingleOwnerChunk` | Owner is missing (when required) or not a valid address. |
| `feed_not_found` | `swarm_readFeedEntry`, `swarm_updateFeed`, `swarm_writeFeedEntry` | The origin has no feed by that name. |
| `feed_empty` | `swarm_readFeedEntry` | The feed has no entries (latest-entry read). |
| `entry_not_found` | `swarm_readFeedEntry` | No entry at the requested index. |
| `index_already_exists` | `swarm_writeFeedEntry` | An entry already exists at the explicit index (overwrite protection). |
| `invalid_reference` | `swarm_updateFeed`, `swarm_readChunk`, `swarm_readData`, `swarm_readSingleOwnerChunk` | Reference or address is not a valid 64-character hex string, or is of a kind the method does not accept. |
| `invalid_identifier` | `swarm_writeSingleOwnerChunk`, `swarm_readSingleOwnerChunk` | SOC identifier is not a valid 64-character hex string. |
| `invalid_span` | `swarm_publishChunk`, `swarm_writeSingleOwnerChunk` | Explicit `span` is negative, non-integer, exceeds the 8-byte unsigned range, or is a `number` outside `Number.MAX_SAFE_INTEGER` (use `bigint` for values above 2⁵³ − 1). |
| `chunk_not_found` | `swarm_readChunk`, `swarm_readData`, `swarm_readSingleOwnerChunk` | No chunk exists at the requested address (for `swarm_readData`: the root or a chunk under it). |
| `chunk_type_mismatch` | `swarm_readChunk`, `swarm_readData`, `swarm_readSingleOwnerChunk` | Returned bytes do not validate as the requested chunk type. |
| `data_too_large` | `swarm_readData` | The blob is larger than the origin's whole read byte budget. |
| `rate_limited` | Permission-free reads | The calling origin has exceeded its per-origin rate or bandwidth budget for permission-free reads. Retry after backoff. |

**Extra `data` fields.** Where they apply, implementations SHOULD include:

| Field | On | Meaning |
|---|---|---|
| `limit`, `actual` | `payload_too_large`, `too_many_files`, `data_too_large` | The limit and the size or count the request had (bytes, or files). |
| `option` | `unsupported_option` | The unrecognized option key. |
| `path` | `duplicate_path` | The duplicated path. |
| `windowMs`, `maxRequests`, `maxBytes`, `retryAfterMs` | `rate_limited` | The budget that was exceeded, and when to retry, as far as the implementation can say. |

---

### Methods

#### `swarm_requestAccess`

Requests the user's permission to interact with their Swarm node from this origin. Implementations MUST present a user-visible consent prompt. The user MUST be able to deny access.

**Params:** None.

**Result:**

```javascript
{
  connected: true,
  origin: string,          // The normalized origin that was granted access
  capabilities: string[]   // This revision defines only "publish", which every connection carries
}
```

Pages SHOULD use `swarm_getCapabilities`, not `capabilities`, to learn what the provider can do for them.

**Errors:** `4001` if the user rejects. `4900` if the origin's [permission manifest](#permission-manifests) is on record but cannot be read.

**Behavior:**
- If the origin serves a valid [permission manifest](#permission-manifests), the implementation SHOULD show the manifest consent sheet instead of the plain connect prompt.
- Calling `swarm_requestAccess` when already connected MUST NOT show the connect prompt again, and MUST resolve with the existing connection state. The one exception: if the origin's permission manifest names capabilities the user has not yet answered, the implementation MAY show the manifest sheet for those.
- When the call establishes a new connection, the provider MUST emit a `connect` event to the calling page (see [Events](#standard-events)).

---

#### `swarm_getCapabilities`

Returns the current capabilities of the provider for this origin. Does NOT require prior access — can be called before `swarm_requestAccess` for feature detection.

**Params:** None.

**Result:**

```javascript
{
  specVersion: string,              // Specification version (e.g. "1.0")
  canPublish: boolean,              // true if connected AND node is ready to write
  reason: string | null,            // null if canPublish is true, otherwise a reason code
  features: string[],               // Extensions implemented, e.g. ["messaging", "encryption-key"]
  publisherIdentityModes: string[], // Identity modes the implementation supports, see Identity Model
  limits: {
    maxDataBytes: number,         // Maximum payload size for swarm_publishData and swarm_writeFeedEntry
    maxFilesBytes: number,        // Maximum total size for swarm_publishFiles
    maxFileCount: number,         // Maximum number of files per swarm_publishFiles call
    maxPathBytes: number,         // Maximum length of a file path in swarm_publishFiles, measured in UTF-8 bytes
    maxChunkPayloadBytes: number  // Maximum payload size for swarm_publishChunk and swarm_writeSingleOwnerChunk (4096 by protocol)
    // Extensions add their own limits here.
  }
}
```

**Reason codes:**

| Code | Meaning |
|---|---|
| `"not-connected"` | Origin has not called `swarm_requestAccess`. Reported whenever the origin is not connected, whatever the node's state. |
| `"node-stopped"` | Node is not running. |
| `"ultra-light-mode"` | Node is in ultra-light mode (browse only, cannot publish). |
| `"node-not-ready"` | Node is running but not yet ready to write. |
| `"no-usable-stamps"` | No postage stamps with remaining capacity. |

**`specVersion`:** The version of this specification that the provider implements. This SWIP defines version `"1.0"`. Implementations SHOULD return this field regardless of whether the origin has called `swarm_requestAccess`.

**`features`:** MUST be present (possibly empty). Each companion extension names the string an implementation of it MUST list: `"messaging"` for [Messaging](./swip-draft_provider_api_messaging.md), `"encryption-key"` for [Encryption Key](./swip-draft_provider_api_encryption.md). Pages detect extensions here, not from `specVersion`.

**`publisherIdentityModes`:** MUST be present and list every [identity mode](#identity-model) the implementation can sign with: always at least one of `"app-scoped"` and `"bee-wallet"`, and `"ethereum-wallet"` if supported.

Implementations MAY add implementation-specific fields. Pages MUST NOT rely on them to detect standard features.

---

#### `swarm_publishData`

Upload a single blob of data to Swarm, as a one-file site.

**Params:**

| Field | Type | Required | Description |
|---|---|---|---|
| `data` | `string \| Uint8Array \| ArrayBuffer` | Yes | The content to upload. Strings are encoded as UTF-8. |
| `contentType` | `string` | Yes | MIME type (e.g. `"text/html"`, `"application/json"`). |
| `name` | `string` | No | File name of the entry, and a display label for the upload. |

**Result:**

```javascript
{
  reference: string,   // 64-character hex Swarm reference
  bzzUrl: string       // "bzz://<reference>"
}
```

> **Note on `bzz://` URLs:** The `bzz://` URI scheme is a Swarm convention for content-addressed references. It is not an IANA-registered scheme. Implementations resolve `bzz://` URLs through a local Bee gateway or native protocol handler.

**Errors:** `4001` if the user rejects a per-publish prompt. `4100` if not connected. `4900` if node unavailable. `-32602` if params invalid (`missing_content_type`, `invalid_params`) or payload exceeds `maxDataBytes` (`payload_too_large`).

**Behavior:**
- `bzzUrl` MUST open the data, served with `contentType`. The `reference` is therefore that of a Swarm manifest with one entry, not of the raw bytes: `swarm_readData` on it returns the manifest's own bytes.
- Implementations SHOULD prompt the user for per-publish consent unless the user has opted into auto-approve for this origin.
- The implementation selects an appropriate postage batch automatically — dApps do not manage batches.
- Uploads MUST be pinned on the local node.

---

#### `swarm_publishFiles`

Upload a collection of files as a Swarm manifest (directory).

**Params:**

| Field | Type | Required | Description |
|---|---|---|---|
| `files` | `Array<FileEntry>` | Yes | Array of file objects. |
| `indexDocument` | `string` | No | Path of the default document (e.g. `"index.html"`). Must match an entry in `files`. If omitted, implementations MAY default to `index.html` when that path is present. |

**FileEntry:**

| Field | Type | Required | Description |
|---|---|---|---|
| `path` | `string` | Yes | Virtual path in the manifest (e.g. `"index.html"`, `"assets/style.css"`). |
| `bytes` | `Uint8Array \| ArrayBuffer` | Yes | File content. |
| `contentType` | `string` | No | MIME type. Implementations SHOULD serve the file with it; if it is omitted they MAY infer one from the path. |

**Path validation rules:**
- Must be a non-empty string. Maximum length is `maxPathBytes` UTF-8 bytes, advertised via `swarm_getCapabilities`. Length MUST be measured in UTF-8 bytes, not Unicode characters or UTF-16 code units — a single emoji is 4 bytes, not 1.
- No backslashes, no leading slash, no empty segments, no `.` or `..` segments.
- No control characters (code points < 32).
- Paths must be unique within the `files` array.
- Implementations MAY reject further paths that their upload pipeline cannot represent (for example names a host file system reserves) with `invalid_path`.

**Result:**

```javascript
{
  reference: string,     // 64-character hex Swarm reference
  bzzUrl: string,        // "bzz://<reference>"
  tagUid: number | null  // Upload tag for progress tracking (if supported)
}
```

**Errors:** `4001` if the user rejects a per-publish prompt. `4100` if not connected. `4900` if node unavailable. `-32602` for `empty_files`, `too_many_files`, `invalid_path`, `duplicate_path`, `invalid_index_document`, or `payload_too_large` (total size over `maxFilesBytes`).

---

#### `swarm_getUploadStatus`

Query the upload progress of a previously initiated file upload.

**Params:**

| Field | Type | Required | Description |
|---|---|---|---|
| `tagUid` | `number` | Yes | The tag ID returned by `swarm_publishFiles`. A positive integer. |

**Result:**

```javascript
{
  tagUid: number,
  split: number,     // Total chunks
  seen: number,      // Chunks the node already had
  stored: number,    // Chunks stored locally
  sent: number,      // Chunks dispatched to the network
  synced: number,    // Chunks acknowledged by the network
  progress: number,  // 0-100 percentage (based on sent/split)
  done: boolean      // true when split > 0 and every chunk is synced or was already seen
}
```

Implementations MAY include further fields (for example the upload's `reference`).

`progress` measures chunks *sent*, because syncing can lag far behind on a light node. `done` is stricter: a sent chunk is not yet acknowledged, so `done` waits until `synced + seen >= split`. `progress` can therefore reach 100 before `done` is true.

**Errors:** `4100` if not connected (`not_connected`) or the tag is not this origin's (`tag_ownership_mismatch`). `-32602` if `tagUid` is invalid.

**Security:** Implementations MUST enforce origin-scoped tag ownership. A page MUST NOT be able to query upload status for tags created by a different origin. This prevents cross-origin upload snooping.

**Persistence:** Tag ownership is session-scoped. Implementations are NOT REQUIRED to persist it across restarts, and MAY forget a tag once it has reported `done: true`. A query for a forgotten tag fails like one for another origin's tag (`tag_ownership_mismatch`).

---

#### `swarm_createFeed`

Create a mutable Swarm feed with a stable `bzz://` URL.

Feeds allow web applications to maintain updatable content at a fixed address. The feed manifest provides a permanent `bzz://` URL that always resolves to the latest update.

**Params:**

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | `string` | Yes | Feed identifier, unique per origin. 1–64 UTF-16 code units (JavaScript `string.length`), no `/`, no control characters. |

**Result:**

```javascript
{
  feedId: string,              // The feed name (same as input)
  owner: string,               // Checksummed Ethereum address of the signing key
  topic: string,               // 64-character hex topic
  manifestReference: string,   // 64-character hex reference to the feed manifest
  bzzUrl: string,              // "bzz://<manifestReference>"
  identityMode: string         // "app-scoped", "bee-wallet" or "ethereum-wallet"
}
```

**Errors:** `4001` if the user rejects the prompt. `4100` if the origin lacks basic connection or the implementation cannot surface a permission prompt. `4900` if node unavailable. `-32602` for `invalid_feed_name`.

**Behavior:**
- The feed is signed by the origin's **active** publisher identity (see [Identity Model](#identity-model)) and stays bound to it.
- Creating a feed that already exists MUST be idempotent: return the existing feed's metadata, including the owner and `identityMode` of the identity that created it, even if the origin's active identity has changed since. Nothing is written.
- Feed creation is in the [feeds tier](#permission-tiers). The first feeds-tier or signing-tier call acquires the feed grant, and the implementation MAY let the user choose the publisher identity at that point.

**Feed Topic Derivation:**

The topic for a feed MUST be derived from the normalized origin (the [permission key](#origin-normalization)) and the feed name:

```
topic = keccak256(utf8(normalizedOrigin + "/" + feedName))
```

This ensures feeds are origin-scoped at the protocol level — two different origins using the same feed name produce different topics.

---

#### `swarm_updateFeed`

Update a feed to point at a new content reference.

**Params:**

| Field | Type | Required | Description |
|---|---|---|---|
| `feedId` | `string` | Yes | Feed name (as returned by `swarm_createFeed`). |
| `reference` | `string` | Yes | 64-character hex Swarm reference to point the feed at. |

**Result:**

```javascript
{
  feedId: string,    // The feed name
  reference: string, // The new content reference
  bzzUrl: string,    // "bzz://<manifestReference>" (stable feed URL)
  index: number      // The sequence index that was written
}
```

**Errors:** `4001` if the user rejects the prompt. `4100` if the origin lacks basic connection or the implementation cannot surface a permission prompt. `4900` if node unavailable. `-32602` for `invalid_reference`, `invalid_feed_name`, `feed_not_found`.

**Behavior:**
- The update is signed with the identity that created the feed, not the origin's active identity.
- Writes are serialized per-topic (same mutex as `swarm_writeFeedEntry`). The returned `index` reflects the actual sequence index written.

---

#### `swarm_writeFeedEntry`

Write an arbitrary payload directly to a feed index as a Single Owner Chunk (SOC).

This enables the **journal pattern** — an append-only log where each entry is an independently-addressed chunk at an incrementing index. Unlike `swarm_updateFeed` (which stores a 32-byte content reference), `swarm_writeFeedEntry` stores the payload in the feed itself.

The 4 KB SOC body limit is internal to the storage envelope and is NOT a dApp-visible payload cap. Payloads up to the SOC limit are stored directly in the chunk; larger payloads MUST be wrapped transparently — the implementation uploads the bytes (e.g., as a BMT tree via `POST /bytes`) and writes only the root chunk reference into the SOC body. DApps see one consistent payload cap, `maxDataBytes`, across `swarm_publishData` and `swarm_writeFeedEntry`.

**Params:**

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | `string` | Yes | Feed name (same namespace as `swarm_createFeed`). The feed must already exist. |
| `data` | `string \| Uint8Array \| ArrayBuffer` | Yes | The payload to write. Strings are encoded as UTF-8. |
| `index` | `number` | No | Explicit index to write at: a non-negative integer. If omitted, auto-increments to the next available index. Values above `Number.MAX_SAFE_INTEGER` (2⁵³ − 1) are not portable: implementations MAY reject them with `invalid_params`, and pages SHOULD NOT use them. |

**Result:**

```javascript
{
  index: number   // The feed index that was written
}
```

**Errors:** `4001` if the user rejects the prompt. `4100` if the origin lacks basic connection or the implementation cannot surface a permission prompt. `4900` if node unavailable. `-32602` for `invalid_feed_name`, `feed_not_found`, `index_already_exists`, `payload_too_large` (over `maxDataBytes`), or an invalid index (`invalid_params`).

**Behavior:**

- The feed MUST already exist (created via `swarm_createFeed`). The entry is signed with the identity that created the feed.
- Implementations MUST reject a payload larger than `maxDataBytes` with `payload_too_large`.
- If `index` is omitted, the provider MUST resolve the next available index and write at that index. The resolved index is returned.
- If `index` is provided, the provider MUST check whether an entry already exists at that exact index. If it does, the write MUST be rejected with error data `{ reason: "index_already_exists" }`. This overwrite protection is the core safety guarantee of the journal pattern. On nodes that report a missing entry as HTTP 500, it is only as strong as the node's error reporting (see [Node error mapping](#node-error-mapping)).
- The existence check MUST use **exact-match** semantics — see [Implementation Note: Exact-Match Feed Reads](#implementation-note-exact-match-feed-reads). Implementations MUST NOT use Bee's feed lookup (`GET /feeds/{owner}/{topic}`) for this probe: it answers with the latest entry it finds, not the entry at a given index, so it would report the index as occupied whenever the feed has any entry and falsely reject sparse-index writes.
- The overwrite check and the write MUST be atomic — implementations MUST NOT allow concurrent writes to race between the check and the write to the same index. A per-topic serialization mechanism (e.g., a mutex) is RECOMMENDED.
- Sparse indices are valid. Writing to index 1000 without indices 0–999 existing is allowed. Applications that want sequential journal semantics SHOULD omit `index` and rely on auto-increment.

**Write serialization:** Implementations MUST serialize writes to the same feed topic. Two concurrent `swarm_writeFeedEntry` calls to the same feed MUST NOT produce index collisions. Writes to different feeds MAY execute in parallel.

---

#### `swarm_readFeedEntry`

Read a feed entry at a specific index, or read the latest entry. This is a read-only operation on public Swarm data and does NOT require any permission grant — neither connection (`swarm_requestAccess`) nor feed-level. Any web page MAY call it.

**Params:**

| Field | Type | Required | Description |
|---|---|---|---|
| `topic` | `string` | Conditional | 64-character hex topic. Required if `name` is not provided. |
| `name` | `string` | Conditional | Feed name (same namespace as `swarm_createFeed`). Required if `topic` is not provided. |
| `owner` | `string` | Conditional | Ethereum address of the feed owner (signer). See owner resolution rules below. |
| `index` | `number` | No | Specific index to read: a non-negative integer, with the same range as in `swarm_writeFeedEntry`. If omitted, reads the latest entry (see `nextIndex` below for sparse feeds). |

Exactly one of `topic` or `name` MUST be provided (otherwise `invalid_params`):
- **`topic`**: Raw 64-character hex topic. The provider constructs the topic directly from these bytes (no hashing). Use this to read feeds created by other origins or other users. `owner` is required.
- **`name`**: Feed name string. The provider derives the topic by hashing: `keccak256(normalizedOrigin + "/" + name)`, always against the *calling* origin. Use this to read your own feeds.

**Owner resolution:**
- When using `topic`: `owner` is required.
- When using `name` without `owner`: The provider looks up the owner from its local feed store (populated when `swarm_createFeed` was called). No key access or signing is needed — this is a metadata lookup.
- When using `name` with `owner`: The provided owner is used. This allows reading another user's feed that shares the same app-scoped topic derivation.

**Result:**

```javascript
{
  data: string,           // Base64-encoded payload
  encoding: "base64",     // Explicit encoding identifier
  index: number,          // The index that was read
  nextIndex: number|null  // index + 1 (see below)
}
```

**`data`** is the entry's payload as its writer stored it, the same bytes whether the entry is read as the latest entry or at its `index`. A payload that `swarm_writeFeedEntry` wrapped because it was larger than one chunk is returned whole, as it was written. Otherwise, implementations MUST NOT return content the payload refers to in its place. In particular, an entry that holds a reference, such as one written by `swarm_updateFeed` (an 8-byte big-endian timestamp followed by the 32-byte reference), is returned as those 40 bytes, although the node's feed lookup (`GET /feeds/{owner}/{topic}`) answers with the content the reference points to. An implementation that uses the lookup to find the latest entry therefore takes only the index from it and reads the entry at that index (see [When to use which endpoint](#when-to-use-which-endpoint)). Callers read the referenced content with `swarm_readData`.

**`nextIndex`** is the index after the one returned. For a latest-entry read it is the next index as the node reports it, which on a contiguous feed is the index the next append will use; it MAY be `null` if the node does not report one. On a feed written with sparse explicit indexes, the node's lookup can stop at a gap (see [Implementation Note](#implementation-note-exact-match-feed-reads)), so a latest-entry read MAY return an older entry, and a `nextIndex` that is not the end of the feed: treat both as hints there. For an explicit-index read it is `index + 1` whether or not that slot is taken — it is NOT the next free index. It MUST be `null` where the next index would exceed `Number.MAX_SAFE_INTEGER`. To append, use `swarm_writeFeedEntry` without `index`, or read the latest entry first.

**Errors:** `4900` if node unreachable (`node-stopped`). `-32602` for `invalid_topic`, `invalid_owner`, `invalid_feed_name`, `invalid_params`, `feed_not_found`, `feed_empty`, `entry_not_found`, or `rate_limited`. Notably does NOT return `4100` — no permission is required to call this method.

**Behavior:**

- This method does NOT require any permission grant. Feeds are public data on Swarm and the same lookup is available from any Bee gateway without auth; gating it would be friction without security benefit. This is symmetric with `swarm_listFeeds`.
- Pre-flight checks MUST be limited to verifying the node's HTTP API is reachable. Mode checks (ultra-light), readiness checks, and stamp checks MUST NOT be applied — reads work regardless of node mode and do not consume stamps.
- Implementations MUST NOT report a timeout or connection failure as `feed_empty` or `entry_not_found`; those propagate as `-32603`. Because some nodes report a missing feed entry as HTTP 500, an implementation MAY treat a node's 500 as "not found" here (see [Node error mapping](#node-error-mapping)). A "not found" on a feed that is expected to exist is then worth one retry.
- When `index` is provided, implementations MUST use **exact-match** semantics — see [Implementation Note: Exact-Match Feed Reads](#implementation-note-exact-match-feed-reads). The returned `result.index` MUST equal the requested `index`. Implementations MUST NOT return data from a different index, such as the feed's latest entry.

---

#### `swarm_listFeeds`

Return the calling origin's feed records — every feed previously created via `swarm_createFeed` under the caller's normalized origin.

This is an introspection method scoped to the calling origin. It does NOT require any permission grant — neither connection (`swarm_requestAccess`) nor feed-level. Any web page MAY call it.

**Params:** None. The result is determined entirely by the caller's normalized origin. Implementations SHOULD silently ignore any params supplied; strict rejection buys nothing for an introspection-only method.

**Result:**

```javascript
[
  {
    name: string,                  // Feed name (as passed to swarm_createFeed)
    topic: string,                 // 64-character hex topic (no 0x prefix)
    owner: string,                 // Checksummed Ethereum address of the identity that created the feed
    manifestReference: string,     // 64-character hex reference to the feed manifest
    bzzUrl: string,                // "bzz://<manifestReference>"
    createdAt: number,             // ms unix timestamp when createFeed was called
    lastUpdated: number | null,    // ms unix timestamp of last swarm_updateFeed (null otherwise)
    lastReference: string | null   // last reference written via swarm_updateFeed (null otherwise)
  },
  ...
]
```

Returns an empty array `[]` for an origin with no feeds, whatever its connection state: an origin that never connected, or connected without creating a feed, gets `[]`. An origin whose access was revoked still gets the feeds it created (see Behavior).

**Errors:** None expected on the happy path. `-32602` with `data.reason = "rate_limited"` if the calling origin has exceeded its per-origin read budget (see [Resource Exhaustion](#resource-exhaustion)). `-32603` (Internal Error) only on unexpected internal failures (e.g., feed-store read errors).

**Behavior:**

- **Origin scoping is mandatory.** The result MUST contain only feeds created under the calling page's normalized origin. Implementations MUST NOT return feeds created under any other origin.
- **No permission required**, by design:
  - Feed coordinates are deterministic given `(origin, name)` — the calling origin can compute the same set itself given the names of feeds it created. Listing them does not reveal anything beyond what the caller already knows.
  - Feed metadata persists across permission revocation by design (so an origin re-granted access after revocation resumes its prior identity and can continue using its existing feeds). Requiring permission for introspection would create UX friction without security benefit.
  - Symmetric with `swarm_readFeedEntry`: both are read-only operations on data the caller could obtain elsewhere.
- **Owners can differ.** Each feed stays bound to the identity that created it, so after the user switched the origin's active identity, entries can carry different `owner` values, and some can differ from `swarm_getSigningIdentity`'s.
- **`lastUpdated` and `lastReference`** are populated only by `swarm_updateFeed`-style usage. For feeds maintained as journals via `swarm_writeFeedEntry`, both fields stay `null` (those operations do not update the manifest reference).
- **`bzzUrl`** is a stable convenience built from `manifestReference`. For `swarm_updateFeed`-style feeds it resolves to the latest pointed-at content; for journal-style feeds (where the SOC payload is raw bytes rather than a content reference) the manifest URL has limited utility — applications should use `swarm_readFeedEntry` to read journal contents.
- Each call counts against the origin's permission-free read budget.

---

### Low-Level Chunk Methods

The methods specified above are *high-level*: they express publishing and feed intent, and the provider translates that intent into Swarm's underlying chunk operations. This section defines the *low-level* tier — direct read and write access to individual Swarm chunks — plus `swarm_readData`, a verified read of a whole blob.

Every Swarm data structure is ultimately a graph of chunks: **content-addressed chunks (CACs)** for immutable data, and **Single Owner Chunks (SOCs)** — signed chunks at a caller-chosen address — used as the building block for mutable structures such as feeds. Exposing the chunk layer lets dApps and libraries implement higher-level constructs this specification does not define directly — custom manifests, epoch feeds, access-controlled data, and structures not yet supported natively by Bee — without a revision of this specification for each one.

The chunk tier is **additive**. It does not replace the high-level methods, which carry consent semantics, safety guarantees (e.g. the journal overwrite protection of `swarm_writeFeedEntry`), and an efficient bulk-data path. Applications SHOULD prefer the high-level methods where they fit, and reach for the chunk tier only for primitives the high-level methods do not provide.

**Chunk type is explicit.** Swarm carries no in-band chunk-type tag, and a chunk's 32-byte address does not reveal whether it is a CAC or a SOC; Bee currently distinguishes them heuristically (see [SWIP-67](https://github.com/ethersphere/SWIPs/pull/67) and [bee#5445](https://github.com/ethersphere/bee/pull/5445)). This specification sidesteps that ambiguity by making the chunk type explicit in the **method name** — `swarm_publishChunk`/`swarm_readChunk` operate on CACs, `swarm_writeSingleOwnerChunk`/`swarm_readSingleOwnerChunk` operate on SOCs. The provider never has to guess, and the API does not depend on the resolution of the upstream heuristic.

**Chunk size.** A Swarm chunk payload is at most 4096 bytes. This is a protocol constant, advertised as `maxChunkPayloadBytes` in `swarm_getCapabilities` for completeness, but it is not implementation-configurable.

**Not for bulk data.** Each chunk method call crosses a process boundary and incurs IPC overhead. Uploading large content as thousands of individual `swarm_publishChunk` calls is far slower than `swarm_publishData`, which splits content into a BMT tree in a single node operation. Applications SHOULD use `swarm_publishData`/`swarm_publishFiles` for bulk content, `swarm_readData` for bulk reads, and reserve the chunk methods for single chunks and custom structures that cannot be expressed at the high-level tier.

All chunk uploads MUST be pinned on the local node, as with `swarm_publishData`. The implementation selects an appropriate postage batch automatically.

**Type validation on read.** Implementations MUST validate the returned bytes against the requested chunk type using Swarm's standard chunk verification rules — BMT hash recomputation for CACs, SOC signature recovery and `keccak256(identifier_32 || owner_20)` address derivation for SOCs. A mismatch MUST be reported as `chunk_type_mismatch`. See [Implementation Note: Exact-Match Feed Reads](#implementation-note-exact-match-feed-reads) for SOC address derivation and the span-endianness rules; refer to Swarm's chunk-format specification for the canonical byte layout (with [bee-js](https://github.com/ethersphere/bee-js) as a reference implementation).

**Options object.** Several chunk methods accept an optional `options` object. The v1 surface is deliberately narrow — see [Future Extensions](#future-extensions) for the deferred fields and the rationale. `options` MUST be absent or an object; implementations MUST reject any option key they do not recognize with `-32602` and `data.reason = "unsupported_option"`, so that future option additions degrade safely.

---

#### `swarm_publishChunk`

Upload a single content-addressed chunk (CAC).

**Params:**

| Field | Type | Required | Description |
|---|---|---|---|
| `data` | `string \| Uint8Array \| ArrayBuffer` | Yes | Chunk payload. Strings are encoded as UTF-8. 1 to `maxChunkPayloadBytes` (4096) bytes. |
| `span` | `number \| bigint` | No | The value committed in the chunk's 8-byte span field. Defaults to the byte length of `data` (always representable as a `number`). Set explicitly only when constructing intermediate nodes of a custom BMT tree. MUST be a non-negative integer fitting in 8 unsigned bytes (`0 ≤ span ≤ 2⁶⁴ − 1`). Pass a `bigint` when the value exceeds `Number.MAX_SAFE_INTEGER` (2⁵³ − 1); a `number` outside the safe-integer range MUST be rejected with `invalid_span`. |
| `options` | `object` | No | Reserved for future use. No fields are standardized in v1; unknown fields MUST be rejected with `unsupported_option`. |

**Result:**

```javascript
{
  reference: string   // 64-character hex chunk address
}
```

**Errors:** `4001` if the user rejects a per-publish prompt. `4100` if not connected. `4900` if node unavailable. `-32602` if params invalid (`invalid_params` for an empty payload, `payload_too_large`, `invalid_span`, `unsupported_option`).

**Behavior:**
- The chunk address is the BMT hash of `span_8LE || data`. See [Implementation Note: Exact-Match Feed Reads](#implementation-note-exact-match-feed-reads) for the **little-endian** span convention.
- Subject to the **publish permission tier** — the same per-operation consent / auto-approve as `swarm_publishData`.
- CACs are immutable; uploading a chunk that already exists is idempotent and returns an identical reference.
- Chunk payloads are opaque bytes. Per-chunk encryption is the caller's responsibility — the provider stores the bytes as given. (Swarm's encryption primitive lives at the BMT-tree layer, not at single `/chunks` endpoints; see [Future Extensions](#future-extensions).)

---

#### `swarm_readChunk`

Retrieve a single content-addressed chunk by its address. This is a read-only operation on public Swarm data and does NOT require any permission grant — symmetric with `swarm_readFeedEntry`. Any web page MAY call it. Subject to the per-origin rate limits described in [Security Considerations](#resource-exhaustion).

**Params:**

| Field | Type | Required | Description |
|---|---|---|---|
| `reference` | `string` | Yes | 64-character hex chunk address. |
| `options` | `object` | No | Reserved for future use. Unknown fields MUST be rejected with `unsupported_option`. |

**Result:**

```javascript
{
  data: string,        // Base64-encoded chunk payload (span field stripped)
  encoding: "base64",
  span: number | bigint   // Decoded span value; `number` when ≤ Number.MAX_SAFE_INTEGER, `bigint` otherwise
}
```

**Errors:** `4900` if node unreachable, `-32602` for invalid params (`invalid_reference`, `unsupported_option`). `data.reason` is `chunk_not_found` if no chunk exists at the address, `chunk_type_mismatch` if the returned bytes do not validate as a CAC at the requested address, or `rate_limited` if the per-origin budget is exceeded.

**Behavior:**
- Works regardless of node mode; reads consume no stamps. Pre-flight checks MUST be limited to verifying the node's HTTP API is reachable (same rule as `swarm_readFeedEntry`).
- Implementations MUST recompute the BMT hash of the returned bytes (over `span_8LE || data`) and reject any chunk whose hash does not equal the requested `reference` (`chunk_type_mismatch`). A returned SOC at the same address — which is structurally possible — MUST be rejected here; the caller intended a CAC and should use `swarm_readSingleOwnerChunk` for SOCs.
- Implementations MUST NOT report a timeout or connection failure as `chunk_not_found`; those propagate as `-32603`. A node error that the node uses to report a missing chunk MAY be mapped to `chunk_not_found` (see [Node error mapping](#node-error-mapping)).

---

#### `swarm_readData`

Retrieve a whole blob by its root reference — the counterpart of Bee's `GET /bytes/{reference}`. The node fetches the chunk tree and joins it, verifying each chunk against its address. This is a read-only operation on public Swarm data and does NOT require any permission grant. Subject to the per-origin read budget, with blob-size rules below.

**Params:**

| Field | Type | Required | Description |
|---|---|---|---|
| `reference` | `string` | Yes | 64-character hex root reference of the blob. |
| `options` | `object` | No | Reserved for future use. Unknown fields MUST be rejected with `unsupported_option`. |

**Result:**

```javascript
{
  data: string,        // Base64-encoded blob bytes
  encoding: "base64"
}
```

**Errors:** `4900` if node unreachable. `-32602` for `invalid_reference`, `unsupported_option`, `chunk_not_found` (the root, or a chunk under it), `chunk_type_mismatch` (the root is not a CAC at that address), `data_too_large` (+ `limit`, `actual`), or `rate_limited`.

**Behavior:**
- Implementations MUST read and verify the root chunk as a CAC at `reference` first, and take the blob's size from its span. The span's most significant byte carries the erasure-coding redundancy level, not size, and MUST be cleared before the span is read as a length.
- Implementations MUST verify every chunk the node fetches against its address, and SHOULD recover erasure-coded content from parity chunks where the root declares a redundancy level, so a page needs no tree or coding logic of its own.
- **Budget before download.** Before downloading anything beyond the root chunk, the implementation MUST refuse a blob larger than the origin's whole read byte budget with `data_too_large`, and a blob that fits the budget but not what remains of it with `rate_limited`. The root chunk itself counts against the budget even when the read is then refused.
- This revision accepts only unencrypted 32-byte references. An encrypted (64-byte) reference MAY be refused with `invalid_reference`.
- `swarm_readData` does not resolve Swarm manifests. On a manifest reference — such as the one `swarm_publishData` and `swarm_publishFiles` return — it returns the manifest's own bytes.
- Works regardless of node mode; pre-flight limited to HTTP reachability, as for the other reads.

---

#### `swarm_writeSingleOwnerChunk`

Write a Single Owner Chunk at a caller-chosen `identifier`, signed by the origin's active publisher identity. SOCs are the primitive behind every mutable Swarm structure (feeds, epoch feeds, custom mutable references).

**Params:**

| Field | Type | Required | Description |
|---|---|---|---|
| `identifier` | `string` | Yes | 64-character hex (32-byte) SOC identifier, chosen by the caller. |
| `data` | `string \| Uint8Array \| ArrayBuffer` | Yes | Payload. Strings encoded as UTF-8. 1 to `maxChunkPayloadBytes` (4096) bytes; Bee's chunk layer refuses an empty SOC payload. |
| `span` | `number \| bigint` | No | As in `swarm_publishChunk` (same range, same `bigint`-for-out-of-safe-integer-range rule). |
| `options` | `object` | No | Reserved for future use. Unknown fields MUST be rejected with `unsupported_option`. |

**Result:**

```javascript
{
  reference: string,    // 64-character hex SOC address
  owner: string,        // Checksummed Ethereum address of the signing key
  identifier: string    // The identifier (echoed)
}
```

**Errors:** `4001` if the user rejects the prompt. `4100` if the origin lacks basic connection or the implementation cannot surface a permission prompt. `4900` if node unavailable. `-32602` if params invalid (`invalid_identifier`, `invalid_span`, `payload_too_large`, `unsupported_option`, `invalid_params` for an empty payload).

**Behavior:**
- The provider signs the SOC with the origin's **active** publisher identity — the one `swarm_getSigningIdentity` returns. There is no caller-supplied `identityMode` parameter in v1 (see [Future Extensions](#future-extensions)). Key material MUST NOT be exposed to the page context (see [Key Material](#key-material)).
- The SOC address is `keccak256(identifier_32 || owner_20)` — see [Implementation Note: Exact-Match Feed Reads](#implementation-note-exact-match-feed-reads).
- Subject to the **signing permission tier** — it needs the feed grant, the same grant as `swarm_createFeed`, and has its own auto-approve.
- Because the owner is one of the origin's own publisher identities, a dApp can only create SOCs under that identity. It cannot forge SOCs owned by another origin's app-scoped identity or by an arbitrary address. (Identities that are shared across origins by design — `bee-wallet`, `ethereum-wallet` — are discussed in [Identity Model](#identity-model).)
- **No overwrite guarantee.** The same `(owner, identifier)` pair SHOULD be treated as immutable; the behavior of re-writing a SOC at an existing `(owner, identifier)` is undefined and discouraged. Applications that need mutability MUST use distinct identifiers (e.g. the feed pattern `keccak256(topic || index)`), and applications that need overwrite-rejection MUST use `swarm_writeFeedEntry`.

---

#### `swarm_readSingleOwnerChunk`

Retrieve a Single Owner Chunk. Read-only operation on public Swarm data; requires NO permission grant. Subject to the per-origin rate limits described in [Security Considerations](#resource-exhaustion).

**Params:**

| Field | Type | Required | Description |
|---|---|---|---|
| `address` | `string` | Conditional | 64-character hex SOC address. |
| `owner` | `string` | Conditional | Ethereum address of the SOC owner. Required if `address` is not provided. |
| `identifier` | `string` | Conditional | 64-character hex identifier. Required if `address` is not provided. |
| `options` | `object` | No | Reserved for future use. Unknown fields MUST be rejected with `unsupported_option`. |

Exactly one of `address`, or the pair (`owner` + `identifier`), MUST be provided; anything else is `invalid_params`. When the pair is given, the provider derives the address as `keccak256(identifier_32 || owner_20)`.

**Result:**

```javascript
{
  data: string,            // Base64-encoded payload (span field stripped)
  encoding: "base64",
  span: number | bigint,   // Decoded span value; `number` when ≤ Number.MAX_SAFE_INTEGER, `bigint` otherwise
  reference: string,       // 64-character hex SOC address (echoed/derived)
  owner: string,           // Recovered owner address
  identifier: string,
  signature: string        // 130-character hex (65-byte) SOC signature, for caller-side verification
}
```

**Errors:** `4900` if node unreachable, `-32602` if params invalid (`invalid_owner`, `invalid_identifier`, `invalid_reference`, `invalid_params`, `unsupported_option`). `data.reason` is `chunk_not_found` if no SOC exists at the address, `chunk_type_mismatch` if the returned bytes do not validate as a SOC for the requested address, or `rate_limited` if the per-origin budget is exceeded.

**Behavior:**
- Exact-match by address — no heuristics, no feed lookup.
- Implementations MUST parse the returned SOC, recover the signer from the signature, derive `keccak256(identifier_32 || owner_20)`, and reject any chunk whose derived address does not equal the requested address (`chunk_type_mismatch`). A returned CAC at the same address MUST be rejected here.
- Node-mode independent; consumes no stamps; pre-flight limited to HTTP reachability. Transient vs. not-found distinction as in `swarm_readChunk`.

---

#### `swarm_getSigningIdentity`

Return the origin's active publisher identity — the key the provider uses to sign SOCs and new feeds on this origin's behalf. Useful for pre-computing SOC addresses (e.g. for epoch-feed construction) before any write, and as the entry point for acquiring the feed grant without committing to a specific write.

This method is in the **signing permission tier**: the returned address is the dApp's persistent public identity, and exposing it without consent would let arbitrary pages enumerate per-origin identities.

**Params:** None.

**Result:**

```javascript
{
  owner: string,        // Checksummed Ethereum address of the signing key
  identityMode: string  // "app-scoped", "bee-wallet" or "ethereum-wallet"
}
```

**Errors:** `4001` if the user rejects the prompt. `4100` if the origin has not been granted basic connection (`swarm_requestAccess`) or the implementation cannot surface a permission prompt in the current context (e.g. a nested browsing context where consent UI is unavailable).

**Behavior:**
- If the calling origin does not yet hold the feed grant, the implementation MUST prompt for it (the same prompt as the first feeds-tier call) and return the identity on approval. This makes `swarm_getSigningIdentity` a viable bootstrap path for dApps that need to pre-compute SOC addresses before any write.
- Once the grant exists, the implementation MAY still ask the user to approve each call, as for the other signing-tier methods, unless the user has enabled auto-approve for signing on this origin.
- It MUST NOT require the Swarm node to be running.
- The returned `owner` MUST be the identity that `swarm_writeSingleOwnerChunk` and a new `swarm_createFeed` would sign with at that moment. It can change when the user switches the origin's active identity (see [Identity Model](#identity-model)); existing feeds keep the owner that created them.

---

### Implementation Note: Exact-Match Feed Reads

This section is **informative**. It documents a Bee API behavior that affects the correctness of `swarm_writeFeedEntry`'s overwrite protection and of both `swarm_readFeedEntry` paths.

#### The problem

Bee's feed lookup, `GET /feeds/{owner}/{topic}`, cannot be asked for the entry at a given index. It takes an `at` timestamp and an `after` index (0 by default) from which to start a search for the newest entry of the sequence ([`feed.go`](https://github.com/ethersphere/bee/blob/master/pkg/api/feed.go), [`sequence.go`](https://github.com/ethersphere/bee/blob/master/pkg/feeds/sequence/sequence.go)), and answers with the newest entry it reaches, plus that entry's index. A query such as `?index=N` is not an exact read, and on a sparse feed the search can stop at a gap. So:

- On a feed with entries 0 to 5, a read of index 0 through the lookup returns index 5's chunk, with no signal that anything is wrong.
- A pre-write existence probe at index 6 through the lookup finds an entry whenever the search reaches one, falsely flagging index 6 as occupied; on a sparse feed its answer can be wrong either way.

For the journal pattern, this breaks both halves of the contract: explicit-index reads can return the wrong entry, and overwrite protection falsely rejects sparse writes.

#### The fix

Implementations SHOULD derive the SOC address directly and fetch via `GET /chunks/{socAddress}`, which is exact-match.

```
identifier  = keccak256( topic_32 || index_8BE )
socAddress  = keccak256( identifier_32 || ownerAddress_20 )
```

Where:
- `topic_32` — the feed's 32-byte topic (the same topic used elsewhere in the feed API).
- `index_8BE` — the feed index encoded as **8 bytes big-endian**.
- `ownerAddress_20` — the 20-byte Ethereum address of the feed signer.

A 404 from `GET /chunks/{socAddress}` means no entry exists at the exact requested index — translated to `entry_not_found` for reads and "index available" for pre-write probes. Connection errors and timeouts MUST propagate as transient (`-32603` Internal Error), NOT be misclassified as not-found. For 5xx responses see [Node error mapping](#node-error-mapping).

#### When to use which endpoint

Finding the latest entry is what the lookup is for: `GET /feeds/{owner}/{topic}` names it in the `swarm-feed-index` and `swarm-feed-index-next` response headers (`%016x`) that drive auto-increment. Its body is not always that entry: for an entry that holds a reference, the node serves the content the reference points to (see `data` under `swarm_readFeedEntry`). So a latest-entry read takes the index from the lookup and reads the entry at that index by address; sending `Swarm-Only-Root-Chunk: true` keeps the node from streaming content the read does not use. Because the search can stop at a gap, "latest" is reliable only on a contiguous feed; journals that need it SHOULD write with auto-increment rather than sparse explicit indexes.

| Operation | Endpoint | Semantics |
|---|---|---|
| `swarm_readFeedEntry` (no `index`) — latest entry + `nextIndex` | `GET /feeds/{owner}/{topic}`, then `GET /chunks/{socAddress}` | Latest index, then exact-match |
| `swarm_writeFeedEntry` auto-increment — find next writable index | `GET /feeds/{owner}/{topic}` | Latest index |
| `swarm_readFeedEntry` with explicit `index` | `GET /chunks/{socAddress}` | Exact-match |
| `swarm_writeFeedEntry` overwrite probe | `GET /chunks/{socAddress}` | Exact-match |

If `GET /feeds/{owner}/{topic}` cannot answer for a feed that has entries (some nodes fail the lookup once the content the latest entry refers to has expired), an implementation MAY find the latest and next index by probing entries directly by address, stepping as the lookup does (1, 2, 4, … past the highest index found, then bisecting) so that it reads a sparse feed as the node would.

#### Endianness trap

Bee uses **big-endian** for the 8-byte feed index in identifier derivation, but **little-endian** for the 8-byte `span` (payload length) prefix in the SOC body. These are different fields with different conventions in the same operation. Implementations going through `bee-js` get both right via its built-in helpers; implementations writing directly against this spec MUST handle the two conventions separately.

#### Node error mapping

Bee and other node implementations do not always report a missing entry or chunk as 404. Some feed lookups answer HTTP 500 when the entry does not exist, and some chunk reads answer 500 with a "read chunk failed" body. An implementation cannot always tell those from a genuine node failure.

- For feed reads (latest or exact) and the next-index lookup behind feed writes, implementations MAY treat any node 500 as "not found". A "not found" result then means "not found, or the node failed while looking".
- For chunk reads, implementations MAY map a 500 that the node uses for a missing chunk to `chunk_not_found`.
- Connection errors and timeouts MUST NOT be mapped to "not found" anywhere.

One consequence for the overwrite probe of `swarm_writeFeedEntry`: if a 500 counts as "index available", a node failure during the probe lets the write proceed. The node itself may still refuse a second SOC at the same address, but overwrite protection is then only as strong as the node's error reporting.

---

### Identity Model

#### Publisher identities

A **publisher identity** is the secp256k1 key that signs feed updates and SOCs for an origin. Its Ethereum address is the `owner` of everything it signs. This specification defines three modes, reported as `identityMode`:

| `identityMode` | Key | Linkable across origins? | Support |
|---|---|---|---|
| `app-scoped` | A dedicated key per origin, derived from the user's master secret. Never funded, never used for transactions. | No | RECOMMENDED; the default |
| `bee-wallet` | The Swarm node's own wallet key. | Yes: every origin using it shares the address | OPTIONAL |
| `ethereum-wallet` | One of the user's Ethereum accounts whose private key the implementation holds. Accounts whose key lives elsewhere (hardware wallets, remote signers, smart accounts) cannot be used. | Yes: it is the user's public wallet address | OPTIONAL; contested, see below |

Implementations MUST support at least one of `app-scoped` and `bee-wallet`, and MUST list every supported mode in `swarm_getCapabilities().publisherIdentityModes`. When offering a linkable mode, the consent UI SHOULD say that the identity is shared across sites.

> **Contested: `ethereum-wallet`.** This mode is under discussion and may be removed from this specification. Swarm signs a single-owner chunk the way a wallet signs a `personal_sign` message, so a wallet account that owns feeds can be made to sign a feed update by any dApp that gets it to `personal_sign` 32 bytes, which look like gibberish to the user. That hands the dApp the user's feeds. See [Wallet Keys as Publisher Identities](#wallet-keys-as-publisher-identities). Applications that use the mode should be prepared for it to go away.

#### Several identities per origin

- An origin MAY hold several publisher identities, one of them **active**. Implementations MAY let the user pick the identity in the first feeds- or signing-tier prompt, and switch the active one later (in settings, or in a later prompt).
- `swarm_getSigningIdentity`, `swarm_writeSingleOwnerChunk` and a new `swarm_createFeed` use the **active** identity.
- Each feed stays bound to the identity that created it: `swarm_updateFeed` and `swarm_writeFeedEntry` sign with that identity, whichever is active now. This keeps every feed's owner, and so its address, stable.
- Identities and feed records MUST survive disconnection, so a reconnected origin signs with the same keys and finds its feeds again.

#### Deriving app-scoped keys

So that restoring the user's recovery phrase restores every site's identities, in any conforming implementation, an implementation whose keys come from a [BIP-39](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki) recovery phrase SHOULD derive app-scoped keys as follows:

```
digest  = SHA-256( utf8("swarm-provider/publisher-key/v1\n" + permissionKey) )
a, b, c = the first three big-endian 32-bit words of digest, each with its top bit cleared
path    = m/44'/73406'/a'/b'/c'/n'
```

- `permissionKey` is the origin's normalized [permission key](#origin-normalization), and the label ends in a single line feed (`\n`, 0x0a).
- `n` numbers the origin's app-scoped identities: `0` for the first, `1` for the second, and so on. An implementation restoring from a phrase can find an origin's identities by deriving `n = 0, 1, 2, …`.
- The key is derived from the BIP-39 seed (empty passphrase unless the user set one) with [BIP-32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki) secp256k1 derivation.
- Identities an implementation created under an earlier scheme MAY keep their keys: changing an existing identity's key would change the owner of every feed it signed.

Test vectors, with the phrase `abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about` and permission key `myapp.eth`:

| `n` | Path | Address |
|---|---|---|
| 0 | `m/44'/73406'/407696474'/1227618452'/2121415264'/0'` | `0x503f3954CD2F37dc3d30C870BB0bfCAcEbe280e1` |
| 1 | `m/44'/73406'/407696474'/1227618452'/2121415264'/1'` | `0x2aF0A2C773114575eDd623248B01dc3392faB4F1` |

A key that follows the permission key follows its lifetime too: a `bzz://<reference>` origin changes with every redeploy (see [Origin Normalization](#origin-normalization)), and so do its identities.

---

### Permission Model

#### Origin Normalization

Implementations MUST normalize the requesting page's origin to a stable root identity, the **permission key**. For decentralized web contexts where pages are served via local gateways, the HTTP origin (e.g., `http://127.0.0.1:1633`) does NOT represent the content's identity. The permission key is derived from the **user-visible URL** (the address bar), not from `window.location`, which reflects the internal gateway routing.

| Display URL | Permission key | Rule |
|---|---|---|
| `ens://myapp.eth/#/path` | `myapp.eth` | A name is keyed by the lowercased bare name. |
| `myapp.eth/blog` | `myapp.eth` | |
| `bzz://myapp.eth/page` | `myapp.eth` | A name stays keyed by the name, whichever transport scheme displays it. |
| `ipfs://myapp.eth/docs` | `myapp.eth` | |
| `bzz://abc123def.../page` | `bzz://abc123def...` | Content roots are keyed by scheme and root, without path, query or fragment. |
| `ipfs://QmABC.../docs` | `ipfs://QmABC...` | |
| `ipns://host/guide` | `ipns://host` | |
| `rad://z123.../tree` | `rad://z123...` | |
| `web3://0xAbC.../swap` | `web3://0xabc...` | Contract-hosted app (draft ERC-8244) on chain 1: lowercased address. |
| `web3://0xAbC...:100/swap` | `web3://0xabc...:100` | Another chain: the chain ID is part of the key. |
| `https://app.example.com/page` | `https://app.example.com` | Web origins are keyed by scheme and host (and non-default port). |

"Names" are those of the name systems the implementation resolves (for example ENS). The permission key is part of the protocol, not only of the consent store: feed topics and the [key derivations](#deriving-app-scoped-keys) hash it. Implementations MUST therefore produce the same permission key for the same display URL as these rules do, or feeds and keys will not be found across implementations.

A content-addressed `bzz://<reference>` key changes with every redeploy, and with it the origin's grants, feed topics and derived keys. Applications that need these to survive a redeploy SHOULD be served under a name, or from a feed manifest whose reference does not change.

#### Permission Tiers

| Tier | Methods | Grant needed | Prompt |
|---|---|---|---|
| **Public** | `swarm_getCapabilities`, `swarm_readFeedEntry`, `swarm_listFeeds`, `swarm_readChunk`, `swarm_readData`, `swarm_readSingleOwnerChunk` | none | never |
| **Connection** | `swarm_requestAccess` | — | once per origin |
| | `swarm_getUploadStatus` | connection | never |
| **Publish** | `swarm_publishData`, `swarm_publishFiles`, `swarm_publishChunk` | connection | per call, unless auto-approve (publish) |
| **Feeds** | `swarm_createFeed`, `swarm_updateFeed`, `swarm_writeFeedEntry` | connection + feed grant | the first feeds- or signing-tier call acquires the feed grant; later calls MAY prompt unless auto-approve (feeds) |
| **Signing** | `swarm_getSigningIdentity`, `swarm_writeSingleOwnerChunk` | connection + feed grant | as Feeds, with auto-approve (signing) |

Extensions add their own tiers and methods ([Messaging](./swip-draft_provider_api_messaging.md)) or join existing ones ([Encryption Key](./swip-draft_provider_api_encryption.md) joins the signing tier).

The feeds and signing tiers share **one** feed grant; they differ only in which auto-approve skips the per-call prompt. Signing a SOC or disclosing the identity is a different act from writing a feed the app named, so users can allow one without the other.

#### Permission Lifecycle

1. **Connection:** Granted via `swarm_requestAccess` (or a manifest sheet). Persisted per origin.
2. **Publish:** Each publish-tier call MAY require per-operation user approval, unless the user has opted into auto-approve for this origin. If the prompt is rejected, the method MUST reject with `4001`.
3. **Feeds and signing:** Operations that sign with the origin's identity, and the identity-disclosure method `swarm_getSigningIdentity`, need the feed grant on top of the connection. The first such call MAY prompt to acquire it, and the implementation MAY let the user choose the publisher identity there. After that, each call MAY still be approved individually, unless the tier's auto-approve is on. Raw SOC writes are gated here (not under the publish tier) because they involve signing and share key material with feeds. Every rejected prompt is `4001`; `4100` is reserved for a missing connection or a context where no prompt can be surfaced.
4. **Reads and introspection:** `swarm_readFeedEntry`, `swarm_readChunk`, `swarm_readData`, `swarm_readSingleOwnerChunk`, and `swarm_listFeeds` require NO permission grant. The data is either public on Swarm or deterministic given the caller's origin — none of it reveals anything the caller couldn't obtain elsewhere or compute itself. They MUST work without a prior `swarm_requestAccess`, and MUST be rate-limited per origin (see [Security Considerations](#resource-exhaustion)).
5. **Disconnection:** The user can revoke access at any time. Disconnecting MUST revoke the connection, every tier grant and every auto-approval for the origin. The provider MUST emit `disconnect` to the origin's pages and reject subsequent permission-gated requests with `4100`. Permission-free methods MUST continue to function. Publisher identities and feed records MUST be kept (see [Identity Model](#identity-model)), so a re-grant restores identity continuity, and `swarm_listFeeds` keeps returning the previously-created feed records.

Implementations MAY also let the user revoke a single tier or a single auto-approval. That is not a disconnect: no `disconnect` event is emitted.

The prompts and the consent UI are where the user decides, but they are not the security boundary: the part of the implementation that talks to the node MUST check every grant again for every call.

#### Auto-Approve (OPTIONAL)

Implementations MAY offer users the option to auto-approve, per origin, each tier that prompts per call: publish (including CAC chunk uploads), feeds, and signing (including SOC writes and identity disclosure); extensions add their own (messaging sends). When auto-approve is active:
- The implementation MUST still enforce all validation, size limits, and pre-flight checks.
- The implementation SHOULD provide a visual indicator that auto-approve is active.
- The user MUST be able to revoke auto-approve at any time.

Implementations MAY decline to auto-approve, and prompt instead, in contexts where the user may not see what the page is doing (for example a tab driven by automation).

#### Permission Manifests

A site served from Swarm MAY declare the capabilities it needs in a **permission manifest**, so the user can grant them in one consent sheet, with the site's own reasons, instead of meeting a prompt per capability. Implementations SHOULD support permission manifests.

**Location.** The manifest is the file `swarm-manifest.json` at the root of the site's Swarm content: `bzz://<host>/swarm-manifest.json`, where `<host>` is the reference or name the address bar shows. This revision defines manifests only for origins whose content is served from Swarm; other origins use the per-call prompts. The implementation MUST fetch the manifest itself, from the origin's own content, never from a location the page supplies.

**Format.** A UTF-8 JSON object of at most 8 KiB:

```json
{
  "schema": "swarm-manifest/1",
  "name": "My App",
  "description": "Optional, up to 160 characters",
  "capabilities": {
    "swarm": {
      "publish":   { "why": "Up to 140 characters, shown in the consent sheet" },
      "feeds":     { "why": "…" },
      "signing":   { "why": "…" },
      "messaging": { "why": "…" }
    }
  }
}
```

| Field | Rule |
|---|---|
| `schema` | MUST be `"swarm-manifest/1"`. |
| `name` | Required. 1–32 Unicode code points, not blank. |
| `description` | Optional. At most 160 code points. |
| `capabilities.swarm` | Required. One or more of `publish`, `feeds`, `signing` (this specification) and `messaging` ([Messaging](./swip-draft_provider_api_messaging.md)). Each is an object with exactly one field, `why`: 1–140 code points, not blank. |

Implementations MUST treat a manifest as **invalid** if it has any field not listed here (at any level), a value outside these rules, or a text field containing control characters (U+0000–U+001F, U+007F–U+009F), bidirectional formatting characters (U+202A–U+202E, U+2066–U+2069), or line and paragraph separators (U+2028, U+2029). New capabilities will come with a new `schema` version.

**What a capability grants.** Approving a capability grants everything it needs, including its auto-approval, so the app does not prompt for it again:

| Capability | Grants |
|---|---|
| `publish` | connection, auto-approve (publish) |
| `feeds` | connection, a publisher identity (if the origin has none: an `app-scoped` one, or the implementation's default mode where it does not support `app-scoped`), feed grant, auto-approve (feeds) |
| `signing` | connection, a publisher identity (as for `feeds`), feed grant, auto-approve (signing) |
| `messaging` | connection, messaging grant, auto-approve (messaging sends) |

**When it is read.** The implementation SHOULD fetch the manifest when the page calls `swarm_requestAccess`, and MUST check it again before applying any grant that came from it, at least once per loaded document. It MAY cache the result for the rest of that document.

**Consent.** When a manifest names capabilities the user has not yet answered, the implementation shows one consent sheet covering all of them, attributed to the origin, with each capability's `why`. The sheet MUST let the user decline (`4001`), and SHOULD let the user connect without the capabilities' auto-approvals ("ask each time"), in which case the origin is connected and every later action prompts as if there were no manifest. A declined sheet MAY be remembered for the rest of the document, so later calls reject with `4001` without showing it again. Capabilities the origin already holds through other grants MAY be acknowledged without asking.

**Changes.**
- A capability added in a redeployed manifest is asked for, as above.
- A capability removed from the manifest MUST have the grants it created withdrawn. A manifest that disappears (404) or becomes invalid withdraws all of them. The publisher identity is never withdrawn.
- A change to the wording of `why`, `name` or `description` alone SHOULD NOT prompt again.
- If the user changes a manifest-created grant by hand, that grant SHOULD stop being managed by the manifest.

**Failures.**
- A manifest that is absent (404) or invalid is "no manifest": the origin uses the per-call prompts.
- When the fetch fails transiently (network error, 5xx, a stalled body) and the origin has no manifest on record, the implementation SHOULD proceed as if there were none.
- When the fetch fails transiently and the origin does have grants from a manifest, the implementation MUST NOT keep applying those grants without the check. It SHOULD fail permission-gated calls with `4900` (no `data.reason`) and retry the fetch with backoff.

---

### Limits

Implementations MUST enforce upload size, file count, and path length limits. The limits MUST be discoverable via `swarm_getCapabilities`. Recommended defaults:

| Limit | Recommended Default | Notes |
|---|---|---|
| `maxDataBytes` | 10 MiB (10,485,760 bytes) | Applies to both `swarm_publishData` and `swarm_writeFeedEntry` payloads. |
| `maxFilesBytes` | 50 MiB (52,428,800 bytes) | Total size across all entries in `swarm_publishFiles`. |
| `maxFileCount` | 100 files | Per `swarm_publishFiles` call. |
| `maxPathBytes` | 100 (UTF-8 bytes) | Per `files[].path` in `swarm_publishFiles`. Implementations MUST advertise at least 100. |
| `maxChunkPayloadBytes` | 4096 (bytes) | Per `swarm_publishChunk` and `swarm_writeSingleOwnerChunk` payload. Fixed by the Swarm chunk protocol; implementations MUST advertise exactly 4096. |

Implementations MAY use different limits but MUST report them accurately in `swarm_getCapabilities`. Extensions add their own limits to the same object.

The `maxPathBytes` floor of 100 reflects current implementation reality: reference implementations build uploads as USTAR tar archives (`Content-Type: application/x-tar` with `Swarm-Collection: true`), and USTAR's `name` header field is exactly 100 bytes. Without PAX extensions — which are out of scope for v1 — paths longer than 100 UTF-8 bytes cannot be encoded. Future implementations supporting PAX or alternative upload formats MAY advertise a larger `maxPathBytes`.

Not reported in `swarm_getCapabilities`:

- **Feed names:** 1–64 UTF-16 code units (JavaScript `string.length`, so a character outside the Basic Multilingual Plane counts as 2), no `/`, no characters below U+0020.
- **Permission-free read budgets:** see [Resource Exhaustion](#resource-exhaustion).

---

## Rationale

### Why EIP-1193 style?

The `window.ethereum` pattern ([EIP-1193](https://eips.ethereum.org/EIPS/eip-1193)) is well-understood by web3 developers. By following the same request/response pattern and reusing compatible error codes, we minimize the learning curve and allow existing tooling patterns (event listeners, promise-based flows, error handling) to transfer directly.

### Why not expose the full Bee API?

Exposing the full Bee HTTP API to web pages would be a security risk — it includes administrative endpoints (staking, chequebook management, debug APIs) that should never be accessible to arbitrary web content. The provider API exposes only publishing and feed primitives, with user consent gating every operation.

### Why origin normalization?

In decentralized web browsers, all content from IPFS, Swarm, ENS, etc. is typically served through a local gateway at `127.0.0.1`. Using `window.location.origin` would collapse all dApps into a single permission scope. Origin normalization uses the user-visible URL to derive the content's true identity, providing meaningful per-dApp isolation. Because feed topics and derived keys hash the permission key, the rules have to be identical across implementations; a name keeps one key whichever transport scheme displays it, so a site does not lose its feeds when a browser changes how it shows the same name.

### Why app-scoped feed identities?

Without app-scoped identities, any dApp with feed permission could create feeds signed by the user's main wallet key — making feeds from different dApps indistinguishable and creating impersonation risks. App-scoped keys (derived per origin from the user's master secret) provide cryptographic isolation: each dApp's feeds are signed by a unique key that the dApp cannot extract. The linkable modes stay available for applications that want a portable identity, with the user's consent, though using a wallet account is contested (see [Wallet Keys as Publisher Identities](#wallet-keys-as-publisher-identities)).

### Why derive app-scoped keys from the recovery phrase with a fixed recipe?

A key kept only in a browser profile is lost with the profile, and with it the ability to update every feed it owns. Deriving the key from the recovery phrase makes it restorable. Fixing the recipe in the standard (rather than leaving it to each implementation) makes it restorable *in another implementation* too: a user who moves browsers keeps the same identity on every site. Hashing the permission key into hardened path indexes gives each origin a practically unique key without a stored index, and the trailing `n'` lets an origin hold several identities that can be enumerated on restore.

### Why may an origin switch identities?

Users have legitimate reasons to separate personas on one site, or to move a site from an app-scoped key to their wallet identity. Binding each feed to the identity that created it keeps every existing feed's owner and address stable, so switching never breaks a published URL; it only changes what *new* signatures use. The cost is that `swarm_getSigningIdentity` is not a constant, and `swarm_listFeeds` can show more than one owner. Pages that cache the owner should re-read it.

### Why separate feed permissions?

Feeds involve key material (signing) and have long-term implications (the feed URL is permanent, updates are irrevocable). This warrants a separate, more deliberate permission grant beyond "allow this site to upload data."

### Why may feed and signing calls prompt every time?

Every signature is a public, irrevocable statement made under the user's identity, just as every publish spends the user's postage. Implementations that prompt per call give the user the same control over signatures as over uploads; the tier's auto-approve, or a permission manifest, gives a frictionless path to apps the user trusts. Splitting feeds from signing lets a user auto-approve writes to the feeds an app named without also auto-approving arbitrary SOC signatures.

### Why permission manifests?

An app that needs publishing, feeds and messaging otherwise meets the user with three or more prompts, each without context, at moments the app does not control. A manifest states up front what the app needs and why, in one sheet, before the app does anything. Binding it to the origin's own content means the declaration changes only when the site is redeployed, and the rules on changes (new capabilities are asked for, removed ones are withdrawn, wording changes are free) keep the grants in step with what the site says it needs. The format is strict so that a typo cannot silently drop or widen a capability, and the text rules keep site-supplied text from imitating browser UI.

### Why indexed feed entries alongside `updateFeed`?

`swarm_updateFeed` treats a feed as a mutable pointer — it stores a 32-byte Swarm reference at the next sequence index. This is ideal for "latest version" use cases (e.g., a website, a profile).

Many applications also need an **append-only log**: user activity feeds, message history, notification streams. Without indexed entry access, applications must implement a fragile "read current blob → append → re-upload entire blob → update feed" pattern. If the read fails (404, network blip), the entire history is silently overwritten.

`swarm_writeFeedEntry` and `swarm_readFeedEntry` expose the native Swarm feed index, enabling O(1) appends (write one SOC, no reads) and O(N) parallel reconstruction (fetch individual entries by index). The overwrite protection on explicit indices ensures the core safety guarantee: no silent history corruption.

### Why do feed and chunk reads require no permission grant?

Swarm feeds, CACs, and SOCs are public data — anyone who knows the address (or owner + identifier, or owner + topic + index) can read any chunk on the network from any Bee gateway. Adding a per-origin ACL on top would be security theater: gating doesn't restrict access to the data, only routes around the user's node, which any page can do on its own.

What gating *would* restrict is the user's bandwidth, local cache, and connection. Those are the real resources at risk, and they are protected by the **mandatory per-origin rate and bandwidth limits** specified in [Resource Exhaustion](#resource-exhaustion) — including tighter budgets for origins that have never called `swarm_requestAccess`. This puts the control where the actual risk lives, instead of behind a permission prompt that would block legitimate use cases (profile pages, cross-user activity views, link previews) for no security gain.

The same logic extends to `swarm_listFeeds`: feed coordinates are deterministic given `(origin, name)`, so listing them reveals nothing the calling origin could not compute itself.

### Why `swarm_readData`?

Reading a published blob through the chunk methods means fetching and walking its chunk tree in the page, one call per chunk, with no erasure recovery. `swarm_readData` lets the node join the tree, verify every chunk and recover from parity, and hands the page the bytes. It is the verified counterpart of `GET /bytes/{reference}`, which implementations should not expose to pages directly (see [Direct Node Access](#direct-node-access)). Checking the size from the root chunk before downloading the rest keeps one call from spending more than the origin's read budget.

### Why expose chunk operations?

Every Swarm data structure — feeds, manifests, ACT, GSOC, future constructions not yet supported by Bee such as epoch feeds — is built from chunks. Without a chunk tier, every new high-level primitive Swarm gains would require a new method in this specification and a new revision to ship it.

Exposing CAC and SOC read/write at the provider boundary moves that experimentation into the library layer: a library author can build a new manifest format, a new feed scheme, or a new access-control overlay on top of `swarm_publishChunk`/`swarm_readChunk`/`swarm_writeSingleOwnerChunk`/`swarm_readSingleOwnerChunk`, without touching this specification. This mirrors EIP-1193's role on Ethereum, where `eth_sendRawTransaction` and friends are the low-level transport and ergonomic abstractions (ethers, viem) live as libraries.

The chunk tier is constrained the same way the high-level methods are: origin-scoped permissions, host-managed postage, fixed payload size limit, per-origin rate budgets on permission-free reads, and signing-key custody inside the trusted context. It adds expressiveness without weakening the security posture — a 4 KB opaque blob upload is strictly less powerful than a 10 MB `swarm_publishData` call the same origin can already make.

### Why type-explicit method names?

Swarm chunks carry no in-band type tag, and a chunk's address does not encode whether the chunk is a CAC or a SOC. Bee disambiguates heuristically, which has produced real edge cases (see [SWIP-67](https://github.com/ethersphere/SWIPs/pull/67), [bee#5445](https://github.com/ethersphere/bee/pull/5445)).

Rather than wait on a Bee-level resolution or introduce our own heuristics, the API encodes the type in the method name: `swarm_publishChunk`/`swarm_readChunk` for CACs, `swarm_writeSingleOwnerChunk`/`swarm_readSingleOwnerChunk` for SOCs. The caller declares intent, the provider applies the correct upload endpoint and verification rules, and the spec does not depend on the upstream ambiguity ever being resolved.

This also unblocks a clean validation contract: on every read, the provider recomputes the type-specific address from the returned bytes and rejects mismatches as `chunk_type_mismatch`. The caller can rely on "if `swarm_readChunk(addr)` returned bytes, those bytes are the CAC at `addr`" — a guarantee the underlying Bee endpoint does not give.

### Why may a node 500 count as "not found"?

Because the nodes do it. Bee and other node implementations report some missing feed entries and chunks as HTTP 500, and a provider cannot always tell those from real failures. Insisting on "500 is transient" would turn every read of an empty feed slot into an error and break the journal pattern on those nodes. The specification therefore lets implementations follow the node's convention for 500, keeps timeouts and connection failures strictly transient, and says plainly what that costs (see [Node error mapping](#node-error-mapping)).

### Why base64 for read responses?

Feed entry payloads are arbitrary bytes. The response travels from the Bee node through the main process, across IPC to the renderer, and via `postMessage` to the page context. Binary data (`Uint8Array`) does not survive this serialization chain intact across all environments. Base64 is the safe universal encoding. The `encoding: "base64"` field makes the contract explicit so callers know exactly how to decode.

## Backwards Compatibility

This SWIP introduces a new API surface. There are no backwards compatibility concerns as no prior standard for `window.swarm` exists. Implementations that predate this specification SHOULD migrate to conform to these interfaces.

Web pages that do not use `window.swarm` are unaffected. The provider MUST NOT modify any existing browser APIs or globals.

## Security Considerations

### Origin Trust Model

The provider relies on the host application (browser) to supply the correct origin for each request. If the host misidentifies a page's origin, the entire permission model breaks. Implementations MUST derive the origin from the user-visible URL (address bar), not from `window.location` or any value the page can control. The page context MUST NOT be able to influence its own origin identification.

### Direct Node Access

The permission model holds only if pages cannot reach the node some other way. Implementations SHOULD block requests from web content to the node's HTTP API (for example `localhost:1633`, and whichever port the node actually listens on), and SHOULD serve `bzz://` content to pages read-only (`GET` and `HEAD`). Reads that pages need go through the permission-free read methods, which verify what they return.

### Resource Exhaustion

A connected origin could repeatedly publish data up to the size limit, consuming postage stamp capacity and local storage. Implementations SHOULD consider:

- **Rate limiting:** Throttling publish requests per origin per time window.
- **Stamp monitoring:** Warning the user when stamp capacity is running low due to dApp activity.
- **Per-origin accounting:** Tracking cumulative usage per origin so users can identify heavy consumers.

This specification does not mandate a specific rate-limiting scheme for permissioned operations, as appropriate limits depend on the node's capacity and the user's preferences.

#### Permission-Free Reads

The read methods that require no permission grant — `swarm_readFeedEntry`, `swarm_readChunk`, `swarm_readData`, `swarm_readSingleOwnerChunk`, and `swarm_listFeeds` — can be invoked by any page that loads the provider, including pages the user has never granted access to. `swarm_readChunk` and `swarm_readData` in particular widen this to arbitrary addresses, effectively turning the user's node into a public retrieval path on behalf of any page.

The capability itself is benign — the same data is available from any public Swarm gateway — but the user's bandwidth, local cache, and network connection are not. Without throttling, a hostile page could drain the user's bandwidth by issuing high-volume reads, or probe the local cache via timing.

Implementations MUST enforce **per-origin rate and bandwidth limits** on permission-free read methods, shared across them. Origins exceeding the configured budget MUST receive `-32602` with `data.reason = "rate_limited"` (a signal to back off, not an error the dApp should retry immediately). Implementations MAY apply tighter limits to origins that have never called `swarm_requestAccess` than to connected origins. A read whose size is only known after the download MAY be counted, and refused, after the fact; `swarm_readData` sizes the blob first (see its Behavior).

This specification does not mandate specific numeric thresholds, as appropriate limits depend on the node's bandwidth and the user's preferences. Implementations MAY make the limits user-visible and adjustable. As an example, one implementation allows each origin 600 requests and 5 MiB per 60-second window when connected, and 120 requests and 512 KiB when not.

### Iframe and Nested Context Behavior

This specification does not define behavior for `window.swarm` inside iframes or nested browsing contexts. Implementations SHOULD restrict provider availability to the top-level browsing context, and SHOULD ignore provider messages that do not come from it. If an implementation chooses to expose the provider to iframes, the iframe's origin MUST be evaluated independently — it MUST NOT inherit the parent frame's permissions.

### Key Material

Publisher identities involve signing keys. Implementations MUST ensure that:

- Private key material is never exposed to the page context.
- App-scoped key derivation is deterministic (the same origin and identity number always produce the same key) but not reversible (the page cannot derive the master secret, or another origin's key, from its own).
- Signing occurs in a trusted context (main process or secure enclave), never in the renderer or page context.

### Wallet Keys as Publisher Identities

Bee signs a single-owner chunk as an [EIP-191](https://eips.ethereum.org/EIPS/eip-191) "Ethereum Signed Message" over the 32-byte digest `keccak256(identifier ‖ address of the inner content-addressed chunk)` (Bee's `pkg/soc` and `pkg/crypto`). That is byte for byte what a wallet produces for a `personal_sign` of those 32 bytes. When a user's wallet account is also a feed owner (the `ethereum-wallet` mode):

- any dApp that obtains a `personal_sign` of a chosen 32-byte value from that account — a value that looks like gibberish to the user — holds a valid single-owner chunk signature, and can publish, from anywhere, the next entry of any feed the account owns, taking the feed over for readers that follow its latest entry;
- the provider cannot prevent it, because the same key also lives in the user's other wallets (any wallet restored from the same recovery phrase), which sign such messages on their own terms.

An implementation that offers the mode SHOULD always ask the user before a `personal_sign` of exactly 32 bytes, never auto-approve one, and warn that it can authorize a Swarm feed update. Because that protects only the implementation's own wallet, the mode is contested and may be removed from this specification. A future revision could bring wallet-owned feeds back with a signature that a `personal_sign` cannot produce, which needs a change to how Swarm signs single-owner chunks. Until then, the safe choice is a publisher identity dedicated to Swarm, which signs nothing for pages.

### Linkable Identities

`bee-wallet` and `ethereum-wallet` identities are the same for every origin that uses them, so they link the user's activity across sites, and an `ethereum-wallet` identity links it to the user's on-chain history. Two origins sharing such an identity can also sign SOCs under the same owner, including at identifiers the other origin uses; feed topics stay origin-scoped, but raw SOC identifiers do not. Consent UI that offers these modes SHOULD say so, and applications that need per-site separation SHOULD keep the default `app-scoped` mode.

### Manifest Text

Everything in a permission manifest is chosen by the site. The consent sheet MUST present it as the site's own statement, attributed to the origin, and MUST NOT let it stand in for the browser's description of what a capability allows. The length and character rules exist so that a manifest cannot overflow the sheet, reorder text with bidirectional controls, or imitate browser UI.

### Temporary Artifacts

Implementations that create temporary files or directories during upload processing (e.g., for `swarm_publishFiles` manifest construction) MUST clean up these artifacts regardless of whether the upload succeeds or fails.

## Future Extensions

The following capabilities are anticipated for future versions of this specification and are explicitly out of scope for version 1.0:

- **`capabilitiesChanged` event** — Proactive notification when the provider's capabilities change (e.g., node goes offline, stamps exhausted). Currently dApps must poll `swarm_getCapabilities` to detect state changes.
- **`preferredIdentityMode` parameter** — Allow dApps to express a preference for an identity mode, rather than relying solely on user/implementation choice. When added, this parameter SHOULD land on `swarm_createFeed`, `swarm_writeSingleOwnerChunk`, and `swarm_getSigningIdentity` consistently, and a matching permission manifest field SHOULD let a site ask for it up front.
- **`encoding` parameter for the read methods** — Allow callers to request a specific response encoding (e.g., `"utf8"`) to avoid manual base64 decoding for text payloads.
- **Encrypted references in `swarm_readData`**, and write-side options for encryption and redundancy on the publish methods.
- **Permission manifests for origins not served from Swarm.**
- **Expanded chunk `options`** — The chunk methods' `options` object is intentionally empty in v1, with `unsupported_option` providing forward-compat enforcement. Candidate fields for future revisions, each requiring a precise mapping to Bee's endpoint contract before standardization:
  - `trackProgress: true` on writes — provider issues a fresh origin-scoped `tagUid` (returned in the result) usable with `swarm_getUploadStatus`. The page MUST NOT supply a `tag` value directly, to preserve the origin ownership invariant `swarm_getUploadStatus` enforces.
  - `cache: boolean` on reads — pass-through to Bee's cache header.
  - **ACT (Access Control Trie)** on reads — Bee's `/chunks` endpoint accepts ACT headers; once ACT semantics are specified at this layer (key custody, grantee management, history references), it is a natural fit for the chunk tier. Meanwhile the [Encryption Key](./swip-draft_provider_api_encryption.md) extension lets a page build and read ACT data itself with the chunk methods.
  - **Per-chunk encryption** — Bee's encryption primitive currently lives at the `/bytes` BMT-tree layer, not at single `/chunks`. Adding it at the chunk tier requires either a Bee-level addition or a client-side convention; both are out of scope for v1. Pages can encrypt client-side with keys from the Encryption Key extension.
  - **Redundancy (erasure coding)** on writes — a BMT-tree-layer feature in Bee, not a single-chunk one. (`swarm_readData` already recovers erasure-coded content on reads.)
  - **`deferred` upload** — Bee's `swarm-deferred-upload` header is endpoint-general but not guaranteed at `/chunks`; pending verification.

Additionally, the `specVersion` field in `swarm_getCapabilities` is currently a SHOULD. A future revision may promote it to MUST once multiple implementations exist and version negotiation becomes necessary. Until then, `features` is the way to detect extensions.

Implementations MAY experiment with these features, but they are not part of the 1.0 standard interface and MUST NOT be required for conformance.

## Test Cases

### Detection

```javascript
// Provider available
assert(typeof window.swarm !== 'undefined');
assert(typeof window.swarm.request === 'function');
```

### Connection Flow

```javascript
// Request access (user approves)
const result = await window.swarm.requestAccess();
assert(result.connected === true);
assert(typeof result.origin === 'string');
assert(Array.isArray(result.capabilities));

// Capabilities reflect connection
const caps = await window.swarm.getCapabilities();
assert(caps.canPublish === true || typeof caps.reason === 'string');
```

### Publish Data

```javascript
const result = await window.swarm.publishData({
  data: '<h1>Hello Swarm</h1>',
  contentType: 'text/html',
  name: 'greeting.html',
});
assert(/^[0-9a-f]{64}$/.test(result.reference));
assert(result.bzzUrl === `bzz://${result.reference}`);
```

### Publish Files

```javascript
const encoder = new TextEncoder();
const result = await window.swarm.publishFiles({
  files: [
    { path: 'index.html', bytes: encoder.encode('<h1>Hello</h1>'), contentType: 'text/html' },
    { path: 'style.css', bytes: encoder.encode('body { color: red; }'), contentType: 'text/css' },
  ],
  indexDocument: 'index.html',
});
assert(/^[0-9a-f]{64}$/.test(result.reference));
```

### Feed Lifecycle

```javascript
// Create feed
const feed = await window.swarm.createFeed({ name: 'my-blog' });
assert(feed.feedId === 'my-blog');
assert(/^[0-9a-f]{64}$/.test(feed.manifestReference));

// Publish content, then update feed
const content = await window.swarm.publishData({
  data: '<h1>Post #1</h1>',
  contentType: 'text/html',
});
const updated = await window.swarm.updateFeed({
  feedId: 'my-blog',
  reference: content.reference,
});
assert(updated.bzzUrl === feed.bzzUrl); // Same stable URL
assert(typeof updated.index === 'number'); // Sequence index returned
```

### Error Handling

```javascript
// Calling without access
try {
  await window.swarm.publishData({ data: 'test', contentType: 'text/plain' });
  assert.fail('Should have thrown');
} catch (err) {
  assert(err.code === 4100); // Unauthorized
}

// Invalid params
try {
  await window.swarm.publishData({}); // missing data and contentType
  assert.fail('Should have thrown');
} catch (err) {
  assert(err.code === -32602); // Invalid params
}
```

### Post-Disconnect Behavior

```javascript
// After the user revokes access, permission-gated methods reject with 4100.
// Permission-free methods (getCapabilities, readFeedEntry, readChunk, readData,
// readSingleOwnerChunk, listFeeds) MUST continue to function.
let disconnected = false;
window.swarm.on('disconnect', ({ origin }) => { disconnected = true; });
// (simulate disconnect via browser UI, then:)
assert(disconnected === true);
try {
  await window.swarm.publishData({ data: 'test', contentType: 'text/plain' });
  assert.fail('Should have thrown');
} catch (err) {
  assert(err.code === 4100); // Unauthorized
}

// Permission-free introspection still works:
const feeds = await window.swarm.listFeeds();
assert(Array.isArray(feeds)); // Returns previously-created feed records

const caps = await window.swarm.getCapabilities();
assert(caps.canPublish === false);
assert(caps.reason === 'not-connected');
```

### Feed Idempotency

```javascript
// Creating the same feed twice returns identical metadata
const feed1 = await window.swarm.createFeed({ name: 'my-blog' });
const feed2 = await window.swarm.createFeed({ name: 'my-blog' });
assert(feed1.manifestReference === feed2.manifestReference);
assert(feed1.owner === feed2.owner);
assert(feed1.topic === feed2.topic);
```

### Feed Journal (Write and Read Entries)

```javascript
// Create feed first
const feed = await window.swarm.createFeed({ name: 'activity' });

// Write entries (auto-increment)
const w0 = await window.swarm.writeFeedEntry({
  name: 'activity',
  data: JSON.stringify({ action: 'post', id: 1 }),
});
assert(w0.index === 0);

const w1 = await window.swarm.writeFeedEntry({
  name: 'activity',
  data: JSON.stringify({ action: 'post', id: 2 }),
});
assert(w1.index === 1);

// Read latest entry
const latest = await window.swarm.readFeedEntry({ name: 'activity' });
assert(latest.encoding === 'base64');
assert(latest.index === 1);
assert(latest.nextIndex === 2);

// Decode the payload
const bytes = Uint8Array.from(atob(latest.data), c => c.charCodeAt(0));
const entry = JSON.parse(new TextDecoder().decode(bytes));
assert(entry.action === 'post');
assert(entry.id === 2);

// Read specific index; nextIndex is index + 1, not the next free index
const first = await window.swarm.readFeedEntry({ name: 'activity', index: 0 });
assert(first.index === 0);
assert(first.nextIndex === 1);

// Read all entries in parallel
const all = await Promise.all(
  Array.from({ length: latest.nextIndex }, (_, i) =>
    window.swarm.readFeedEntry({ name: 'activity', index: i })
  )
);
assert(all.length === 2);
```

### Feed Journal — Overwrite Protection

```javascript
// Writing at an occupied index is rejected
await window.swarm.writeFeedEntry({ name: 'activity', data: 'first', index: 0 });
try {
  await window.swarm.writeFeedEntry({ name: 'activity', data: 'duplicate', index: 0 });
  assert.fail('Should have thrown');
} catch (err) {
  assert(err.code === -32602);
  assert(err.data.reason === 'index_already_exists');
}
```

### Feed Journal — Size Limit

```javascript
const { maxDataBytes } = (await window.swarm.getCapabilities()).limits;
try {
  await window.swarm.writeFeedEntry({ name: 'activity', data: new Uint8Array(maxDataBytes + 1) });
  assert.fail('Should have thrown');
} catch (err) {
  assert(err.code === -32602);
  assert(err.data.reason === 'payload_too_large');
}
```

### Feed Journal — Sparse-Index Writes (Exact-Match Requirement)

```javascript
// Sparse writes are valid: writing index 5 with no entries 1-4 must succeed.
// An implementation that probed through the feed lookup would see index 0
// and falsely reject this write.
await window.swarm.createFeed({ name: 'sparse' });
await window.swarm.writeFeedEntry({ name: 'sparse', data: 'first', index: 0 });
const w5 = await window.swarm.writeFeedEntry({ name: 'sparse', data: 'fifth', index: 5 });
assert(w5.index === 5);

// Explicit-index reads are exact: each returns its own index's data, whatever
// the feed lookup would have found.
const decode = (r) => new TextDecoder().decode(Uint8Array.from(atob(r.data), c => c.charCodeAt(0)));
const r0 = await window.swarm.readFeedEntry({ name: 'sparse', index: 0 });
assert(r0.index === 0 && decode(r0) === 'first');
const r5 = await window.swarm.readFeedEntry({ name: 'sparse', index: 5 });
assert(r5.index === 5 && decode(r5) === 'fifth');

// An empty slot between them is not found, not the nearest entry.
try {
  await window.swarm.readFeedEntry({ name: 'sparse', index: 3 });
  assert.fail('Should have thrown');
} catch (err) {
  assert(err.data.reason === 'entry_not_found');
}
```

### Feed Journal — Cross-User Read

```javascript
// Read another user's feed using raw topic + owner from their published objects
const otherUserEntry = await window.swarm.readFeedEntry({
  topic: 'a1b2c3...64-char-hex-topic...',
  owner: '0xOtherUserSignerAddress',
  index: 0,
});
assert(otherUserEntry.encoding === 'base64');
assert(typeof otherUserEntry.index === 'number');
```

### Feed Journal — Empty Feed

```javascript
// Reading latest from empty feed returns structured error
await window.swarm.createFeed({ name: 'empty-feed' });
try {
  await window.swarm.readFeedEntry({ name: 'empty-feed' });
  assert.fail('Should have thrown');
} catch (err) {
  assert(err.code === -32602);
  assert(err.data.reason === 'feed_empty');
}
```

### Feed Read Without Any Permission

```javascript
// readFeedEntry requires NO permission — no requestAccess, no feed grant.
// An origin that has never called requestAccess can still read public feeds.
const entry = await window.swarm.readFeedEntry({
  topic: 'a1b2c3...64-char-hex-topic...',
  owner: '0xSomeAddress',
  index: 0,
});
assert(typeof entry.data === 'string');
assert(entry.encoding === 'base64');
```

### Read a Blob

```javascript
// Upload raw bytes elsewhere (e.g. POST /bytes on any node), then read them
// back through the user's node without any grant:
const blob = await window.swarm.readData({ reference: rawBytesReference });
assert(blob.encoding === 'base64');

// A manifest reference returns the manifest's bytes, not the file:
const { reference } = await window.swarm.publishData({ data: 'hi', contentType: 'text/plain' });
const manifestBytes = await window.swarm.readData({ reference });
assert(atob(manifestBytes.data) !== 'hi');
```

### List Feeds — Origin-Scoped Introspection

```javascript
// Returns feeds previously created under the calling origin
await window.swarm.createFeed({ name: 'user-feed' });
await window.swarm.createFeed({ name: 'comments' });

const feeds = await window.swarm.listFeeds();
assert(Array.isArray(feeds));
assert(feeds.length >= 2);

const userFeed = feeds.find(f => f.name === 'user-feed');
assert(userFeed.bzzUrl.startsWith('bzz://'));
assert(/^[0-9a-f]{64}$/.test(userFeed.topic));
assert(/^0x[0-9a-fA-F]{40}$/.test(userFeed.owner));
assert(typeof userFeed.createdAt === 'number');
```

### List Feeds — Empty for Un-Granted Origin

```javascript
// listFeeds requires NO permission. An origin with no granted access
// and no created feeds simply gets an empty array — never an error.
const feeds = await window.swarm.listFeeds();
assert(Array.isArray(feeds));
assert(feeds.length === 0);
```

### Re-Connection Without Re-Prompt

```javascript
// Second requestAccess call returns existing state without prompting
const first = await window.swarm.requestAccess();
const second = await window.swarm.requestAccess();
assert(second.connected === true);
assert(second.origin === first.origin);
```

### Capabilities Before Connection

```javascript
// getCapabilities works without prior requestAccess
const caps = await window.swarm.getCapabilities();
assert(caps.canPublish === false);
assert(caps.reason === 'not-connected');
assert(Array.isArray(caps.features));
assert(Array.isArray(caps.publisherIdentityModes));
assert(typeof caps.limits.maxDataBytes === 'number');
assert(typeof caps.limits.maxPathBytes === 'number');
assert(caps.limits.maxPathBytes >= 100);
// specVersion is optional (SHOULD) in 1.0
if (caps.specVersion) assert(typeof caps.specVersion === 'string');
```

### Signing Identity Matches New Writes

```javascript
const { owner } = await window.swarm.getSigningIdentity();
const soc = await window.swarm.writeSingleOwnerChunk({
  identifier: '00'.repeat(32),
  data: 'hello',
});
assert(soc.owner === owner);
```

## Implementation

[Freedom Browser](https://github.com/solardev-xyz/freedom-browser) implements every method of this specification and of the Messaging extension; the Encryption Key extension comes with [PR #1153](https://github.com/solardev-xyz/freedom-browser/pull/1153). Its developer documentation, [`docs/swarm-provider-api.md`](https://github.com/solardev-xyz/freedom-browser/blob/main/docs/swarm-provider-api.md), describes its behavior method by method. The main parts are:

- **Provider injection:** [`src/main/webview-preload.js`](https://github.com/solardev-xyz/freedom-browser/blob/main/src/main/webview-preload.js) — `window.swarm` in the page's main world, before page scripts run
- **Consent and routing:** [`src/renderer/lib/swarm-provider.js`](https://github.com/solardev-xyz/freedom-browser/blob/main/src/renderer/lib/swarm-provider.js) — tiers, prompts, permission manifest checks
- **Enforcement:** [`src/main/swarm/swarm-provider-ipc.js`](https://github.com/solardev-xyz/freedom-browser/blob/main/src/main/swarm/swarm-provider-ipc.js) — re-checks every grant, validates params and limits, read budgets, dispatch
- **Publishing, chunks and feeds:** [`publish-service.js`](https://github.com/solardev-xyz/freedom-browser/blob/main/src/main/swarm/publish-service.js), [`chunk-service.js`](https://github.com/solardev-xyz/freedom-browser/blob/main/src/main/swarm/chunk-service.js), [`feed-service.js`](https://github.com/solardev-xyz/freedom-browser/blob/main/src/main/swarm/feed-service.js)
- **Identities and grants:** [`feed-store.js`](https://github.com/solardev-xyz/freedom-browser/blob/main/src/main/swarm/feed-store.js), [`swarm-permissions.js`](https://github.com/solardev-xyz/freedom-browser/blob/main/src/main/swarm/swarm-permissions.js)
- **Permission manifests:** [`permission-manifests.js`](https://github.com/solardev-xyz/freedom-browser/blob/main/src/main/swarm/permission-manifests.js)
- **Origin normalization:** [`src/shared/origin-utils.js`](https://github.com/solardev-xyz/freedom-browser/blob/main/src/shared/origin-utils.js)

Where Freedom does not yet match this draft, the work is tracked in its issue tracker:

- Freedom's current releases offer the `ethereum-wallet` mode. Its next core, which is being migrated to Rust, drops the mode because of the takeover risk in [Wallet Keys as Publisher Identities](#wallet-keys-as-publisher-identities), and always asks before a 32-byte `personal_sign`.
- Permission manifests are read from `freedom-manifest.json` (schema `freedom-manifest/1`); reading `swarm-manifest.json` is [#1159](https://github.com/solardev-xyz/freedom-browser/issues/1159).
- App-scoped keys are numbered by a stored counter, not derived with the recipe above, so they are not restorable from the phrase alone ([#1155](https://github.com/solardev-xyz/freedom-browser/issues/1155)).
- `maxDataBytes` is not yet enforced on `swarm_writeFeedEntry` ([#1053](https://github.com/solardev-xyz/freedom-browser/issues/1053)).
- `connect` is not emitted when a manifest sheet connects the origin, and `disconnect` reaches only the active tab ([#1158](https://github.com/solardev-xyz/freedom-browser/issues/1158)).
- Per-file `contentType` in `swarm_publishFiles` is ignored; types are inferred from the path.

[Swarmit](https://github.com/flotob/swarmit) — a decentralized message board — exercises the core publish/feed flows from a consuming dApp, using `swarm_requestAccess`, `swarm_publishData`, `swarm_createFeed`, `swarm_writeFeedEntry`, `swarm_readFeedEntry`, and `swarm_listFeeds`. [Freedom Community](https://github.com/flotob/freedom-community) builds community apps on feeds and GSOC messaging.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
