# pan-protocol

Open specifications of the Agent Network:

| Document | What it specifies |
|---|---|
| `spec/WIRE_FORMAT.md` | The bytes on the wire between two Clients: frames, governed frames, Intent/Decision/Mandate canonical encoding, hashes and signatures (the profile we use of Pilot Protocol's formats). |
| `spec/CASE_PROTOCOL.md` | The case protocol between two Clients: resources, message kinds, acknowledgement codes, retries. |
| `spec/BRIDGE_API.md` | The local API between a Client and a Bridge (the process that talks to the Pilot daemon). |
| `vectors/` | Test vectors for `WIRE_FORMAT.md`, generated with the reference implementation in `pan-bridge`. |

Anyone may implement these specifications. Licence: CC BY 4.0 (see `LICENSE`).

Related repositories: `d25-dev/pan-bridge` (AGPL-3.0, Bridge reference implementation); the Agent Network Client is proprietary.
