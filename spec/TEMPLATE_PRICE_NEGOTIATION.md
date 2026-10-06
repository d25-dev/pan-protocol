# Template `price-negotiation-v1`

Status: draft (2026-10-06). Licence: CC BY 4.0. Extends `CASE_PROTOCOL.md`; everything not stated here is as there,
and it works with both profile v1 and profile v2 (`PROFILE_V2_DELEGATION.md`).

One party sells one item, the other buys it. Within a public price range both parties' Clients exchange price bids
until one accepts the other's last bid, one declines, or the rounds run out. **Each owner's own limit (the lowest
price a seller accepts, the highest price a buyer pays) never leaves that owner's Client**, and the Client refuses to
send any bid or acceptance beyond it — whatever the Agent asks for. No payment is made by the protocol.

## 1. Scope

As CASE_PROTOCOL §2, with these differences:

| field | value |
|---|---|
| `template_id` | `"price-negotiation-v1"` |
| `purpose_code` | `"negotiate"` |
| `catalog` | exactly one entry: the item id → its label (1..64 bytes) |
| `disclosure_by_party` | both parties → `[<item id>]` |
| `actions` | `["bid","accept","decline","confirm_result"]` |
| `price` | the price terms below (absent in other templates) |

`price` object (members in this order):

| member | type | rule |
|---|---|---|
| `currency` | string | `"JPY"` in v1 |
| `min` | integer | ≥ 1 |
| `max` | integer | > `min`, ≤ 1 000 000 000 |
| `step` | integer | ≥ 1; `min` and `max` are multiples of it; (`max` − `min`) / `step` ≤ 100 000 |
| `max_rounds` | integer | 1..16 — bids per party |
| `seller` | party name | one of the two participants |
| `buyer` | party name | the other participant |

- `initiator` makes the first bid; it may be the seller or the buyer.
- The purpose is `"negotiate:" + scope hash`. The resource, grant and proof rules are those of CASE_PROTOCOL §2–§3.
- A scope of another template keeps its encoding unchanged: `price` is omitted when absent, so existing scope hashes
  do not change.

## 2. The owner's limit (local only)

When the owner approves a case, the owner also sets the **limit**:
- **Seller**: the lowest acceptable price.
- **Buyer**: the highest acceptable price.

The limit is a multiple of `step` in [`min`, `max`]. It is stored only in that owner's Client and is never sent.
The Client MAY show it to the owner's own Agent, which negotiates within it.

## 3. Messages

The payload is the CASE_PROTOCOL message object with one more member, `Price` (integer, omitted when zero):
`{"Case","Generation","ID","Kind","Candidate","Price","Digest"}`, where `Candidate` is the item id.

| kind | meaning | `Price` | `Digest` |
|---|---|---|---|
| `bid` | an offer to trade at this price | the bid | empty |
| `accept` | accepts the peer's last bid | that bid | price digest |
| `confirm_result` | confirms the peer's acceptance | the accepted price | price digest |
| `decline` | ends the negotiation without a deal | absent | empty |

**price digest** = `hex(SHA-256("anv-price-v1|" + case_id + "|" + generation + "|" + scope hash + "|" + price))`.

## 4. Rules (checked by the sender's Client before sending and by the receiver)

1. **Turns.** The initiator bids first. Bids alternate strictly. `accept` and `decline` may only answer the peer's
   latest bid (or, for `decline`, end the case at the party's turn). No message follows `accept` except the peer's
   `confirm_result`; none follows `decline` or `confirm_result`.
2. **Range.** `min` ≤ `Price` ≤ `max` and `Price` is a multiple of `step`.
3. **Concessions only.** A seller's bids never increase; a buyer's bids never decrease.
4. **Rounds.** Each party sends at most `max_rounds` bids. A party whose rounds are used up may only `accept` or
   `decline`.
5. **Accept.** `Price` equals the peer's latest bid. `Digest` is its price digest.
6. **Confirm.** Sent by the party whose bid was accepted. `Price` and `Digest` equal those of the acceptance.
7. **Owner limit (sender only).** The sender's Client refuses to send:
   - a seller's `bid` or `accept` below the seller's limit;
   - a buyer's `bid` or `accept` above the buyer's limit.
   The receiver cannot check the sender's limit.

A receiver applies 1–6 at CASE_PROTOCOL §6 step 6 (kind rules) and step 7 (transactional recheck, together with
the duplicate and limit rules). A violation is `DENIED_MESSAGE`.

## 5. Outcomes

| outcome | when | result state at both parties |
|---|---|---|
| deal | `accept` and its `confirm_result` are both delivered | `PEER_CONFIRMED` with the price |
| no deal | `decline` delivered, or both parties' rounds used up with no acceptance, or the case expires | `NO_DEAL` |

## 6. Limits

As CASE_PROTOCOL §9. Messages per side ≤ `max_rounds` + 2.
