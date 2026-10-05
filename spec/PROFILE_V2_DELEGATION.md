# Profile v2 — Delegated case keys

Status: draft (2026-10-05). Licence: CC BY 4.0. Extends `CASE_PROTOCOL.md` and `WIRE_FORMAT.md` (profile v1);
everything not stated here is unchanged.

## 1. Purpose

In v1 every business message is signed with the party's **authority key**, so the process that sends
messages automatically (the Client and the Agent connected to it) must hold that key at all times. Anyone who
takes over that process can approve new cases as the owner.

v2 splits the keys, on **one device** (no second device is required):

| key | held where | used for | owner presence |
|---|---|---|---|
| **authority key** (Ed25519) | the device's protected key store (e.g. macOS Keychain); never exported | signing grants (approving a case) | REQUIRED for every signature on devices that support it (Touch ID / password) |
| **delegate key** (Ed25519, one per case per party) | the Client's normal storage | signing every Intent and Decision of that case | not required: automation runs with it |

A party approves a case by signing a grant that **names its delegate key**. A peer checks the grant with the
pinned authority key and then accepts messages of that case signed by the named delegate key — and nothing else.
A stolen delegate key can only act within a case the owner already approved, until that case ends; a stolen
Client process cannot approve anything new without the owner's presence.

## 2. Selecting the profile

- A scope with `schema_version` **2** uses this profile; `schema_version` 1 keeps v1. A Client that does not
  implement v2 MUST reject a v2 scope when normalizing it (CASE_PROTOCOL §2), so the owner never approves a case
  the peer's Client cannot verify.
- Participant records (CASE_PROTOCOL §1) are unchanged; in v2, `intent_key` is not used (it MAY be empty).

## 3. Grant (Mandate) in v2

As profile v1 (WIRE_FORMAT §7.5), signed with the **authority key**, with exactly one constraint:

| key | operator | value |
|---|---|---|
| `pan.delegate.ed25519` | `eq` | the grantor's delegate public key for this case, 64 lowercase hex characters |

- The grant MUST contain exactly this one constraint; any other constraint, a second delegate entry, or a value
  that is not 32 bytes of lowercase hex makes the grant invalid.
- A grant is valid only if it verifies under the grantor's pinned **authority key** and every other field
  equals the v1 profile value for this scope (CASE_PROTOCOL §3).

## 4. Proofs in v2

Every Intent and Decision of a v2 case (invite, business messages, revoke) is signed with the sender's
**delegate key** for that case. Field values differ from v1 as follows:

| field | v1 | v2 |
|---|---|---|
| `intent.key_id` | `<self>-intent` | `<self>-case-<case_id>` |
| `decision.key_id` | `<self>-authority` | `<self>-case-<case_id>` |
| `decision.provider_id` | `<self>-authority` | `<self>-case-<case_id>` |

All other fields, the envelope, the payloads and the canonical encodings are as in v1.

## 5. Receiver key store in v2

For a v2 case, the key store (WIRE_FORMAT §6.2 steps 3 and 5) returns:
- for intent key id `<peer>-case-<case_id>` and decision key id `<peer>-case-<case_id>`: the delegate key named
  by the **peer's verified grant** for this case;
- for anything else: nothing (`DENIED_PROOF`).

The ceiling (WIRE_FORMAT §6.2 step 8) checks `decision.provider_id` = `<peer>-case-<case_id>` instead of
`<peer>-authority`; all other ceiling conditions are unchanged.

Where the peer's grant comes from:
- **Invite** (resource kind `invite`): the grant is the invite's payload. The receiver MUST first decode the
  payload strictly and verify it as a grant of (peer → self, this scope) under the peer's pinned authority key
  (`DENIED_GRANT`), and only then verify the frame's proofs with the delegate key it names (`DENIED_PROOF`).
  This is the one change to the receive order (CASE_PROTOCOL §6 step 5 and 6 swap for invites only). The
  payload is still not stored before every check passed.
- **Business messages and revoke**: the peer's grant stored when its invite was accepted. If there is none,
  the proofs cannot be verified (`DENIED_PROOF`); a sender keeps retrying until its invite was accepted
  (unchanged retry rules).

A new grant from the same peer for the same case (a re-sent invite) MUST name the same delegate key; a
different key is rejected (`DENIED_GRANT`). Changing a delegate key mid-case is not supported in v2: the owner
revokes the case and approves a new one.

## 6. Key lifecycle

- **Authority key**: created once per party. Signing MUST require owner presence where the device supports it
  (macOS: Keychain item with a user-presence access control; Touch ID or the login password). It is used only
  to sign grants. Replacing it follows the existing key-change review (`KEY_CHANGE_REQUIRES_REVIEW`).
- **Delegate key**: generated when the owner approves the case, in the same step as the grant. The private
  key is stored with the case and never reused across cases. It is deleted:
  - at once when the case is closed, revoked by the peer, or expires;
  - when the owner revokes the case: as soon as the revoke message (which is signed with this key, §4) is no
    longer pending — delivered, rejected or cancelled — or the case expires, whichever comes first.
- **Approval is one owner action**: one presence check signs the grant; nothing else in the case needs the
  owner again until the result.

## 7. Security notes

- Compromise of the Client process or an Agent: the attacker can send, within already-approved cases, what the
  case scope allows until expiry; it cannot approve new cases, enroll contacts, or change keys.
- Compromise of a delegate key alone: limited to that case and that scope.
- Compromise of the authority key: as in v1 (full control of new approvals); hence the presence requirement.
- Receivers keep the v1 duplicate and limit rules; nothing in v2 widens what a case permits.

## 8. Vectors

`vectors/` will contain v2 vectors: a v2 grant with a delegate constraint, invite/offer/revoke frames signed
with delegate keys, and negative cases (grant without or with two delegate constraints, proof signed by the
authority key instead of the delegate key, delegate key of another case, re-sent invite naming another key,
provider id mismatch).
