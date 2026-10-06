# pan-protocol

Open specifications of the Agent Network:

| Document | What it specifies |
|---|---|
| `spec/WIRE_FORMAT.md` | The bytes on the wire between two Clients: frames, governed frames, Intent/Decision/Mandate canonical encoding, hashes and signatures (the profile we use of Pilot Protocol's formats). |
| `spec/CASE_PROTOCOL.md` | The case protocol between two Clients: resources, message kinds, acknowledgement codes, retries. |
| `spec/BRIDGE_API.md` | The local API between a Client and a Bridge (the process that talks to the Pilot daemon). |
| `spec/TEMPLATE_PRICE_NEGOTIATION.md` | Template `price-negotiation-v1`: bounded price bids between a seller and a buyer; each owner's limit stays local. |
| `spec/PROFILE_V2_DELEGATION.md` | Profile v2: per-case delegate keys, so the key that approves cases never signs automatic messages. |
| `vectors/` | Test vectors for `WIRE_FORMAT.md`, generated with the reference implementation in `pan-bridge`. |

Anyone may implement these specifications. Licence: documents CC BY 4.0 (`LICENSE`); test vectors CC0 1.0 (`vectors/LICENSE`).

Related repositories: `d25-dev/pan-bridge` (AGPL-3.0, Bridge reference implementation); the Agent Network Client is proprietary.

Copyright (c) 2026 Yuya Uwatoko.
