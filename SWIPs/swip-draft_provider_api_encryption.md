---
SWIP: <to be assigned>
title: Swarm Provider API — Encryption Key Extension
author: Florian Glatz (@heckerhut)
discussions-to: https://discord.com/channels/799027393297514537/1239813439136993280
status: Draft
type: Standards Track
category: Interface
created: 2026-11-10
requires: <Swarm Provider API SWIP>
---

## Simple Summary

An extension to the [Swarm Provider API (`window.swarm`)](./swip-draft_provider_api.md)
that gives each site an **encryption key** of its own: a secp256k1 key, derived
from the user's recovery phrase, that the site can use for key agreement
(ECDH) but never sees. Pages get the public key and the shared secrets they ask
for, and do the encryption themselves.

## Abstract

Applications on Swarm that keep data private — group chats, community
archives, shared documents, access-controlled content — need a key per user
that others can encrypt to, and that the user can use to decrypt. The core
specification's publisher identities are signing keys, and two of their modes
(`bee-wallet`, `ethereum-wallet`) are shared across sites, so they are unsafe
for key agreement: a site that could compute shared secrets with a shared key
could read what was encrypted to that key for another site.

This extension adds a separate per-site key used only for key agreement, and
two methods:

- **`swarm_getEncryptionKey`** — the site's public key, to publish so others
  can encrypt to it.
- **`swarm_deriveSharedSecrets`** — ECDH between the site's key and up to
  `maxSharedSecretKeys` public keys in one call, returning each shared x
  coordinate.

The key is derived from the user's BIP-39 recovery phrase with a fixed recipe,
so restoring the phrase restores it, in any conforming implementation. The
private key never leaves the trusted context.

## Motivation

Swarm content is public. Applications that need privacy encrypt before they
write, and the hard part is not the cipher but the keys: who holds which key,
how a new member gets the group key, and how a user gets their keys back on a
new device.

The common answer is envelope encryption: a random content or group key
encrypts the data, and the group key is "wrapped" once per member with a key
derived from an ECDH shared secret between the group's wrapping key and the
member's public key. That needs, per user, a long-lived key pair whose public
half others can find and whose private half can do ECDH. Today a page has two
options, both bad:

1. **Generate a key in the page** and keep it in browser storage. It is lost
   with the browser profile, cannot be restored from anything the user backs
   up, and is readable by any script that runs in the page.
2. **Reuse a signing identity.** The core specification deliberately never
   exposes private keys, and two identity modes are shared across sites.
   Doing ECDH with a shared key would let any site with signing access derive
   the secrets protecting another site's data.

A provider-held, per-site, phrase-derived key solves both: it survives device
loss through the phrase the user already backs up, no other site can use it,
and pages never touch the private half.

### Design Goals

Identical to the core specification, plus:

- **Key agreement only.** The key never signs anything, so a signature cannot
  be tricked out of it and its public key cannot be confused with an identity.
- **Per site, never shared.** Two sites never share an encryption key,
  whatever publisher identities they use.
- **Restorable everywhere.** The derivation is fixed by this specification, so
  the same phrase gives the same key for a site in every conforming
  implementation.
- **Primitive, not protocol.** The provider exposes ECDH and nothing above it.
  Applications choose their own KDF, cipher and envelope format, and can build
  Swarm's access control (ACT) on it.

## Specification

This document is an **extension**. It adds methods, error reasons and a limit
to the core Swarm Provider API, and places the methods in the core's signing
tier. It changes none of the existing method semantics. An implementation of
this extension MUST also conform to the core specification.

### Feature Detection

An implementation of this extension MUST include the string `"encryption-key"`
in the core `swarm_getCapabilities().features` array, and report
`maxSharedSecretKeys` in `limits`:

```javascript
const caps = await window.swarm.getCapabilities()
if (caps.features?.includes('encryption-key')) {
  // swarm_getEncryptionKey and swarm_deriveSharedSecrets are available
}
```

### Convenience Methods

```javascript
window.swarm.getEncryptionKey()                          // swarm_getEncryptionKey
window.swarm.deriveSharedSecrets({ publicKeys, options }) // swarm_deriveSharedSecrets
```

Each MUST be equivalent to calling `window.swarm.request()` with the
corresponding method name and params.

### The Site Encryption Key

Each origin — each [permission key](./swip-draft_provider_api.md#origin-normalization)
— has one **encryption key**: a secp256k1 key pair whose private key the
implementation holds.

- It is used **only** for ECDH on the origin's behalf. Implementations MUST NOT
  sign anything with it, and MUST NOT use it for any other origin.
- It is **independent of the publisher identity**. It is the same whichever
  identity the origin signs with, and different for every origin, also when
  origins share a `bee-wallet` or `ethereum-wallet` publisher identity.
- It is **not rotatable**. Disconnecting the origin does not change it.
  Applications that need rotation do it one layer up, by re-wrapping group
  keys (envelope encryption).
- It **follows the permission key**. A `bzz://<reference>` origin changes with
  every redeploy, and so does its encryption key; an application that
  encrypts SHOULD be served under a name, or from a feed manifest whose
  reference does not change, or it will not be able to read what was encrypted
  to it before a redeploy.

#### Key Derivation

An implementation whose keys come from a [BIP-39](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki)
recovery phrase MUST derive the encryption key as follows, so that a restored
phrase gives the same key for the same origin in any conforming
implementation:

```
digest  = SHA-256( utf8("swarm-provider/encryption-key/v1\n" + permissionKey) )
a, b, c = the first three big-endian 32-bit words of digest, each with its top bit cleared
path    = m/44'/73407'/a'/b'/c'
```

- `permissionKey` is the origin's normalized permission key, exactly as the
  core specification's origin normalization produces it (for example
  `myapp.eth`, `bzz://<reference>`, `https://app.example.com`). The label ends
  in a single line feed (`\n`, 0x0a).
- The key is derived from the BIP-39 seed (empty passphrase unless the user set
  one) with [BIP-32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki)
  secp256k1 derivation along `path`.
- Three hardened 31-bit indexes give 93 bits of the digest, so two origins
  practically never share a key, and no index needs to be stored.

An implementation whose keys do not come from a BIP-39 phrase MUST still give
each origin its own key with the properties above, and SHOULD derive it
deterministically from whatever master secret it has.

**Test vectors.** Phrase
`abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about`:

| Permission key | Path | Compressed public key |
|---|---|---|
| `myapp.eth` | `m/44'/73407'/1359753964'/1747624480'/1277407076'` | `035d600fe9b13fb91f71e00fb423c133ab67e809999712b67149bc86287d271d7e` |
| `bzz://abababababababababababababababababababababababababababababababab` | `m/44'/73407'/1301474304'/1411201871'/182715752'` | `037e38bda264bdad75fcf121d79458bca61c730c19f70e21fc5898b0610d2d66a4` |
| `https://app.example.com` | `m/44'/73407'/1369640046'/207123139'/853429575'` | `0392e199f2ba7b687eb98402bc1913fb0a5edd25f696df60c73962c9b1d13619b9` |

---

### Methods

#### `swarm_getEncryptionKey`

Return the origin's encryption public key. Publish it — for example in one of
the origin's feeds — so others can encrypt to it.

**Params:** None.

**Result:**

```javascript
{
  publicKey: string,    // 66-char hex, 33-byte compressed secp256k1 public key
  curve: "secp256k1"
}
```

**Errors:** `4001` if the user rejects the prompt. `4100` if the origin lacks
basic connection or a prompt cannot be surfaced (`not_connected`,
`feed_not_granted`).

**Behavior:**
- In the core **signing tier**: it needs the feed grant, and the first call
  acquires it (see [Permission Model](#permission-model)).
- It MUST NOT require the Swarm node to be running.
- The result MUST be the same on every call for the same origin and the same
  recovery phrase.

---

#### `swarm_deriveSharedSecrets`

Compute ECDH between the origin's encryption key and each of a list of public
keys.

**Params:**

| Field | Type | Required | Description |
|---|---|---|---|
| `publicKeys` | `string[]` | Yes | 1 to `maxSharedSecretKeys` secp256k1 public keys, each compressed (66 hex, starting `02` or `03`) or uncompressed (130 hex, starting `04`). A `0x` prefix MAY be given. |
| `options` | `object` | No | Reserved. Unknown fields rejected with `unsupported_option`. |

**Result:**

```javascript
{
  sharedSecrets: string[]   // In the same order as publicKeys: each shared point's x coordinate, 64 hex
}
```

Each value is the 32-byte big-endian x coordinate of `d · P`, where `d` is the
origin's private key and `P` the given public key: the same value the other
party computes from its own private key and the origin's public key.

**Errors:** `4001` if the user rejects the prompt. `4100` as for
`swarm_getEncryptionKey`. `-32602` with:

| Reason | Meaning |
|---|---|
| `invalid_public_key` | `publicKeys` is missing, empty or not an array, or one of its entries is not a valid point on secp256k1. `data.index` gives the position of the offending entry, when there is one. |
| `payload_too_large` | More than `maxSharedSecretKeys` keys. `data.limit` and `data.actual` give the cap and the count. |
| `unsupported_option` | An unrecognized `options` field. |

**Behavior:**
- In the core **signing tier**, like `swarm_getEncryptionKey`.
- **One call, one prompt.** An implementation that prompts per call MUST cover
  the whole list with one prompt, so an application can wrap a group key for
  every member at once.
- Implementations MUST validate that every key is a point on secp256k1 (not the
  point at infinity) before using the private key with it, and MUST reject the
  whole call if any key fails.
- It MUST NOT require the Swarm node to be running.

**Using the result.** A shared secret is key material, not a key.
Applications MUST NOT use it directly as a cipher key. They MUST derive keys
from it with a KDF and domain separation: either the derivation a protocol
they implement defines (such as Swarm's ACT derivation below), or one of their
own (for example HKDF-SHA-256 with an application- and purpose-specific
`info`).

```javascript
// One AES-GCM key per group member from one prompt.
const { sharedSecrets } = await window.swarm.deriveSharedSecrets({ publicKeys: memberKeys })
const memberAesKeys = await Promise.all(sharedSecrets.map(async (secret) => {
  const material = await crypto.subtle.importKey('raw', hexToBytes(secret), 'HKDF', false, ['deriveKey'])
  return crypto.subtle.deriveKey(
    { name: 'HKDF', hash: 'SHA-256', salt: new Uint8Array(32),
      info: new TextEncoder().encode('my-app/member-key/v1') },
    material,
    { name: 'AES-GCM', length: 256 },
    false,
    ['encrypt', 'decrypt'],
  )
}))
```

#### Access Control (ACT)

The shared secret also lets a page use Swarm's Access Control Trie with the
origin's encryption key as its ACT key, building and reading the grantee data
itself with the core chunk methods. Bee derives ACT keys from the ECDH x
coordinate as `keccak256(x ‖ nonce)` — nonce `0x00` for the lookup key, `0x01`
for the access-key decryption key — where `x` is encoded **without leading zero
bytes** (as Go's `big.Int.Bytes()` produces it). About one shared secret in 256
has a leading zero byte; a page that hashes the full 32 bytes derives a
different key in those cases. The page MUST strip leading zero bytes from the
returned value before hashing it to interoperate.

---

### Limits

`swarm_getCapabilities().limits` is extended with:

| Limit | Recommended Default | Notes |
|---|---|---|
| `maxSharedSecretKeys` | 256 | Maximum `publicKeys` per `swarm_deriveSharedSecrets` call. Implementations MUST advertise at least 1. |

### Permission Model

Both methods are in the core **signing tier**: they need the connection and the
feed grant, the first signing- or feeds-tier call acquires the grant, and later
calls MAY prompt unless the user has enabled auto-approve for signing on the
origin. The core permission manifest's `signing` capability therefore covers
them.

Implementations SHOULD name the encryption key in the prompt (as distinct from
the publisher identity), so the user knows the site is asking to use its key
rather than to sign.

### Encoding

Hex in and out, as in the core specification: public keys and shared secrets
are lowercase hex without `0x` in results; inputs are case-insensitive and MAY
carry `0x`.

## Rationale

### Why a separate key instead of the publisher identity?

A publisher identity signs. Using one key for both signing and key agreement is
possible but couples two lifetimes and two threat models, and two of the core
identity modes are shared across sites: with `bee-wallet` or
`ethereum-wallet`, every site the user signs for would hold the same private
key, so any of them could derive the secrets protecting another's data. A
separate key per permission key, never used to sign, has none of these
problems, and it does not change when the user switches publisher identity.

### Why ECDH, and not encrypt/decrypt methods?

Encryption formats differ by application — group envelopes, per-message keys,
ACT, ratchets — and each needs different ciphers, nonces and framing. A provider
that offered `encrypt`/`decrypt` would have to standardise one format, or
several, and would still not fit the next application. ECDH is the one
operation that needs the private key; everything above it can run in the page
with WebCrypto. Returning the shared secret to the page does not weaken the
model: the page is the site, and the site is the party the user lets use this
key. What the page cannot do is learn the private key, use another site's key,
or sign with it.

### Why derive the key from the recovery phrase with a fixed recipe?

A key that lives only in one browser profile is lost with it, and with it
everything encrypted to the user. Deriving it from the phrase makes it
restorable without a separate backup. Fixing the recipe in the standard makes
it restorable *across implementations*: a member of a private community can
open it in another conforming browser, restore the same phrase, and still read
what was encrypted to them. The label is vendor-neutral for that reason, and
because the key can never change once data has been encrypted to it, the
recipe is versioned (`/v1`) rather than revised.

### Why batch up to `maxSharedSecretKeys` keys per call?

Wrapping a group key for a group of 50 members would otherwise mean 50
prompts. One call with one prompt covers them all, and the cap keeps a single
call's cost bounded.

### Why the signing tier?

Using the key is a sensitive act of the same kind as signing — it uses a
private key on the user's behalf — and the tier already has the grant,
prompts and auto-approve the encryption key needs. A separate tier would add a
grant and a manifest capability for no practical gain.

## Backwards Compatibility

Purely additive. Implementations without this extension omit the methods and
the `"encryption-key"` feature string. No core method changes.

## Security Considerations

Inherits the core specification's origin trust model and key-custody rules.
Additionally:

- **Private key custody.** The private key MUST NOT reach the page or any
  untrusted context, and ECDH MUST run in the trusted context.
- **Invalid-curve and small-subgroup inputs.** Every input key MUST be
  validated as a point on secp256k1 before it is combined with the private key.
  Without validation, a hostile page could submit points off the curve to learn
  information about the private key.
- **ECDH as an oracle.** A page with the grant can compute the shared secret
  with any public key, including the ephemeral key of a message someone
  encrypted to the origin. That is the method's purpose: the origin can read
  what was sent to it. It also means the protection of data encrypted to an
  origin's key is the user's grant to that origin. Users SHOULD only allow
  sites whose code they trust with that data, and implementations SHOULD make
  per-call prompts and auto-approval for this tier visible and revocable.
- **No signatures.** Because the key never signs, a page cannot use it to
  produce a signature that could be mistaken for the user's or the site's.
- **Linkability.** The public key is per origin and the same in every
  implementation for the same phrase. Publishing it links the user's activity
  *within* that origin across devices and browsers, which is the point; it
  does not link activity across origins.
- **Origin changes.** Because the key follows the permission key, an origin
  that moves (a redeployed `bzz://<reference>`, a renamed site) gets a new key.
  Applications that rely on the key SHOULD be served from a stable origin, and
  SHOULD plan a re-wrap step if they move.
- **KDF required.** The raw ECDH x coordinate is not uniformly random as a
  bit string. Applications MUST pass it through a KDF before use.

## Test Cases

### Feature Detection

```javascript
const caps = await window.swarm.getCapabilities()
if (caps.features.includes('encryption-key')) {
  assert(typeof caps.limits.maxSharedSecretKeys === 'number')
  assert(caps.limits.maxSharedSecretKeys >= 1)
}
```

### Public Key

```javascript
const { publicKey, curve } = await window.swarm.getEncryptionKey()
assert(curve === 'secp256k1')
assert(/^0[23][0-9a-f]{64}$/.test(publicKey))
// Stable across calls
assert((await window.swarm.getEncryptionKey()).publicKey === publicKey)
```

### Shared Secret Against the Generator

```javascript
// ECDH with the generator point G (the public key of private key 1) returns
// the x coordinate of the origin's own public key.
const G = '0279be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798'
const { publicKey } = await window.swarm.getEncryptionKey()
const { sharedSecrets } = await window.swarm.deriveSharedSecrets({ publicKeys: [G] })
assert(sharedSecrets[0] === publicKey.slice(2))
```

For the test-vector phrase and permission key `myapp.eth`, the shared secret
with `G` is `5d600fe9b13fb91f71e00fb423c133ab67e809999712b67149bc86287d271d7e`, and
with `02c6047f9441ed7d6d3045406e95c07cd85c778e4b8cef3ca7abac09b95c709ee5`
(the public key of private key 2) it is
`ba6f305451ef361d0627ff26e3fb27703da0a4bb668143d8eb0c246baba3de29`.

### Both Sides Agree

```javascript
// The other party, holding private key k, computes the same secret from the
// origin's public key (here with @noble/curves).
const { publicKey } = await window.swarm.getEncryptionKey()
const peerPriv = secp256k1.utils.randomPrivateKey()
const peerPub = bytesToHex(secp256k1.getPublicKey(peerPriv, true))
const { sharedSecrets } = await window.swarm.deriveSharedSecrets({ publicKeys: [peerPub] })
const theirs = bytesToHex(secp256k1.getSharedSecret(peerPriv, publicKey, true).slice(1))
assert(sharedSecrets[0] === theirs)
```

### Invalid Key

```javascript
try {
  await window.swarm.deriveSharedSecrets({ publicKeys: [G, '02' + '00'.repeat(32)] })
  assert.fail('should reject')
} catch (err) {
  assert(err.code === -32602)
  assert(err.data.reason === 'invalid_public_key')
  assert(err.data.index === 1)
}
```

### Too Many Keys

```javascript
const { maxSharedSecretKeys } = (await window.swarm.getCapabilities()).limits
try {
  await window.swarm.deriveSharedSecrets({ publicKeys: Array(maxSharedSecretKeys + 1).fill(G) })
  assert.fail('should reject')
} catch (err) {
  assert(err.code === -32602)
  assert(err.data.reason === 'payload_too_large')
}
```

## Future Work

- **Key rotation** at the provider level (a versioned key per origin), if
  application-level re-wrapping proves insufficient.
- **Provider-built ACT**, once ACT semantics are specified at the provider
  layer (see the core specification's Future Extensions).
- **Other curves** (for example X25519), should Swarm adopt them for its own
  encryption.

## Implementation

[Freedom Browser](https://github.com/solardev-xyz/freedom-browser) implements
this extension in [PR #1153](https://github.com/solardev-xyz/freedom-browser/pull/1153):
the derivation in `src/main/identity/derivation.js`, the methods in
`src/main/swarm/swarm-provider-ipc.js`, and the developer documentation in
`docs/swarm-provider-api.md`. The test vectors above come from its test suite.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
