# Agent Network wire format (Pilot profile)

Status: draft 1 (2026-10-02). Licence: CC BY 4.0.

This document specifies the bytes two Agent Network Clients exchange over a Pilot Protocol stream, and how a
receiver decides whether to accept them. It describes a *profile* — one fixed way of using — Pilot Protocol's
`dataexchange` frame format and `decision` signed objects, as linked by the Client today:

| Module | Version | Used for |
|---|---|---|
| `github.com/pilot-protocol/dataexchange` | v0.2.2 | frames, governed frames, payload hash, receiver-side frame verifier |
| `github.com/pilot-protocol/common` (package `decision`) | v0.5.13 | Intent, Decision, Mandate: validation, canonical encoding, hashing, signing, verification, enforcer |

It is written so that the format can be implemented without reading Pilot source code. Conformance is
checked with the test vectors in `../vectors/` (see `../vectors/README.md`). Where this text and the vectors
disagree, the vectors (generated from the module versions above) are authoritative and this text is a bug.

The key words MUST, MUST NOT, SHOULD, MAY are used as in RFC 2119. "Reference receiver" means the
Client's receive path built on the Pilot modules above. Every section ends with **Derived from**, naming the
Pilot file and function the behaviour was taken from (descriptions only; no code is reproduced).

## Contents

1. Conventions
2. Frames
3. Governed frame
4. Signed objects: Intent, Decision, Mandate
5. Payload hash
6. Receiver verification
7. Agent Network profile
8. Interoperability notes
9. Rule identifiers (index)

---

## 1. Conventions

- **u16 / u32 / u64**: unsigned integers, big-endian (network order), 2 / 4 / 8 bytes.
- **i64**: signed 64-bit integer, two's complement, big-endian, 8 bytes.
- **hex**: lowercase hexadecimal, two characters per byte, no prefix. "64-hex" means exactly 64 characters
  from `0-9a-f`.
- **base64**: the standard alphabet of RFC 4648 §4 (`A-Z a-z 0-9 + /`) **with** `=` padding.
- **SHA-256**: FIPS 180-4. **Ed25519**: RFC 8032 (pure Ed25519, no prehash, no context).
- **Unix time**: integer seconds since 1970-01-01T00:00:00Z. "now" is the verifier's clock truncated to whole
  seconds.
- **JSON**: RFC 8259. "Go JSON encoding" means the exact output rules in §8.2.
- String lengths are **byte** lengths of the UTF-8 encoding unless stated otherwise.

---

## 2. Frames

### 2.1 Layout

A frame is a header of 8 bytes followed by the payload:

| Offset | Size | Field | Meaning |
|---|---|---|---|
| 0 | 4 | `type` (u32) | frame type, see 2.2 |
| 4 | 4 | `length` (u32) | number of payload bytes that follow |
| 8 | `length` | `payload` | opaque bytes |

There is no magic number, checksum, version byte or terminator. Frames are written back-to-back on a
reliable byte stream (a Pilot stream on port 1001). Example: the acknowledgement `ACCEPTED` is
`00000001 00000008 4143434550544544`.

### 2.2 Types

| Value | Name | Used by this profile |
|---|---|---|
| 1 | text | inner type of every governed frame; acknowledgements |
| 2 | binary | no |
| 3 | json | no |
| 4 | file | no (for type 4 the Pilot reader additionally splits a 2-byte name length + name off the payload; irrelevant here because type 4 is never accepted) |
| 5 | trace | no |
| 6 | reserved | no |
| 7 | file stream | no |
| 8 | governed | every business message (invite, inbox, control) |
| 9 | governed file stream | no |

### 2.3 Size limits

- **Frame cap.** A reader rejects a frame whose header `length` exceeds the configured cap, *before*
  reading any payload byte (rule **F-CAP**). Pilot's default cap is 64 MiB; the Client requires the cap to be
  exactly **65536 bytes** (the Pilot setting `PILOT_DATAEXCHANGE_MAX_FRAME=65536`; the Client refuses to
  start otherwise). The comparison is `length > 65536` → reject; a 65536-byte payload is within the cap.
  (Pilot accepts that setting only in the range 65536 ≤ value < 2^31 and silently falls back to the default for
  other values — which is why the Client checks the effective value at startup.)
- **Envelope limit (profile).** After a frame is read, the Client rejects it if its payload is longer than
  **16384 bytes** (rule **P-ENVELOPE-SIZE**). This limit applies to the *outer* governed frame payload (the
  whole JSON envelope, §3), not to the inner text. With the profile's fixed field values the envelope overhead
  is about 1.1 KB plus 4/3 of the inner payload (base64), so the inner text budget is roughly 11.4 KB.
- A sender MUST NOT write a frame larger than the receiver's cap; the profile sender's envelopes are always
  ≤ 16384 bytes.

### 2.4 Reading

The reader reads exactly 8 header bytes, then exactly `length` payload bytes. If the stream ends before
either is complete, the read fails (rule **F-SHORT**). A reader MUST NOT allocate `length` bytes up front on
the strength of the header alone (the reference grows its buffer as bytes arrive).

The profile exchanges exactly one request frame and one acknowledgement frame per connection (§7.7).
Bytes after the first frame are not interpreted by the receiver.

### 2.5 Writing

The writer emits the 8-byte header then the payload. `length` is the payload length.

**Derived from:** dataexchange v0.2.2 `dataexchange.go`: type constants, `DefaultMaxFrameSize`,
`MaxFrameSize` (initialisation from the environment), `Frame`, `WriteFrame`, `ReadFrame`, `readBounded`.
Client: `service.go` `FrameCap` / `Open`, `wire.go` `accept`.

---

## 3. Governed frame

A governed frame is a frame of **type 8** whose payload (the *envelope*) is one JSON object that carries an
inner data frame plus the sender's signed Intent and Decision for exactly that inner frame.

### 3.1 Envelope members

| Member | JSON type | Required | Meaning |
|---|---|---|---|
| `version` | integer (u16) | yes | envelope version; MUST be `1` |
| `type` | integer (u32) | yes | inner frame type; one of 1, 2, 3, 4 (profile: MUST be 1) |
| `filename` | string | no | inner file name; only for inner type 4; omitted when empty |
| `payload` | string (base64) or `null` | yes | inner payload bytes, base64 (§1) |
| `disclosure` | object | no | optional disclosure binding; omitted when absent (profile: MUST be absent or `null`) |
| `intent` | object | yes | Intent, §4.2 |
| `decision` | object | yes | Decision, §4.3 |

A sender writes the members in exactly the order of the table, with the Go JSON encoding of §8.2, no
insignificant whitespace, `filename` and `disclosure` omitted when empty/absent. An **empty inner payload is
encoded as `null`** (not `""`); the profile never sends one.

### 3.2 Decoding

The reference decoder:

1. Rejects a frame whose type is not 8, or whose payload exceeds the frame cap.
2. Parses the payload as exactly one JSON object. It **rejects** unknown members at any nesting level and any
   non-whitespace data after the object (rule **G-JSON**). Further leniencies of the reference decoder are listed
   in §8.3; senders MUST NOT rely on them.
3. Applies the structural validation of 3.3; any failure rejects the frame.

### 3.3 Structural validation (applied when encoding and when decoding)

In this order; the first failure rejects:

1. `version` = 1 and `type` ∈ {1, 2, 3, 4} (rule **G-VERSION-TYPE**).
2. If `type` = 4: `filename` MUST be non-empty, valid UTF-8, ≤ 255 bytes, contain neither `/` nor `\`, and be
   its own last path element. Otherwise `filename` MUST be empty (rule **G-FILENAME**).
3. Decoded inner payload length ≤ frame cap.
4. The Intent passes Intent validation (§4.2.2) (rule **G-INTENT**).
5. Both `intent.signature` and `decision.signature` are non-empty strings (rule **G-UNSIGNED**). (Only
   non-emptiness is checked here; the signatures are verified in §6.)
6. `intent.action` equals the action for the inner type: 1 → `data.send.text`, 2 → `data.send.binary`,
   3 → `data.send.json`, 4 → `file.share` (rule **G-ACTION**).
7. Payload binding (rule **G-PAYLOAD-HASH**): without `disclosure`, `intent.payload_hash` MUST equal the
   payload hash of §5 over (inner type, filename, inner payload). With `disclosure`, Pilot instead binds a
   disclosure object; this profile does not use it (§7.4), so its rules are not specified here.
8. The Decision passes Decision validation (§4.3.2) (rule **G-DECISION**).

When encoding, the sender additionally fails if the encoded envelope would exceed the frame cap.

**Derived from:** dataexchange v0.2.2 `governed.go`: `GovernedFrame`, `NewGovernedFrame`,
`GovernedFrame.Validate`, `verifyPayloadBinding`, `EncodeGovernedFrame`, `DecodeGovernedFrame`,
`GovernedAction`, `validGovernedFilename`; `dataexchange.go` `maxFilenameLen`.

---

## 4. Signed objects: Intent, Decision, Mandate

All three objects share one model: a JSON object for transport, a separate **canonical binary encoding** used
for hashing and signing, an Ed25519 signature over the canonical bytes, and validation rules that must hold
before a canonical encoding exists at all. JSON member order, spacing and escaping never affect hashes or
signatures.

### 4.1 Common definitions

**Canonical writer.** A canonical encoding is the concatenation of fields, each written as one of:

- *str(s)*: u32 byte length of `s`, then the bytes of `s` as given (UTF-8; no normalisation, no terminator).
- *u16(n)*, *u64(n)*, *i64(n)*: as in §1.

**Domain strings.** The first field of every canonical encoding is a *str* of a domain string:
`pilot-intent-v1`, `pilot-intent-v1/delegated`, `pilot-decision-v1`, `pilot-mandate-v1`.

**Hash.** `Hash(obj)` = lowercase hex of SHA-256(canonical bytes). It is defined only for valid objects.
The signature is not part of the canonical bytes, so it is not covered by the hash.

**Signature.** Ed25519 signature (64 bytes) over the canonical bytes themselves (not over the hash), encoded
as base64 (88 characters, ending in `==`). Verification decodes the base64 string, requires exactly 64 bytes,
and verifies with the 32-byte public key; a decoding failure and a verification failure are both rejections.

**Identifier** (used for ids, tenant, agent, key, provider, constraint keys): 1–128 bytes, valid UTF-8, first
character an ASCII letter or digit, each further character an ASCII letter, digit or one of `. _ : / @ -`.

**Action name**: 1–128 bytes, first character `a-z`, each further character `a-z`, `0-9` or one of `. _ -`.

**Text(max, empty-allowed)**: at most `max` bytes, valid UTF-8, empty only where allowed, and no character
below U+0020 and no U+007F. (C1 controls U+0080–U+009F and all other Unicode characters are allowed.)

**Validity window** (Intent, Decision): `issued_at` > 0, `expires_at` > `issued_at`, and
`expires_at − issued_at` ≤ 300.

**Freshness at time now** (Intent, Decision, Mandate): reject if `issued_at` > now + 60 ("not yet valid");
reject if `expires_at` < now ("expired"). An object is therefore still fresh at `now = expires_at`, and may be
issued up to 60 s in the future.

**Constraint** object: `{"key": identifier, "operator": one of eq, max, min, one_of, redact, require,
"value": text(1024, empty allowed)}`; within one list, (key, operator) pairs MUST be unique. Constraint lists
are sorted for canonical encoding by key, then operator, then value, each compared as raw bytes.

### 4.2 Intent

#### 4.2.1 JSON members (in sender order)

| Member | Type | Omitted when empty | Notes |
|---|---|---|---|
| `version` | integer u16 | no | MUST be 1 |
| `id` | string | no | identifier |
| `tenant_id` | string | no | identifier |
| `agent_id` | string | no | identifier |
| `action` | string | no | action name |
| `resource` | string | no | text(1024), non-empty |
| `mandate_id` | string | **yes** | identifier when present |
| `audience` | string | **yes** | text(256) |
| `purpose` | string | **yes** | text(256) |
| `payload_hash` | string | no | 64-hex |
| `risk` | string | no | `low`, `medium`, `high` or `critical` |
| `issued_at` | integer i64 | no | Unix time |
| `expires_at` | integer i64 | no | Unix time |
| `nonce` | string | no | 32-hex (128 bits) |
| `key_id` | string | no | identifier |
| `signature` | string | no | base64 Ed25519 signature |

#### 4.2.2 Validation

`version` = 1; `id`, `tenant_id`, `agent_id`, `key_id` are identifiers; `action` is an action name;
`resource` is non-empty text(1024); `mandate_id` is an identifier if non-empty; `audience` and `purpose` are
text(256) (may be empty); `payload_hash` is 64-hex; `risk` is one of the four values; `nonce` is 32-hex;
validity window holds. Uppercase hex is invalid.

#### 4.2.3 Canonical encoding

The Intent is **delegated** if any of `mandate_id`, `audience`, `purpose` is non-empty.

1. str(domain): `pilot-intent-v1/delegated` if delegated, else `pilot-intent-v1`
2. u16(version)
3. str(id), str(tenant_id), str(agent_id), str(action), str(resource), str(payload_hash), str(risk)
4. i64(issued_at), i64(expires_at)
5. str(nonce), str(key_id)
6. only if delegated: str(mandate_id), str(audience), str(purpose) — all three, even if some are empty

Vectors: `intent-offer`, `intent-invite` (delegated, as in the profile), `intent-plain` (base domain),
`intent-audience-only` (delegated with empty `mandate_id`/`purpose`).

### 4.3 Decision

#### 4.3.1 JSON members (in sender order)

| Member | Type | Omitted when empty | Notes |
|---|---|---|---|
| `version` | integer u16 | no | MUST be 1 |
| `id` | string | no | identifier |
| `intent_hash` | string | no | 64-hex, Hash of the Intent |
| `tenant_id` | string | no | identifier |
| `agent_id` | string | no | identifier |
| `outcome` | string | no | `allow`, `deny`, `constrain` or `approval_required` |
| `reasons` | array of string | **yes** | |
| `constraints` | array of Constraint | **yes** | |
| `policy_revision` | integer u64 | no | |
| `revocation_epoch` | integer u64 | no | |
| `provider_id` | string | no | identifier |
| `issued_at` | integer i64 | no | |
| `expires_at` | integer i64 | no | |
| `key_id` | string | no | identifier |
| `signature` | string | no | base64 |

#### 4.3.2 Validation

`version` = 1; `id`, `tenant_id`, `agent_id`, `provider_id`, `key_id` are identifiers; `intent_hash` is
64-hex; `outcome` is one of the four values; at most 16 reasons, each non-empty text(256); at most 32
constraints, each valid, (key, operator) unique; `outcome` = `constrain` if and only if there is at least one
constraint; validity window holds.

#### 4.3.3 Canonical encoding

1. str(`pilot-decision-v1`), u16(version)
2. str(id), str(intent_hash), str(tenant_id), str(agent_id), str(outcome)
3. u16(number of reasons), then str(reason) for each reason **in the given order**
4. u16(number of constraints), then for each constraint **in sorted order** (4.1): str(key), str(operator),
   str(value)
5. u64(policy_revision), u64(revocation_epoch)
6. str(provider_id), i64(issued_at), i64(expires_at), str(key_id)

Vectors: `decision-offer`, `decision-invite`, `decision-constrain` (reasons order kept, constraints sorted).

### 4.4 Mandate

#### 4.4.1 JSON members (in sender order)

| Member | Type | Omitted when empty | Notes |
|---|---|---|---|
| `version` | integer u16 | no | MUST be 1 |
| `id` | string | no | identifier |
| `tenant_id` | string | no | identifier |
| `subject_agent_id` | string | no | identifier |
| `actions` | array of string | no | 1–64 |
| `resource_prefixes` | array of string | no | 1–64 |
| `audience` | string | no | `*` or identifier |
| `purpose` | string | no | non-empty text(256) |
| `constraints` | array of Constraint | **yes** | ≤ 32 |
| `required_approvals` | integer u16 | **yes** (when 0) | ≤ 32 |
| `revocation_epoch` | integer u64 | no | ≥ 1 |
| `issued_at` | integer i64 | no | |
| `expires_at` | integer i64 | no | |
| `key_id` | string | no | identifier |
| `signature` | string | no | base64 |

In JSON, `actions` and `resource_prefixes` keep the order the issuer gave; only the canonical encoding sorts.

#### 4.4.2 Validation

`version` = 1; `id`, `tenant_id`, `subject_agent_id`, `key_id` are identifiers; 1–64 actions, each `*`, an
action name, or an action name followed by `.*`, no duplicates; 1–64 resource prefixes, each `*` or non-empty
text(1024), no duplicates; `audience` is `*` or an identifier; `purpose` non-empty text(256); at most 32 valid
constraints with unique (key, operator); `required_approvals` ≤ 32; `revocation_epoch` ≥ 1; `issued_at` > 0,
`expires_at` > `issued_at`, `expires_at − issued_at` ≤ 7 776 000 (90 days).

#### 4.4.3 Canonical encoding

1. str(`pilot-mandate-v1`), u16(version)
2. str(id), str(tenant_id), str(subject_agent_id)
3. u16(number of actions), then str(action) for each action **sorted bytewise**
4. u16(number of resource prefixes), then str(prefix) for each prefix **sorted bytewise**
5. str(audience), str(purpose)
6. u16(number of constraints), then str(key), str(operator), str(value) for each in sorted order (4.1)
7. u16(required_approvals), u64(revocation_epoch), i64(issued_at), i64(expires_at), str(key_id)

#### 4.4.4 Verification

`Verify(mandate, key, now)`: validation (4.4.2), then freshness at now, then signature. (Pilot also defines
a "mandate ceiling" that checks an Intent against a Mandate; this profile does **not** use it — see §7.5.)

Vectors: `mandate-grant-alice`, `mandate-grant-bob` (profile), `mandate-generic-sorting` (sorting,
optional members, non-ASCII purpose).

### 4.5 Signing procedure (sender)

For each object: fill every member except `signature`; validate; compute the canonical bytes; sign them with
Ed25519; base64 the 64-byte signature into `signature`. The Decision is created after the Intent is final,
with `intent_hash` = Hash(Intent).

**Derived from:** common v0.5.13 `decision/decision.go`: `SchemaVersion`, domain constants, `MaxClockSkew`,
`MaxIntentTTL`, `MaxDecisionTTL`, `Intent`, `Decision`, `Constraint`, `Intent.Validate`, `Decision.Validate`,
`Intent.Canonical`, `Decision.Canonical`, `Hash`, `Sign`, `SignWith`, `Verify`, `verifySignature`,
`verifyFresh`, `validateWindow`, `validateConstraint`, `validateIdentifier`, `validAction`, `validateText`,
`lowerHex`, `canonicalWriter`, `NewNonce`; `decision/mandate.go`: `Mandate`, `Mandate.Validate`,
`Mandate.Canonical`, `Mandate.Hash`, `Mandate.Sign`, `Mandate.Verify`, `validMandateAction`,
`validMandateResource`, `validMandateAudience`.

---

## 5. Payload hash

The value placed in `intent.payload_hash` for a governed frame without disclosure:

1. Compute the 32-byte digest D = SHA-256 over the concatenation of:
   - the 36 ASCII bytes `pilot-dataexchange-governed-frame-v1` followed by one zero byte (37 bytes total);
   - u32(inner frame type);
   - u32(byte length of filename);
   - u32(byte length of inner payload);
   - the filename bytes (empty for the profile);
   - the inner payload bytes.
2. `payload_hash` = lowercase hex of **SHA-256(D)** — that is, the digest is hashed a second time and only the
   second hash is hex-encoded.

Example: inner type 1, no filename, empty payload →
`c2d8819f57f35e091730d082f60fd973353698c373ef4f6a2948b700f8c5902a` (vector `payload-text-empty`).

**Derived from:** dataexchange v0.2.2 `governed.go` `GovernedPayloadHash`; common v0.5.13
`decision/decision.go` `HashPayload`.

---

## 6. Receiver verification

### 6.1 Order and outcome codes

The receiver answers every request with one acknowledgement (§7.7) whose code is determined by the
**first** failing step below. Steps are listed in the order the reference receiver applies them; the "stage"
names match the vectors. Steps marked **[Pilot]** are generic Pilot behaviour; **[Profile]** steps are the
Client's own rules.

| # | Stage | Check | Code on failure |
|---|---|---|---|
| 0 | (transport) | **[Profile]** The Pilot peer address of the connection belongs to an enrolled contact whose registry key matches the pinned transport key, and the daemon trusts it. Not covered by vectors. | `DENIED_PEER` |
| 1 | `read_frame` | **[Pilot]** Read one frame (§2.4): F-CAP, F-SHORT. | `DENIED_FRAME` |
| 2 | `profile_frame` | **[Profile]** Outer payload ≤ 16384 bytes (P-ENVELOPE-SIZE), then outer type = 8 (P-OUTER-TYPE). | `DENIED_FRAME` |
| 3 | `decode_governed` | **[Pilot]** Decode and structurally validate the envelope (§3.2, §3.3): G-JSON, G-VERSION-TYPE, G-FILENAME, G-INTENT, G-UNSIGNED, G-ACTION, G-PAYLOAD-HASH, G-DECISION. | `DENIED_FRAME` |
| 4 | `profile_inner` | **[Profile]** Inner type = 1, `filename` empty, `disclosure` absent or null (P-INNER). | `DENIED_FRAME` |
| 5 | `profile_resource` | **[Profile]** `intent.resource` matches §7.3 with the receiver's own party name (P-RESOURCE). | `DENIED_RESOURCE` |
| 6 | `profile_resource` | **[Profile]** The case id taken from the resource is known locally (P-CASE). | `DENIED_UNKNOWN_CASE` |
| 7 | (state) | **[Profile]** The case's peer is the connection's peer (step 0). Not covered by vectors. | `DENIED_PEER` |
| 8 | `verify_governed` | **[Pilot + Profile]** Decision frame verifier, §6.2. | `DENIED_PROOF` |
| 9 | `profile_payload` | **[Profile]** Kind-specific payload checks, §6.3. | `DENIED_GRANT` / `DENIED_CONTROL` / `DENIED_MESSAGE` |
| 10 | (state) | **[Profile]** Transactional case-state checks (case active, duplicates, limits, results). Specified in `CASE_PROTOCOL.md`; not covered by vectors. | various |

A conforming receiver MUST produce the same code as the reference for every vector. The order *within* a
step that yields one code (e.g. which G-rule fires first) is diagnostic only and not observable on the wire.

### 6.2 Decision frame verifier (step 8)

Inputs: the decoded envelope, the receiver's own party name `self`, the sending party `peer` with its pinned
intent and authority public keys, the case scope `S`, and the clock `now`. The resource kind (`invite`,
`inbox` or `control`) is the one parsed in step 5.

In this order; the first failure rejects with `DENIED_PROOF`:

1. **[Pilot]** Structural validation of §3.3 again (already passed).
2. **[Pilot]** Resource binding (V-RESOURCE): `intent.resource` MUST equal the resource the receiver
   expects, `agent:<self>/<kind>/<S.case_id>/g<S.generation>` (§7.3). (Pilot asks the integrator for the
   expected resource; the profile computes it as stated.)
3. **[Pilot, Profile key store]** Intent key lookup (V-INTENT-KEY) with (`intent.tenant_id`,
   `intent.agent_id`, `intent.key_id`). The profile key store returns the peer's pinned intent key only if
   tenant = `anv-client-m1`, agent = `peer`, and key id = `<peer>-intent`; anything else fails.
4. **[Pilot]** Intent verification: validation (§4.2.2), freshness at now (V-INTENT-FRESH), signature with the
   key from 3 (V-INTENT-SIG).
5. **[Pilot, Profile key store]** Decision key lookup (V-DECISION-KEY) with (**`intent.tenant_id`**,
   `decision.key_id`) — note that the tenant comes from the Intent. The profile returns the peer's pinned
   authority key only if tenant = `anv-client-m1` and key id = `<peer>-authority`.
6. **[Pilot]** Decision verification for this Intent, in order:
   1. Decision validation (§4.3.2), freshness at now (V-DECISION-FRESH), signature with the key from 5
      (V-DECISION-SIG);
   2. `decision.intent_hash` = Hash(intent) (V-BIND-HASH);
   3. `decision.tenant_id` = `intent.tenant_id` and `decision.agent_id` = `intent.agent_id`
      (V-BIND-TENANT-AGENT);
   4. `decision.expires_at` ≤ `intent.expires_at` (V-BIND-EXPIRY);
   5. `decision.issued_at` ≥ `intent.issued_at` − 60 (V-BIND-ISSUED).
7. **[Pilot, Profile values]** Minimum state (V-STATE): the profile's minimum is policy revision **1** and
   revocation epoch **S.generation**; reject if `decision.policy_revision` < 1 or `decision.revocation_epoch`
   < S.generation. (The tenant must be `anv-client-m1`.)
8. **[Profile]** Authority ceiling (V-CEILING). All of the following MUST hold:
   - `intent.action` = `data.send.text`;
   - `intent.agent_id` = `peer`;
   - `intent.audience` = `agent:<self>`;
   - `intent.purpose` = the scope purpose `schedule:<scope hash>` (§7.2);
   - `intent.mandate_id` = `<S.case_id>-<peer>`;
   - `intent.expires_at` ≤ S.expires_at and `decision.expires_at` ≤ S.expires_at;
   - `decision.provider_id` = `<peer>-authority`;
   - `decision.revocation_epoch` = S.generation and `decision.policy_revision` = S.policy_revision;
   - `decision.reasons` and `decision.constraints` are both empty.
9. **[Pilot]** Outcome (V-OUTCOME): `allow` permits delivery. `deny` and `approval_required` reject.
   (`constrain` would be evaluated against frame attributes by Pilot, but it can never reach this step in the
   profile because step 8 rejects any constraint.)

What is **not** checked at this step: nonce or intent-id uniqueness (the verifier keeps no replay cache —
duplicate suppression is by business message id, `CASE_PROTOCOL.md`), the relation between `intent.id`
and the payload (checked in 6.3), and the Mandate (checked only for invites, 6.3). Pilot's own dataexchange
*service* (not used by the Client, which reads frames itself) additionally rejects a second governed frame with
the same (tenant, agent, intent id) until that Intent expires; a receiver MUST NOT apply such a rule in this
profile, because a resend deliberately reuses the intent id with a fresh proof.

### 6.3 Payload checks (step 9) [Profile]

All payloads are UTF-8 JSON that MUST be **byte-identical** to the Go JSON encoding (§8.2) of the value they
decode to, with no unknown members, no trailing data, no whitespace (rule "strict JSON"). Field-name
matching is exact in effect, because a differently spelled name would re-encode differently.

- **invite** (resource kind `invite`) → `DENIED_GRANT` unless:
  `intent.id` = `invite`; the payload is a strict-JSON Mandate (M-JSON); `Verify(mandate, peer authority key,
  now)` succeeds (M-SIG, M-FRESH, validation); and the Mandate is exactly the profile grant from `peer` to
  `self` for S (M-PROFILE): `id` = `<S.case_id>-<peer>`, `tenant_id` = `anv-client-m1`, `subject_agent_id` =
  `peer`, `audience` = `agent:<self>`, `purpose` = scope purpose, `key_id` = `<peer>-authority`,
  `revocation_epoch` = S.generation, `expires_at` = S.expires_at, `required_approvals` = 0, no constraints,
  `actions` = exactly [`data.send.text`], `resource_prefixes` = exactly [`agent:<self>/inbox/<S.case_id>/g<S.generation>`].
- **control** (resource kind `control`) → `DENIED_CONTROL` unless `intent.id` = `revoke` and the payload is
  exactly the strict-JSON Message `{"Case":<case>,"Generation":<gen>,"ID":"revoke","Kind":"revoke","Candidate":""}`.
- **inbox** (resource kind `inbox`) → `DENIED_MESSAGE` unless the payload is a strict-JSON Message (§7.6)
  with `Case` = case id, `Generation` = S.generation, `ID` matching `^[a-z0-9-]{1,48}$` and equal to
  `intent.id`, and the kind rules of §7.6 hold.

If step 9 passes, the remaining checks are stateful (step 10). The vectors' "accept" codes (`INVITE_OK`,
`CONTROL_OK`, `ACCEPTED`) are those the receiver returns for a case that is in the right state (for inbox
messages: active, with no earlier messages).

**Derived from:** dataexchange v0.2.2 `governed.go` `DecisionFrameVerifier.VerifyGovernedFrame`,
`enforceFrameConstraints`; `governed_replay.go` `governedReplayGuard` (service path, not used); common v0.5.13 `decision/enforcer.go` `TrustStore`, `Enforcer.Verify` (and
`verify`); `decision/decision.go` `Decision.VerifyFor`, `Intent.Verify`, `Decision.Verify`;
`decision/mandate.go` `Mandate.Verify`. Client: `wire.go` `accept`, `keyStore`, `ceiling`; `scope.go`
`verifyGrant`, `strictJSON`.

---

## 7. Agent Network profile

This section fixes every value a sender chooses, so that two implementations given the same keys, clock and
nonce produce byte-identical frames.

### 7.1 Constants

| Name | Value |
|---|---|
| Tenant (`tenant_id` everywhere) | `anv-client-m1` |
| Action | `data.send.text` (inner type 1) |
| Risk | `medium` |
| Outcome | `allow` |
| Proof lifetime | `issued_at` = sender's now; `expires_at` = min(now + 60, S.expires_at) |
| Nonce | 16 bytes from a CSPRNG, lowercase hex (32 characters), fresh per proof |
| Frame cap / envelope limit | 65536 / 16384 bytes |
| Port | Pilot port 1001 |

Party names (`self`, `peer`) are the enrolled party identifiers. To be usable they MUST satisfy both the
identifier rule (§4.1) and the resource pattern `[A-Za-z0-9-]{1,32}`.

### 7.2 Scope hash and purpose

A case is described by a *scope* that both owners approved. The scope hash is the lowercase hex SHA-256 of
the scope's JSON encoding, produced with the Go JSON encoding (§8.2) and these members in this order:

| Member | Type |
|---|---|
| `schema_version` | integer (1) |
| `case_id` | string, `^[a-z0-9-]{8,48}$` |
| `generation` | integer (1) |
| `template_id` | `finite-choice-v1` |
| `purpose_code` | `schedule` |
| `initiator` | party name |
| `participants` | array of 2 objects sorted by `party` (bytewise), each with members `party`, `addr`, `transport_key`, `intent_key`, `authority_key` in that order (keys as lowercase hex) |
| `catalog` | object, members sorted by key (bytewise) |
| `disclosure_by_party` | object, members sorted by party (bytewise); each value an array of catalog ids sorted bytewise |
| `actions` | `["offer","result","confirm_result"]` |
| `max_messages_per_side` | 32 |
| `max_payload` | 16384 |
| `expires_at` | integer |
| `policy_revision` | integer (1) |

The **scope purpose** is `schedule:` followed by the scope hash (73 characters). The fixture
(`../vectors/fixture.json`) gives an example scope, its exact JSON and hash. Note that free-text catalog values
are subject to the Go escaping rules of §8.2 (`<`, `>`, `&` become `<`, `>`, `&`).

### 7.3 Resources and identifiers

`resource(kind, to)` = `agent:<to>/<kind>/<case_id>/g<generation>`, with `kind` one of `invite`, `inbox`,
`control`. The receiver accepts only resources matching
`^agent:([A-Za-z0-9-]{1,32})/(inbox|invite|control)/([a-z0-9-]{8,48})/g1$` whose first group is its own
party name (the profile supports generation 1 only).

| Field | Value (sender `self`, receiver `peer`) |
|---|---|
| Intent `id` | `invite` for invites, `revoke` for control, the Message `ID` for inbox messages |
| Intent `agent_id` | `self` |
| Intent `resource` | `resource(kind, peer)` |
| Intent `mandate_id` | `<case_id>-<self>` |
| Intent `audience` | `agent:<peer>` |
| Intent `purpose` | scope purpose |
| Intent `key_id` | `<self>-intent` (signed with the intent key) |
| Decision `id` | `decision-<intent id>` |
| Decision `agent_id` | `self` |
| Decision `policy_revision` | S.policy_revision (1) |
| Decision `revocation_epoch` | S.generation (1) |
| Decision `provider_id`, `key_id` | `<self>-authority` (signed with the authority key) |
| Decision `issued_at`, `expires_at` | same as the Intent |
| Decision `reasons`, `constraints` | absent |

Because `mandate_id`, `audience` and `purpose` are always set, every profile Intent uses the delegated
canonical domain.

### 7.4 Envelope

`version` 1, `type` 1, no `filename`, no `disclosure`, `payload` = base64 of the inner JSON (§7.5, §7.6).

### 7.5 Grant (Mandate) — invite payload

Each owner's approval of a scope is a Mandate signed with that party's authority key:

| Field | Value (grantor `self`, grantee `peer`) |
|---|---|
| `version` | 1 |
| `id` | `<case_id>-<self>` |
| `tenant_id` | `anv-client-m1` |
| `subject_agent_id` | `self` |
| `actions` | [`data.send.text`] |
| `resource_prefixes` | [`agent:<peer>/inbox/<case_id>/g<generation>`] |
| `audience` | `agent:<peer>` |
| `purpose` | scope purpose |
| `constraints`, `required_approvals` | absent |
| `revocation_epoch` | S.generation |
| `issued_at` | grantor's clock at approval |
| `expires_at` | S.expires_at |
| `key_id` | `<self>-authority` |

The invite's inner payload is the Go JSON encoding of this Mandate (§4.4.1 member order). The Mandate is
**not** evaluated against Intents by Pilot in this profile: the receiver only checks it on invites (§6.3) and
uses the pair of grants as local case state. In particular, its `resource_prefixes` names only the inbox, yet
invites and control messages use other resources.

### 7.6 Business message — inbox and control payload

Go JSON encoding of an object with members, in this order: `Case` (string), `Generation` (integer), `ID`
(string), `Kind` (string), `Candidate` (string, always present, may be `""`), `Digest` (string, **omitted
when empty**). Kind rules checked by the receiver (step 9):

| Kind | Sender | Rules |
|---|---|---|
| `offer` | initiator | `Digest` absent; `Candidate` is in the sender's disclosure list |
| `result` | non-initiator | `Candidate` in both parties' disclosure lists; `Digest` = result digest |
| `confirm_result` | initiator | `Digest` = result digest |
| `revoke` | either (control resource) | exact form of §6.3 |

Result digest = lowercase hex SHA-256 of the ASCII string
`anv-result-v1|<case_id>|<generation>|<scope hash>|<candidate>` (generation in decimal).

### 7.7 Acknowledgement

The receiver answers with one plain **text frame (type 1)** whose payload is an ASCII code, then waits
(bounded) for the sender to close. Codes produced by the stateless steps are listed in §6.1; the full set and
their handling are in `CASE_PROTOCOL.md`. When the receiver is saturated it sends `BUSY` without reading the
request. The reference sender interprets the acknowledgement payload as the code **without checking the
frame type**.

**Derived from:** Client `scope.go` (`Tenant`, `proofTTL`, `Scope`, `Scope.Hash`, `purpose`, `resource`,
`resourcePattern`, `resultDigest`, `grant`, `Proof`, `Message`), `wire.go` (`receive`, `deliver`),
`service.go` (`FrameCap`, `MaxPayload`, `NewScope`); common v0.5.13 `decision.NewNonce`.

---

## 8. Interoperability notes

### 8.1 What signatures do and do not cover

- Signatures cover canonical bytes only. JSON member order, whitespace and escaping of the envelope,
  Intent and Decision are not signed. The **inner payload** bytes are covered (through `payload_hash`), and
  the profile additionally requires them to be strict JSON.
- The Intent signature is not covered by the Decision's `intent_hash`.
- The payload hash covers the inner type and filename, so a proof for text cannot be replayed as another
  type.

### 8.2 Go JSON encoding (what senders MUST produce)

Profile senders produce, and strict-JSON checks (§6.3) require, exactly the output of Go's `encoding/json`
`Marshal`:

- Members in the documented order; no whitespace anywhere; omitted-when-empty members as documented.
- Integers in plain decimal (no `+`, no leading zeros, no fraction, no exponent; `-` for negatives).
- Byte strings (`payload`) as base64 with padding; an empty byte string as `null`.
- Strings: `"` → `\"`, `\` → `\\`; U+000A, U+000D, U+0009, U+0008, U+000C → `\n`, `\r`, `\t`, `\b`, `\f`; other
  characters below U+0020 → `\u00XX` with lowercase hex; `<` `>` `&` → `\u003c` `\u003e` `\u0026`; U+2028 and
  U+2029 → `\u2028` `\u2029`; invalid UTF-8 bytes → `\ufffd`; U+007F and all other non-ASCII characters
  are written as raw UTF-8; `/` is not escaped.
- Object members of map-typed values (scope `catalog`, `disclosure_by_party`) sorted by key, bytewise.

In the profile, the signed text fields contain only ASCII letters, digits and `-:/._`, so the escaping rules
matter only for scope catalog values (scope hash) and for non-profile objects.

### 8.3 Reference decoder leniency (receivers MAY be stricter)

The reference receiver decodes the envelope, Intent and Decision with Go's `encoding/json` (v1), which:
matches member names case-insensitively; lets the last of duplicate members win; accepts `null` for any
member (leaving it empty/zero — so `"disclosure": null` and an absent `disclosure` are the same, and
`"payload": null` is an empty payload); ignores `\r` and `\n` inside the base64 `payload` string and does
not require the unused low bits of the final base64 character to be zero; replaces invalid UTF-8 and
unpaired surrogate escapes in strings with U+FFFD before validation; rejects integers that are out of range for
the field type or written with a fraction or exponent. Hashes and signatures are computed over the decoded
values, so these variations cannot change what is authorised. Vector `governed-offer-envelope-variant`
(sorted members, indentation, `"disclosure": null`) is accepted by the reference; a stricter receiver MAY
reject it. Conforming senders MUST NOT depend on any of this.

### 8.4 Base64 signatures

Signatures MUST be written as 88-character padded standard base64. The reference verifier decodes with the
padded standard alphabet (unpadded or URL-safe input is rejected — vector `intent-signature-bad-base64`)
and, like 8.3, does not require zero unused bits, so more than one string can denote the same signature.
Never compare signatures as strings to detect duplicates.

### 8.5 Integers and time

- `issued_at`/`expires_at` are signed 64-bit; `policy_revision`/`revocation_epoch` unsigned 64-bit;
  `version` and `required_approvals` unsigned 16-bit; inner `type` unsigned 32-bit. JSON numbers outside
  these ranges are rejected by the decoder. Implementations in languages whose JSON numbers are IEEE
  doubles MUST parse these members as exact integers (values above 2^53 occur only in non-profile
  objects).
- Freshness boundaries are inclusive in the accepting direction: accepted at `now = expires_at`
  (`governed-offer-at-expiry`) and at `now = issued_at − 60` (`governed-offer-clock-skew`).
- The clock used for freshness is the receiver's wall clock in whole seconds.

### 8.6 Text and Unicode

- Canonical strings are raw UTF-8 bytes with a byte-length prefix; there is no Unicode normalisation, so
  visually equal strings in different normal forms hash differently.
- Length limits are in bytes, not characters.
- "Control character" means only U+0000–U+001F and U+007F.
- Sorting (Mandate actions/prefixes, constraints, scope maps, participants) is bytewise on UTF-8, which equals
  code-point order.

### 8.7 Other subtleties

- The canonical domain of an Intent changes when any of `mandate_id`/`audience`/`purpose` is set, and then
  all three are written; an Intent without them cannot be re-labelled as delegated.
- `payload_hash` is SHA-256 applied twice (§5), unlike every other hash here.
- The envelope limit (16384) is a profile rule on the outer frame; Pilot's own checks use only the frame cap.
- A decoded type-4 inner frame or a disclosure object is rejected by the profile even when Pilot considers it
  valid (`inner-type-file`, `disclosure-present`).
- The profile receiver keeps no nonce cache; a captured frame can be replayed until it expires (≤ 60 s in the
  profile). Idempotence comes from the business message id (`CASE_PROTOCOL.md`).

---

## 9. Rule identifiers (index)

Rule ids appear in the vectors' `expect.rule`.

| Rule | Section | Code |
|---|---|---|
| F-CAP, F-SHORT | 2.3, 2.4 | DENIED_FRAME |
| P-ENVELOPE-SIZE, P-OUTER-TYPE | 2.3, 6.1 | DENIED_FRAME |
| G-JSON, G-VERSION-TYPE, G-FILENAME, G-INTENT, G-UNSIGNED, G-ACTION, G-PAYLOAD-HASH, G-DECISION | 3.2, 3.3 | DENIED_FRAME |
| P-INNER | 6.1 | DENIED_FRAME |
| P-RESOURCE | 7.3 | DENIED_RESOURCE |
| P-CASE | 6.1 | DENIED_UNKNOWN_CASE |
| V-RESOURCE, V-INTENT-KEY, V-INTENT-FRESH, V-INTENT-SIG, V-DECISION-KEY, V-DECISION-FRESH, V-DECISION-SIG, V-BIND-HASH, V-BIND-TENANT-AGENT, V-BIND-EXPIRY, V-BIND-ISSUED, V-STATE, V-CEILING, V-OUTCOME | 6.2 | DENIED_PROOF |
| M-JSON, M-SIG, M-FRESH, M-PROFILE | 6.3 | DENIED_GRANT |
| B-MESSAGE | 6.3, 7.6 | DENIED_MESSAGE |
| (control) | 6.3 | DENIED_CONTROL |
