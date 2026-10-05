# Wire-format test vectors

Test vectors for `../spec/WIRE_FORMAT.md`. They are generated deterministically (fixed Ed25519 seeds, fixed
nonces, fixed clock) by `pan-bridge/cmd/vectors` linking `github.com/pilot-protocol/common v0.5.13` and
`github.com/pilot-protocol/dataexchange v0.2.2`, and re-verified with that code by
`go test ./cmd/vectors` in `pan-bridge`. All keys, parties and data are synthetic. Licence: CC BY 4.0.

Do not edit these files by hand; regenerate them (see the end of this file).

## Files

| File | Content |
|---|---|
| `fixture.json` | Shared inputs: parties and keys, the case scope, constants. |
| `objects.json` | Signed objects (Mandate, Intent, Decision) and payload hashes. |
| `frames.json` | Plain frames (acknowledgements) and governed frames that MUST be accepted. |
| `negative.json` | Governed frames (or byte streams) that MUST be rejected, with the expected code. |

All byte strings are lowercase hex unless the field is documented as a JSON string. Timestamps are Unix
seconds.

## `fixture.json`

| Field | Meaning |
|---|---|
| `tenant` | `anv-client-m1` |
| `t0` | the vectors' reference time (1893456000 = 2030-01-01T00:00:00Z); every profile proof is issued at `t0` |
| `frame_cap`, `max_envelope`, `proof_ttl` | 65536, 16384, 60 (spec §2.3, §7.1) |
| `parties[]` | `party`, `addr`, `transport_key` (placeholder), `intent_key`, `authority_key` (hex Ed25519 public keys), `intent_seed`, `authority_seed` (hex 32-byte RFC 8032 private seeds). `alice` is the case initiator, `bob` the other participant; `carol` is not in the case and only supplies wrong keys. |
| `scope_json` | the exact scope JSON (spec §7.2) as a JSON string |
| `scope_hash`, `purpose` | SHA-256 hex of `scope_json`; `schedule:<scope_hash>` |
| `mandate_issued_at` | `issued_at` of both grants |

## `objects.json`

`mandates[]`, `intents[]`, `decisions[]` — each entry:

| Field | Meaning |
|---|---|
| `id`, `description` | name and purpose |
| `kind` | `mandate`, `intent` or `decision` |
| `signer_public` | hex public key that verifies `signature` |
| `json` | the object's JSON exactly as a profile sender encodes it (a JSON string; decode it once to get the bytes) |
| `canonical_hex` | canonical encoding (spec §4) |
| `hash` | lowercase hex SHA-256 of the canonical bytes |
| `signature` | base64 Ed25519 signature over the canonical bytes (same as in `json`) |

To check an implementation: parse `json`; produce the canonical bytes and compare with `canonical_hex`;
hash and compare with `hash`; verify `signature` with `signer_public`; re-encode the object and compare with
`json` (for the profile objects this must be byte-identical). Every object is valid at `t0 + 5`.

`payload_hashes[]`: `frame_type`, `filename`, `payload_hex`, and the expected `hash` (spec §5).

## `frames.json`

`plain[]`: `type`, `payload` (text), and `frame_hex` (header + payload, spec §2). The `ack-*` entries are the
acknowledgements of spec §7.7.

`governed[]` (all MUST be accepted) and `negative.json` (a list; all MUST be rejected) share one schema:

| Field | Meaning |
|---|---|
| `id`, `description` | name and what is special about it |
| `sender`, `receiver` | party names. The receiver is `receiver`; the connection's authenticated peer is `sender` (spec §6.1 step 0 is assumed to have passed). |
| `verify_at` | the receiver's clock (Unix seconds) to use for all freshness checks |
| `frame_hex` | the complete bytes the sender writes on the stream (8-byte header + payload). For some negative vectors these are deliberately short or have a wrong header. |
| `envelope_json` | (accepted vectors only) the outer payload as text |
| `inner_payload` | (accepted vectors only) the decoded inner payload as text |
| `intent_hash` | (accepted vectors only) Hash of the Intent; equals `decision.intent_hash` |
| `expect.accept` | `true` in `frames.json`, `false` in `negative.json` |
| `expect.code` | the acknowledgement code the receiver sends (spec §6.1) |
| `expect.stage` | the receive step that decides: `read_frame`, `profile_frame`, `decode_governed`, `profile_inner`, `profile_resource`, `verify_governed`, `profile_payload`, or `pass` |
| `expect.rule` | the rule id of spec §9 (`accept` / `accept-lenient` for accepted vectors) |

Receiver state assumed by every vector: the receiver knows the case of `fixture.scope_json`, with the
`sender` as its peer and the parties' keys pinned as in the fixture; for inbox messages the case is active and
has no earlier messages. Stateful checks after `profile_payload` (spec §6.1 step 10) are not covered.

A conforming receiver MUST return `expect.code` for every vector. `expect.stage` and `expect.rule` say why;
an implementation whose internal check order differs may report a different rule as long as the code is the
same. `governed-offer-envelope-variant` (`accept-lenient`) is accepted by the reference receiver; a stricter
receiver MAY reject it with `DENIED_FRAME` (spec §8.3).

## Regenerating and checking

With a checkout of `d25-dev/pan-bridge` next to this repository and Go 1.25:

```
cd pan-bridge
go run ./cmd/vectors -out ../pan-protocol/vectors
go test -count=1 -v ./cmd/vectors
```

The generator refuses to write if any vector fails its own check. The test regenerates the vectors, checks
that generation is deterministic and identical to these files, then parses these files and re-verifies each
vector from its serialized fields with the Pilot code.

## Profile v2 (`v2.json`)

Vectors for `spec/PROFILE_V2_DELEGATION.md`. Parties and authority keys are those of `fixture.json`.

| member | content |
|---|---|
| `fixture` | the v2 scope (`schema_version` 2) and its hash, the constraint key, the delegate key seeds per party, and the seed of a delegate key of another case |
| `grants` | v2 grants (Mandates with one `pan.delegate.ed25519` constraint), as object vectors (canonical bytes, hash, signature) |
| `frames` | frame vectors as in `frames.json` / `negative.json`, plus `stored_grant`: the id of the peer grant the receiver has already accepted for the case (absent: none) |

A conforming v2 receiver MUST return `expect.code` for every frame, given the stated stored grant.
