# Bridge API v1

Status: draft (2026-10-02). Licence: CC BY 4.0.

The Bridge is a local process that gives a Client access to the Pilot network. It is deliberately **thin**: it
opens and accepts streams, moves bytes, and reports transport facts. It never builds, parses, signs or verifies
application messages; the Client does all of that from `WIRE_FORMAT.md` and `CASE_PROTOCOL.md`.

## 1. Trust model

- **The Bridge is trusted for transport facts only**: the identity of the remote peer of a stream (Pilot address,
  as authenticated by the Pilot daemon), the set of peers the daemon trusts, registry key lookups, and whether bytes
  were written or read. This is the same trust a Client places in the Pilot daemon itself.
- **The Bridge is not trusted for authorization.** Every proof (signatures, payload hashes, case binding) is
  verified by the Client over the raw bytes it read. A faulty Bridge can cause loss, delay or a misattributed
  transport identity; it cannot make the Client accept a message whose proofs do not verify against the keys the
  Client pinned.
- The Bridge and the Client run as the same OS user in this version. Isolating the Client's key files from the
  Bridge is **not** a goal of v1; a deployment that needs it runs the Bridge as a separate user.

## 2. Transport

- A Unix-domain stream socket at `<runtime-dir>/bridge.sock`. The Bridge creates `<runtime-dir>` with mode 0700
  (refusing to start if it exists with other permissions or another owner), removes a stale socket only if it is
  a socket owned by the same user, and creates the socket with mode 0600.
- On accept, the Bridge checks the peer credentials (`getpeereid` / `SO_PEERCRED`) and closes connections from any
  other uid. The Client checks the same of the Bridge before sending anything.
- One connection is one **session**. Messages are UTF-8 JSON objects, one per line, at most 262144 bytes per line.
  Lines longer than that, or lines that are not a valid request, close the session.
- **One Bridge per runtime directory.** The Bridge holds an exclusive lock on `<runtime-dir>/bridge.lock` for its
  lifetime; a second instance refuses to start. Only then may it replace a leftover socket file.

### Messages
```
request   {"id": <uint>, "method": "<name>", "params": {...}}
response  {"id": <uint>, "result": {...}}            or  {"id": <uint>, "error": {"code": "<CODE>"}}
event     {"event": "<name>", ...}                     (Bridge → Client, no id)
```
- Requests may be pipelined; responses carry the request id and may arrive out of order. At most 64 requests of a
  session are in progress at once; further requests are answered `BUSY`.
- Error codes are fixed strings (§6). Errors never carry message bytes, addresses of third parties or file paths.
- Binary data is standard base64 with padding (`data_b64`).

## 3. Session methods

| method | params | result |
|---|---|---|
| `hello` | `{"api": 1}` | `{"api": 1, "bridge_version": "...", "addr": "<own Pilot address>", "public_key": "<hex Ed25519>"}` |
| `peer.handshake` | `{"addr", "reason"}` | `{}` — asks the daemon to request a trust handshake with `addr` |
| `peer.trusted` | `{}` | `{"peers": [{"addr", "public_key"}]}` — `public_key` may be `""` if the daemon does not report it |
| `registry.lookup` | `{"addr"}` | `{"public_key": "<hex>"}` — the key the registry currently holds for that address |
| `listen` | `{"port": 1001}` | `{}` — from now on the session receives `stream.incoming` events for that port |

- `listen` ownership is exclusive: another session asking for the same port gets `BUSY` until the owner session
  ends; a session that has ended can never become or remain the owner. Streams that arrive while no session listens
  are closed (the sender sees a reset).
- `peer.handshake`, `peer.trusted` and `registry.lookup` take no `timeout_ms`; the Bridge bounds each at 10 s and
  answers `TIMEOUT`.

Addresses are Pilot addresses in their canonical text form (`<network>:<node as XXXX.XXXX.XXXX hex>`); the Bridge
normalizes before reporting, and the Client compares strings exactly.

## 4. Streams

| method | params | result |
|---|---|---|
| `stream.dial` | `{"addr", "port", "timeout_ms"}` | `{"conn": "<handle>"}` |
| `stream.write` | `{"conn", "data_b64", "timeout_ms"}` | `{"written": <n>}` — all bytes written, or an error |
| `stream.read` | `{"conn", "max": <1..65552>, "timeout_ms"}` | `{"data_b64": "...", "eof": <bool>}` — up to `max` bytes; `eof` when the peer closed and nothing remains |
| `stream.close` | `{"conn"}` | `{}` |

Event:
```
{"event": "stream.incoming", "conn": "<handle>", "port": 1001, "remote_addr": "<Pilot address>"}
```
- **No bytes are read from an incoming stream until the Client calls `stream.read`.** The Client therefore checks the
  remote peer (enrollment, registry pin, daemon trust) before any byte of the message is consumed — the receive
  order of `CASE_PROTOCOL.md` §6.
- `remote_addr` is the address of the stream's actual remote end as reported by the daemon (not a value from the
  message).

### Handles
- A handle is a random 128-bit value, hex-encoded, valid only in the session that created or received it.
- Use from another session, after `stream.close`, or after the stream failed returns `UNKNOWN_CONN`.
- When a session ends, the Bridge first releases the session's `listen` ports, then closes every stream of that
  session in the background (a slow daemon close cannot block a replacement Client). Handles are never reused.
- At most 16 open streams per session (dialed + incoming); a dial reserves its slot before it starts. Further
  `stream.incoming` streams are closed by the Bridge immediately (the sender sees a reset), and `stream.dial`
  returns `BUSY`.
- A handle is retired (later calls get `UNKNOWN_CONN`) after `stream.close`, after any `stream.write` error
  (including `TIMEOUT`: an unknown number of bytes may have been sent), and after a `stream.read` error other than
  `TIMEOUT` (a read timeout consumes nothing; the stream stays usable).
- Calls of the same kind on one handle (reads, or writes) are executed one at a time; their order is the order in
  which the Bridge starts them, which for pipelined requests is not guaranteed. A Client that needs ordering waits
  for each result before the next call of that kind.
- `stream.write` carries at most 65552 bytes of data per call.

### Delivery semantics (what a Client may conclude)
- `stream.write` success: the bytes were handed to the daemon. It does **not** mean the peer received them.
- If the session or the Bridge fails after a `stream.write` was sent but before its response, the Client must treat
  the message as possibly delivered (`DELIVERY_UNKNOWN`) and retry later under the case protocol's rules. The
  Bridge never resends anything on its own.
- A Client MUST bound every call — waiting to send it, sending it and waiting for the response — by one deadline
  (the call's `timeout_ms`, or 10 s, plus a grace period). A Bridge that does not answer in time is treated as lost:
  the Client closes the session, which retires every handle of that session.
- **Late delivery is possible and safe by construction.** Once a Client has submitted a `stream.write` to the Bridge
  (the **hand-off**), the bytes cannot be recalled: they may reach the daemon and the peer later than the call's
  timeout, even after the Client closed the stream or the session. A stalled daemon operation is bounded by the
  Bridge's stall watchdog (§5): the Bridge exits, which ends every pending operation. A Client therefore (a) submits
  a message only while it is authorized to send it, checked immediately before submission (CASE_PROTOCOL §5),
  (b) treats any write without a successful result as `DELIVERY_UNKNOWN`, and (c) relies on the receiver's own
  checks at receipt (CASE_PROTOCOL §6), which include the proof's and the case's expiry. A late delivery is
  equivalent to a slow network.
- A reset (Pilot RST) and an orderly close are both reported as `eof` (a limitation of the daemon interface); a
  Client must never treat `eof` as a complete message — completeness is decided by the frame length (WIRE_FORMAT).

## 5. Timeouts and limits
- `stream.dial`, `stream.write` and `stream.read` take `timeout_ms` (1..30000). On expiry they normally return
  `TIMEOUT`: after a read the stream stays usable; after a write the handle is retired (§4); a dial that timed out
  leaves no stream.
- **Stall watchdog.** If the daemon itself stalls, an operation can block beyond its timeout. Any single daemon or
  registry operation (dial, read, write, close, handshake, trusted peers, lookup, listen) that runs longer than 60 s
  makes the Bridge exit; its supervisor restarts it with a fresh daemon connection. Meanwhile the Client's own call
  deadline (§4) has already treated the session as lost.
- `peer.handshake`, `peer.trusted`, `registry.lookup` and `listen` take no `timeout_ms`; the Bridge bounds each at
  10 s (`TIMEOUT`). At most 16 such daemon/registry calls run at once per Bridge, including ones that already timed
  out and are still waiting for the daemon; beyond that the Bridge answers `BUSY`.
- `stream.read` `max` is at most 65552; `stream.write` carries at most 65552 bytes after base64 decoding. A Client
  reading a frame loops until it has the length the frame header declares.

## 6. Error codes
`BAD_REQUEST`, `UNSUPPORTED_API`, `DAEMON_UNAVAILABLE`, `REGISTRY_UNAVAILABLE`, `NOT_FOUND`, `UNKNOWN_CONN`,
`BUSY`, `TIMEOUT`, `DIAL_FAILED`, `WRITE_FAILED`, `READ_FAILED`, `CLOSED`, `INTERNAL`.

## 7. Versioning
`hello` negotiates `api`. A Bridge that does not support the requested version returns `UNSUPPORTED_API`.
New optional fields may be added to results; Clients ignore unknown result fields. New methods get new names.
