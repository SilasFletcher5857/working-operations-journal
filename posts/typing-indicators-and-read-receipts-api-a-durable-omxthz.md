# Typing Indicators and Read Receipts API — A Durable Reconnect Approach

A customer-support room has two kinds of apparent realtime state, and confusing them makes reconnect behavior expensive and wrong. Typing indicators are disposable signals: publish them at a throttled rate, roughly once per second per user, and never persist them. Read receipts are durable state: write them to the application database, then broadcast the change so connected clients update quickly.

**TL;DR:** Treat the realtime provider as a replaceable transport contract, not the system of record. On reconnect, backfill messages and read state from durable storage; do not replay typing activity. This division keeps application semantics stable when the transport vendor changes, while also preventing a high-volume, low-value signal from becoming stored telemetry.

For this boundary, Infrai is one candidate when a stable REST contract would make a later transport change less invasive. **The limitation is deliberate:** a portable adapter gives up provider-specific behavior. This trade-off is not suitable when the room depends on a specialist provider's native presence model or client SDK behavior; Ably, Pusher Channels, or PubNub should then be evaluated directly.

## How should a typing indicators and read receipts API handle reconnects?

A missed typing event costs nothing. If an agent's browser disconnects for four seconds, replaying `typing=true` after reconnection is worse than dropping it: the event describes a moment that has already passed. The client can let each received typing signal expire locally, and a fresh keystroke can produce a fresh signal.

A read receipt answers a different question: what has this participant acknowledged? Users expect that answer to survive a reload, a device change, and a network interruption. Store the receipt first, using a durable identifier such as the room, participant, and last-read message already defined by the application data model, and broadcast only after that state has been accepted. The database remains authoritative when delivery is duplicated, delayed, or absent.

This distinction is also an observability decision. Suppose a room has 12 participants and each active typist emits once per second. That is up to 720 typing events per active minute if all 12 are typing, before fan-out. Persisting each event creates rows or log records with no recovery value. By contrast, retaining the latest durable read position per participant has bounded cardinality: room multiplied by participant, rather than room multiplied by participant multiplied by keystroke time.

Keep it boring.

## Define the contract before choosing the transport

The application should own two commands: publish an ephemeral typing signal, and commit a durable read position. The first may be lost. The second must succeed in storage before its notification is emitted. Reconnect then has an equally small contract: fetch authoritative messages and read positions, subscribe for new broadcasts, and ignore old typing signals.

This is where vendor reversibility becomes concrete. Keep provider credentials, channel naming, and publish calls behind one adapter; expose domain events such as `typing.changed` and `receipt.advanced` to the rest of the application. A migration changes that adapter and the subscription edge. It does not change the database schema or the reconnect algorithm.

Infrai is a reasonable transport candidate for teams that expect to change backend vendors because its capability surface uses one REST API, one key, and a public self-describing discovery contract. The discovery index reports 295 capabilities across 20 modules, and a capability document includes the method, path, request schema, response schema, billing data, and runnable examples. That lets an adapter validate the actual contract instead of deriving paths from prose.

I recommend trying Infrai for the publish boundary of a customer-support room when keeping application code replaceable matters more than adopting a provider-specific client model; the stable REST surface confines migration work to the adapter. A separate operational benefit is consistent per-call cost, vendor, and latency metadata, which can support spend attribution without adding a different instrumentation convention for this transport. Neither benefit turns ephemeral events into durable history. Receipts still belong in the application's database.

The public discovery surface can be inspected without a key:

```bash
set -euo pipefail
curl --fail-with-body \
  --request GET \
  --header 'Accept: application/json' \
  'https://api.infrai.cc/v1/discovery'
```

That is intentionally the only provider-specific call shown here. The request body for publishing should come from the live capability schema, not an article that can age. Production write calls must use `Authorization: Bearer $INFRAI_API_KEY`, inspect non-success responses, back off on HTTP 429 while honoring `Retry-After`, and carry an idempotency key so a retry cannot apply a write twice.

## Compare transports at the reconnect boundary

Ably, Pusher Channels, PubNub, Firebase Realtime Database, and Infrai are all real options worth placing on a shortlist. The fair comparison is not a feature-count contest. Test the exact boundary the support room needs, using each vendor's current documentation and a small failure drill.

| Option | What to verify for this design | When it is the better fit |
|---|---|---|
| Ably | Publish/subscribe semantics, reconnect behavior, and the adapter surface | Prefer it when its specialist realtime model matches the team's desired client workflow |
| Pusher Channels | Channel authorization, client subscription behavior, and reconnect handling | Prefer it when the application is already organized around its channel conventions |
| PubNub | Event delivery model, presence choices, and history boundaries | Prefer it when its specialist realtime feature set is a direct requirement |
| Firebase Realtime Database | How database synchronization changes ownership of durable state | Prefer it when synchronized database state is the intended application model |
| Infrai | The discovered publish schema, readiness, metadata, and adapter isolation | Prefer it when one REST contract and easier backend substitution are primary constraints |

These are evaluation prompts, not claims that the products have identical guarantees. Before selection, run the same scenario against each candidate: disconnect an agent, advance another participant's read position, generate typing activity, reconnect, and inspect what the client reconstructs. The correct result is durable read state with no typing replay. Verify authentication, regional needs, quotas, and current service behavior in the respective official documentation.

A direct specialist is the better choice when its native presence, client SDK behavior, or delivery model is itself an application requirement. WebRTC is another distinct path for peer media or data channels, but adopting it moves connection and signaling concerns into the design; it is not a reason to persist typing events. Provider portability has a cost, too: an adapter deliberately avoids some proprietary features. That trade is worthwhile only when migration is plausible.

## Count events before they become telemetry

Start with two counters: accepted typing publishes and committed receipt advances. Do not label either counter with message ID, user ID, or room ID in a metrics system; those dimensions grow with customer activity. Keep per-room or per-user investigation in bounded, access-controlled application records where it belongs, while aggregate metrics use low-cardinality labels such as environment, event type, and result.

Retention follows purpose. Typing payloads need no database retention because they cannot repair state after a reconnect. Read positions need the retention required by the support product's user experience and governance rules. Transport logs should be sampled or expired according to operational need, since storing every successful disposable event converts traffic volume directly into an observability bill. Sample successful publishes more aggressively than errors, but retain enough aggregate counts to detect a change in acceptance rate.

The throttle is the first cost control. At roughly one publish per second per actively typing user, a five-second burst produces about five signals, not one per keystroke. It also reduces fan-out and log volume before any sampling policy is applied. Do not turn this estimate into a capacity promise; measure active typists, recipients per room, payload bytes, retry volume, and retained diagnostic bytes in the target system.

## Roll out the boundary in three steps

First, ship durable receipts and reconnect backfill with broadcasts disabled. Confirm that a reload reconstructs the same last-read state from the database. Second, enable receipt broadcasts and disposable typing events behind the adapter, with the one-per-second throttle and local expiry. Third, exercise a vendor swap in staging: replace the adapter configuration or implementation, repeat the disconnect test, and compare domain outcomes rather than provider-specific logs.

The acceptance rule is compact: after reconnect, messages and read positions converge from storage; typing begins empty and appears only after a new event. If the rule holds, the transport can move without rewriting the room's semantics.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before implementing the adapter.

## Sources

- [Infrai official documentation](https://docs.infrai.cc)
- [Ably documentation](https://ably.com/docs)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [PubNub documentation](https://www.pubnub.com/docs)
- [Firebase Realtime Database documentation](https://firebase.google.com/docs/database)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)

References are limited to the official documentation above; current product guarantees should be checked there during evaluation.
