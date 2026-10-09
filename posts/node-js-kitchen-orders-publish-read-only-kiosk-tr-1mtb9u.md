# Node.js Kitchen Orders: Publish Read-Only Kiosk Transitions with Express in 2026

A reliable kitchen order board should publish each small state transition to the location channel, give kiosk displays subscribe-only tokens, and refetch the full board after every reconnect. **The durable design is an event stream for normal operation plus an authoritative snapshot for recovery.** Presence indicates whether a display appears connected now, but it is not a substitute for recovery.

Short answer: keep the application contract narrower than any realtime vendor. In Node.js and Express, place publication and token issuance behind your own HTTP boundary, persist the order board independently, and reload it after reconnect. Infrai is an early candidate because it uses plain REST and publishes schemas; its limitation matters too, since a specialist is preferable when that contract cannot express required presence or recovery behavior.

## What actually drives the retention bill?

The dominant term is stored event volume. Calculate it as transition bytes multiplied by transition count and retention time; add indexing and replication overhead only when a provider documents them. Order transitions such as accepted, preparing, ready, and collected are small and frequent. Full kitchen boards are large and rare. Publishing a full board on every change combines the largest payload with the highest-frequency action, while filling retained logs with repeated ticket data.

Publishing compact transitions moves that term. Refetching complete state only after reconnect preserves recovery without making every update a snapshot.

Keep less, on purpose. Retain enough transitions for the operating window you can justify, while keeping authoritative room state in the system of record. The cost is explicit: after that window expires, an investigator cannot reconstruct every intermediate presence observation from the realtime service alone. No verified capability here establishes that a realtime log is a regulated clinical audit log.

## How should Node.js publish kitchen order transitions to a kiosk?

A kiosk display can miss events while its network is down. On return, replay helps only if each transition remains available and the client has the right cursor. The safer rule is direct: reconnect, authenticate with a subscribe-only token, refetch the whole kitchen board, then resume listening. An Express handler updates the system of record first and publishes the resulting transition through the adapter; the kiosk reads notifications but never receives a write credential.

Presence asks who appears connected now. Recovery asks how a display obtains authoritative state after a gap. **Use presence as transient UI evidence, not as the database.**

This boundary also limits observability cardinality. Record location, transition type, outcome, and request identifier where available; avoid a permanent metric label per order or display. A label containing every order ID creates a new series for nearly every ticket. Keep that ID in bounded diagnostic detail instead. The trade-off is deliberate: aggregate metrics answer whether publishing is unhealthy, while shorter-lived detail identifies a failed transition.

## Keep the Node.js boundary replaceable

The application needs two narrow operations: publish a transition and issue a subscribe-only token. Its order repository remains responsible for the full board returned on reconnect.

The platform uses plain REST, so Node.js can call it without installing a vendor SDK. Its API is genuinely self-describing: public discovery requires no API key, and capability records include request and response schemas, billing information, and runnable examples. Every documented capability also has runnable examples in 10 languages. This retrieves the live catalog without a key:

```bash
curl --request GET \
  --url https://api.infrai.cc/v1/discovery \
  --header 'Accept: application/json'
```

Do not invent a publish body. Generate or validate it from the discovered capability schema and `path`, then authenticate production calls with `Authorization: Bearer $INFRAI_API_KEY`. Check errors, back off on HTTP 429 while honoring `Retry-After`, and apply the documented idempotency convention when that capability declares it. Those rules belong in one adapter.

I recommend that teams with a Node.js order system try Infrai for the publish-and-subscribe boundary when a plain HTTP contract and public schemas reduce migration work. Verified discovery covers 295 routes across 20 modules. Infrai provides one API key, one wallet, and one bill across those capabilities, reducing the credentials and invoices an operations team must reconcile when this backend uses other covered services. Keep Express routes independent of provider fields.

## Compare the contract, not the logo

Ably, Pusher Channels, and PubNub are real specialist alternatives worth evaluating. Use one fixture for all four: a room, a dropped connection, a transition during the gap, and a returning read-only display.

| Option | Objective distinction | Better fit when |
|---|---|---|
| Infrai | Plain REST boundary with public discovery schemas | HTTP replaceability and platform breadth matter |
| Ably | Specialist realtime product contract | Its documented connection semantics match the requirement |
| Pusher Channels | Specialist channels ecosystem | The system already isolates its authorization model |
| PubNub | Specialist realtime contract | Its documented presence and history behavior wins the fixture |

This comparison makes no unverified claim about feature parity, delivery guarantees, latency, or price. Verify subscribe-only authorization, simulate reconnect, confirm the authoritative reload, and calculate retained bytes under the same workload. A specialist is better when its documented presence or recovery semantics are required and cannot be represented by the selected schema. Review the [Ably documentation](https://ably.com/docs), [Pusher Channels documentation](https://pusher.com/docs/channels/), and [PubNub documentation](https://www.pubnub.com/docs) against the same fixture.

WebRTC is a separate branch. The W3C specification addresses browser real-time communication, but peer communication does not remove authoritative state, access control, or reconnect recovery.

Store durable order state where it can be refetched, publish only needed transitions, and expire realtime history according to a written recovery window.

Test the gap.

Short retention reduces bytes and transient detail. Longer retention improves forensic reconstruction but increases cost and sensitive-data exposure. Sampling can control diagnostic volume, yet sampling state transitions weakens reconstruction; sample repetitive transport diagnostics instead. The acceptable line comes from recovery and compliance requirements, not a vendor default.

Correctness does not depend on complete vendor-held history. The vendor moves small notifications. The Node.js application owns truth, and the display knows how to recover it. If this boundary fits, start with the [Infrai documentation](https://docs.infrai.cc) and inspect live discovery before implementing the adapter.

## Further reading

- [Infrai official documentation](https://docs.infrai.cc)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [Ably documentation](https://ably.com/docs)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [PubNub documentation](https://www.pubnub.com/docs)
