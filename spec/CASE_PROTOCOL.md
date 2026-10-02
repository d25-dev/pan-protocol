# Case Protocol v1

Status: draft (2026-10-02), describing the behaviour of the reference Client.
Licence: CC BY 4.0. Byte formats, canonical encodings and signature rules are in `WIRE_FORMAT.md`; this document
uses its terms (frame, governed frame, Intent, Decision, Mandate).

A **case** is one bounded piece of coordination between exactly two parties (person↔person, person↔shop,
shop↔shop are treated alike). Both owners approve the same immutable **scope**; only then may business messages
flow, and only those the scope allows. Version 1 has one template: choosing one item from a finite catalogue
(`finite-choice-v1`, purpose `schedule`).

Keywords MUST, MUST NOT, SHOULD are used as in RFC 2119.

## 1. Parties and identities

A party is identified by a **participant record**:

| field | meaning |
|---|---|
| `party` | party name, `[A-Za-z0-9-]{1,32}`, unique within a case |
| `addr` | the party's Pilot address (text form) |
| `transport_key` | hex Ed25519 key of the party's Pilot node (as held by the registry) |
| `intent_key` | hex Ed25519 public key that signs the party's Intents |
| `authority_key` | hex Ed25519 public key that signs the party's Mandates and Decisions |

- Records are exchanged out of band and **enrolled** (pinned) by each owner.
- A Client MUST NOT accept a record that changes the keys of an already-enrolled party without explicit owner
  review (`KEY_CHANGE_REQUIRES_REVIEW`).
- After enrollment, a Client asks its Pilot daemon for a trust handshake with the contact, with reason
  `PAN_CONNECT_V1`. Relationship trust alone never permits business messages.

Both owners obtain the **same scope** before approving it: the initiator's owner creates it and shares it with the
peer's owner out of band (v1 has no in-protocol scope transfer); each Client normalizes and checks it (§2) and
approves it only if it pins exactly the two enrolled parties.

## 2. Scope

The scope is a JSON object; its encoding is the **canonical JSON** of §8, and its hash is
`hex(SHA-256(canonical JSON))`.

| field | type / rule (v1) |
|---|---|
| `schema_version` | `1` |
| `case_id` | `[a-z0-9-]{8,48}` |
| `generation` | `1` |
| `template_id` | `"finite-choice-v1"` |
| `purpose_code` | `"schedule"` |
| `initiator` | one of the two parties |
| `participants` | exactly the two enrolled participant records, sorted by `party` |
| `catalog` | object, 1..16 entries; key `[a-z0-9-]{1,16}`, value a string of 1..64 bytes |
| `disclosure_by_party` | object with exactly the two parties as keys; each value a non-empty, sorted, duplicate-free list of catalog keys that party may reveal |
| `actions` | `["offer","result","confirm_result"]` |
| `max_messages_per_side` | `32` |
| `max_payload` | `16384` |
| `expires_at` | Unix seconds, in the future and at most 7 days ahead when created |
| `policy_revision` | `1` |

- **purpose** = `"schedule:" + scope hash`.
- **resource**(`kind`, `to`) = `"agent:" + to + "/" + kind + "/" + case_id + "/g" + generation`, with `kind` in
  `invite` / `inbox` / `control`. A receiver MUST reject a resource that does not match
  `^agent:([A-Za-z0-9-]{1,32})/(inbox|invite|control)/([a-z0-9-]{8,48})/g1$` or whose first group is not itself.
- **result digest**(candidate) =
  `hex(SHA-256("anv-result-v1|" + case_id + "|" + generation + "|" + scope hash + "|" + candidate))`.

## 3. Approval (grant)

Each owner approves the scope by signing a **Mandate** with their authority key (field values in
`WIRE_FORMAT.md` §Profile):
- id `<case_id>-<self>`, subject `<self>`, audience `agent:<peer>`;
- actions `["data.send.text"]`, resource prefixes `[resource("inbox", peer)]`;
- purpose = scope purpose, revocation epoch = generation, expires_at = scope `expires_at`;
- key id `<self>-authority`, no constraints, no required approvals.

A grant is valid only if it verifies under the granter's pinned authority key **and** every field equals the
value above for this scope.

**Authority state** of a case, computed from the current clock at every enforcement point (never from a timer),
evaluated in this order:

| state | condition |
|---|---|
| REVOKED / CLOSED / PEER_REVOKED | the case was stopped (locally, or by the peer's revoke) |
| EXPIRED | now ≥ `expires_at` |
| DRAFT | not yet approved locally, or the local grant does not verify |
| WAITING_PEER | the peer's grant is missing or does not verify |
| ACTIVE | otherwise |

Business messages may be sent and accepted only while ACTIVE.

## 4. Messages

Every message is one governed frame (WIRE_FORMAT) on its own Pilot stream to port 1001. The Intent's `id` is the
message id, and its `resource` is resource(kind, receiver).

| message | resource kind | Intent id | payload (canonical JSON, §8) | sent when |
|---|---|---|---|---|
| invite | `invite` | `invite` | the sender's grant (Mandate JSON) | the sender approved; repeated until the peer acknowledges or the case expires |
| offer | `inbox` | `[a-z0-9-]{1,48}` | `{"Case","Generation","ID","Kind":"offer","Candidate"}` | the initiator proposes a candidate it may disclose |
| result | `inbox` | same | `{"Case","Generation","ID","Kind":"result","Candidate","Digest"}` | the non-initiator picks a candidate both may disclose |
| confirm_result | `inbox` | `confirm-1` | `{"Case","Generation","ID","Kind":"confirm_result","Candidate","Digest"}` | the initiator confirms the received result |
| revoke | `control` | `revoke` | `{"Case","Generation","ID":"revoke","Kind":"revoke","Candidate":""}` | an approved case is revoked locally |

- The payload's `Case`, `Generation` and `ID` MUST equal the case, its generation and the Intent id.
- **Free text is not representable.** Candidates are catalog keys.
- Receiver rules per kind:
  - offer: the sender is the initiator, `Digest` is empty, and the scope lets the sender disclose `Candidate`.
  - result: the sender is not the initiator, both parties may disclose `Candidate`, `Digest` = result digest, and
    no result exists yet.
  - confirm_result: the sender is the initiator, `Digest` = result digest, and it matches the stored result that
    this Client proposed.

## 5. Sending

Outgoing messages are stored first (outbox), then delivered by a sender loop. Per attempt:

1. Recompute the authority state:
   - invite: only while approved and not expired;
   - revoke: only while revoked and not expired;
   - other messages: only while ACTIVE.
   If not eligible, the message is cancelled, or kept waiting while DRAFT / WAITING_PEER.
2. Look up the peer's transport key in the registry. It MUST equal the pinned `transport_key`.
3. Build fresh proofs (Intent and Decision, valid for at most 60 s, never beyond the case expiry). The message id
   and payload stay the same across retries.
4. Open a stream (5 s timeout). Immediately before submitting the frame, recompute the step-1 condition again (the
   lock excludes every state change, but not the clock: the case may have expired) and cancel if it no longer
   holds. Then write the frame (8 s deadline). The local state lock is held from step 1 until the frame is submitted
   to the transport, so a revoke cannot overtake a message already being written; submitted bytes may still arrive
   late (BRIDGE_API §4). The lock is released before waiting for the acknowledgement.
5. Read one acknowledgement frame: a plain text frame whose payload is an acknowledgement code (§7). A frame of any
   other type is treated like a missing acknowledgement (`DELIVERY_UNKNOWN`).
6. Before adopting success for a business message (offer, result, confirm_result), recompute the authority state:
   an acknowledgement that arrives after local termination does not count as progress. Invite and revoke are
   exempt — their acknowledgement only records that the peer has the grant or the revocation.

Delivery states: `QUEUED` → `PEER_ACCEPTED` | `REJECTED` | `CANCELLED`, with `DELIVERY_UNKNOWN` when the write may
have succeeded but no acknowledgement was read.
- Retries back off 1, 2, 4, 8, 16, then every 30 s.
- A `DELIVERY_UNKNOWN` message is retried with the same id and payload. The receiver's duplicate rule (§6) makes
  this safe.

## 6. Receiving

A receiver MUST apply these checks **in this order**, and MUST NOT store anything before the final step:

1. **Admission.** At most 4 messages are processed at once; otherwise answer `BUSY`.
2. **Actual peer.** The stream's remote address (from the transport, not from the message) MUST belong to an
   enrolled contact. The registry's current key for it MUST equal the pinned `transport_key`. The daemon MUST
   trust that peer, and if the daemon reports a key it MUST equal the pinned one. → `DENIED_PEER`. No byte of the
   message is consumed before this check.
3. **Frame.** Read one frame within the frame cap. Reject on error or a short read. The outer frame MUST be
   governed, its payload (the whole governed envelope, WIRE_FORMAT §7.4) at most 16384 bytes — about 11.4 KB is left for the
   inner payload — the inner type text, no filename, no disclosure. → `DENIED_FRAME`.
4. **Resource.** The resource matches §2 and names this party. → `DENIED_RESOURCE`. The case MUST exist locally
   (`DENIED_UNKNOWN_CASE`), and the sender MUST be its peer (`DENIED_PEER`).
5. **Proof.** Verify Intent and Decision (WIRE_FORMAT §Verification) under the peer's pinned intent and authority
   keys, with the case-binding ceiling: action `data.send.text`, agent = peer, audience `agent:<self>`, purpose,
   mandate id `<case_id>-<peer>`, expiries ≤ case expiry, provider `<peer>-authority`, revocation epoch =
   generation, policy revision = scope's, no reasons, no constraints. → `DENIED_PROOF`.
6. **Kind rules.**
   - invite: Intent id `invite`; the payload is a grant that is valid for (peer → self, this scope)
     → `DENIED_GRANT`; the case is not terminal → `DENIED_TERMINAL`.
   - revoke: exact payload → `DENIED_CONTROL`.
   - Otherwise: payload decoding per §8 and §4 → `DENIED_MESSAGE`.
7. **Transactional recheck**, in one transaction with the commit:
   - the case is ACTIVE; or, if it is still DRAFT / WAITING_PEER and not expired, answer `NOT_YET_ACTIVE`
     (nothing stored, the sender retries); otherwise `DENIED_NOT_ACTIVE`;
   - duplicate: same message id with the same content hash → `DUPLICATE` (not counted again); same id with
     different content → `DENIED_ID_CONFLICT`;
   - at most 32 received messages per case → `DENIED_LIMIT`;
   - result rules → `DENIED_RESULT_EXISTS` / `DENIED_DIGEST`.
8. **Commit, then acknowledge.**
   - The stored value is the verified, re-encoded payload, never the received bytes. §8 guarantees they are equal.
   - Write the acknowledgement frame.
   - Then wait up to 3 s for the sender to close the stream before closing. Closing immediately after writing lost
     acknowledgements on relayed paths.

An invite never activates a case by itself: local approval is still required.

## 7. Acknowledgement codes

| code | meaning | sender action |
|---|---|---|
| `ACCEPTED`, `DUPLICATE`, `INVITE_OK`, `CONTROL_OK` | stored (or already stored) | mark `PEER_ACCEPTED` (subject to §5 step 6) |
| `BUSY`, `UNAVAILABLE`, `NOT_YET_ACTIVE` | temporary | keep `QUEUED`, retry with backoff |
| any `DENIED_*` | rejected | invite: keep retrying until expiry (the peer may not have created or approved the case yet); others: `REJECTED` |

## 8. Canonical JSON

Business payloads and the scope are accepted only if their bytes are **exactly** the canonical encoding of the
decoded value:
- object fields in the order defined above;
- map keys sorted;
- no insignificant whitespace;
- no unknown or duplicate fields;
- standard string escaping.

The check is: decode strictly, re-encode, compare bytes. This makes what is verified identical to what is stored.

The byte-level rules (Go JSON encoding: HTML-safe escaping of `<`, `>`, `&`, number and string forms) are in
WIRE_FORMAT §8.2; receivers MAY be stricter than the reference decoder (§8.3).

## 9. Limits (v1)

| limit | value |
|---|---|
| frame | 65536 bytes |
| governed envelope (outer frame payload) | 16384 bytes |
| messages per side | 32 |
| concurrent receives | 4 |
| proof lifetime | 60 s |
| case lifetime at creation | 7 days at most |
| catalog | 16 entries |
