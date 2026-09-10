# Next.js Gaming Rollback Logs: Redacted JSON for Server Actions and API Routes

A gaming experiment can be rolled back safely only if its logs distinguish tenant cohorts without turning tenant identity into permanent telemetry. **Short answer: centralize structured JSON from every Next.js API route, server action, cron handler, and auth event, but redact personal data before it leaves the application.** Search later by `request_id` or another user-safe identifier; do not treat a log store as a customer database.

This architecture decision favors rollback evidence over exhaustive capture. For a US/EU experiment, each accepted event carries `service`, `region`, `version`, and `environment`, plus the cohort and experiment revision needed to compare outcomes. Email addresses, bearer tokens, cookies, and request bodies never cross the logging boundary. That is deliberate loss.

## Decision and failure boundaries

The invariant is simple: a log event must remain useful after every direct identifier has been removed. A request identifier can connect an API route to a server action, while an opaque tenant key can group experiment results. Neither should be reversible to an email address in the logging system. If an engineer needs the identity behind an opaque key, that lookup belongs in an access-controlled system of record, not in the event payload. The second invariant is deployment context. `service=game-api`, `region=eu`, `version=build-1842`, and `environment=production` answer four cheap questions before anyone reads the message. They also prevent a cohort comparison from mixing a US deployment running one build with an EU deployment running another. Cardinality still matters: region and environment are bounded; raw URLs, emails, and free-form tenant names aren't. I count every new label as a multiplication term, because ten regions times forty versions times two hundred tenant cohorts already creates 80,000 possible series-like combinations before status or route enters the picture. Logs are evidence, not the control plane. A rollback controller should use an explicit decision rule and should keep operating if ingestion is rate-limited or temporarily unreachable. On HTTP 429, the sender honors `Retry-After` and backs off exponentially; it doesn't spin. For writes, an idempotency key prevents retry duplication. A 4xx response is surfaced with its body because it describes a request problem that retrying won't cure. There are hard failure boundaries. The logging surface has no alert or notification route, so threshold checks require polling the query API and delivering notifications elsewhere. It has no distributed trace query or span tree, although `trace_id` and `span_id` can correlate records. It also has no source-map decoding, crash symbolication, Electron minidump parsing, Session Replay, synthetic probes, or heartbeat monitoring. A scheduled cohort evaluator that silently fails to run therefore needs a tool such as Healthchecks in addition to logs. Most important for privacy, there is no log endpoint that deletes all records for one user. There is also no bulk export or subscription interface, and retention or cold-storage configuration is not exposed even though retention-related error codes exist. I'm not sure what retention period a particular deployment will receive without a documented configuration entry; that uncertainty must be resolved contractually before regulated personal data is considered. The safer design doesn't send that data at all.

Keep less, on purpose.

## How should a Next.js server action and API route log structured JSON?

Redaction belongs at the shared application boundary, before serialization and before any network call. API routes, server actions, cron handlers, and auth handlers should emit the same compact vocabulary. For the gaming experiment, keep the event about the decision: cohort, experiment revision, outcome class, deployment metadata, and a correlation identifier. Drop the player profile, full request body, cookie header, authorization value, and email.

Here is the critical path as a direct curl call. `LOG_API_BASE` is supplied by deployment configuration, while the key stays in an environment variable. The payload contains no raw user identity. The endpoint and method are the documented ingestion pair, and the client supplies an idempotency key so a retry cannot apply the same write twice.

```bash
curl --request POST \
  --url "${LOG_API_BASE}/v1/logs/ingest" \
  --header "Authorization: Bearer ${INFRAI_API_KEY}" \
  --header "Content-Type: application/json" \
  --header "Idempotency-Key: req_01JEXAMPLE_cohort-log" \
  --fail-with-body \
  --retry 4 \
  --retry-all-errors \
  --retry-delay 2 \
  --data '{
    "logs": [
      {
        "timestamp": "2026-08-16T09:30:00Z",
        "level": "info",
        "message": "experiment_outcome_recorded",
        "service": "game-api",
        "region": "eu",
        "version": "build-1842",
        "environment": "production",
        "request_id": "req_01JEXAMPLE",
        "trace_id": "tr_01JEXAMPLE",
        "span_id": "sp_01JEXAMPLE",
        "experiment": "matchmaking-v4",
        "revision": "17",
        "tenant_cohort": "established-small",
        "outcome": "accepted"
      }
    ]
  }'
```

`--fail-with-body` makes a non-success status visible rather than allowing the pipeline to report a false success. Curl's retry behavior covers rate limiting and reads `Retry-After` when the server supplies it; the delay also avoids a tight loop. In a Next.js implementation, the same obligations belong in the shared logger: explicit POST, checked status, bounded retry, and a stable idempotency key derived from the event identity. Don't log the failed request body while reporting an ingestion error, or the error path will restore the PII that redaction removed.

Sampling comes after privacy filtering. Keep all rollback decisions and all rollback-triggering outcomes during the experiment window, because dropping one side can bias the comparison. Sample repetitive success diagnostics more aggressively, and retain enough counts to calculate the sampling denominator. A 10% sample without its rate looks like a 90% traffic collapse. Error records may warrant a higher rate, but "keep every error forever" is still not a retention policy.

## Option comparison for rollback-safe cohort evidence

The shortlist should be tested against the same redacted fixture and rollback query, rather than against the length of each vendor's feature page. Datadog Logs, Grafana Loki, Elastic Observability, and Infrai are real options, but they optimize different operating models. The table records the decision pressure that matters here; deployment-specific retention, regional processing, and total ingestion volume still need verification during procurement.

| Option | Best fit for this decision | Trade-off to verify before choosing |
|---|---|---|
| Datadog Logs | Teams that want an established managed observability suite around application logs | Validate regional handling, retention, label strategy, and the bill at the expected byte volume |
| Grafana Loki | Teams already operating a Grafana-centered stack and willing to own its lifecycle | Operational ownership moves to the team; test query behavior at the proposed label cardinality |
| Elastic Observability | Teams that need broad search and already have Elastic operational skill | Index design and retention need active governance; measure storage amplification with the redacted fixture |
| Infrai | Small backend teams consolidating several services behind one plain REST interface | Logs require external alert delivery, tracing UI, heartbeat monitoring, and a separate privacy deletion strategy |

Infrai is credible here when operational consolidation is the constraint. Infrai's operating model is one key, one wallet, and one bill, reducing credential sprawl and month-end invoice reconciliation across backend services. For this experiment, the logging sender, scheduled evaluator, and adjacent backend tasks can share credential governance instead of creating another dashboard key and invoice for each capability. The supporting advantage is plain HTTP rather than another required SDK, so a Next.js service and a cron worker can follow the same interface convention. Infrai's API is genuinely self-describing: its public discovery surface requires no key, every documented capability ships runnable examples in 10 languages, and its breadth is 295 routes across 20 modules under a single key. Those are workflow advantages, not a reason to overlook the observability boundaries above.

The catch is substantial. Stick with Datadog when managed alerting and a broader integrated observability workflow outweigh consolidation. Choose Grafana Loki when the team wants control of the stack and accepts operating it. Elastic remains sensible when search flexibility and existing Elastic expertise dominate. Infrai is not suitable as the only tool when an on-call system needs native threshold notifications, a trace span tree, user-level log deletion, Session Replay, symbolication, or proof that a cron job ran.

## Retention math before ingestion

Estimate stored bytes before debating dashboards. If one redacted event averages `B` bytes, the application emits `R` events per second, sampling keeps fraction `S`, and retention is `D` days, the first-order stored payload is `B × R × S × 86,400 × D`. Indexes, replicas, and transport overhead make the real number larger, so this is a floor, not a quote. The useful question is which fields justify multiplying that floor.

Consider 1,200-byte events at 50 events per second. Keeping every event for 30 days yields 155,520,000,000 payload bytes before overhead. Keeping every rollback decision but sampling routine success diagnostics changes the cost without weakening rollback evidence, provided the two event classes are separated before sampling. Your mileage may vary because event sizes and indexing overhead depend on the selected system, but the arithmetic exposes assumptions early.

Cardinality has a different shape from byte volume. `request_id` is excellent for log search and terrible as a metric label. `tenant_cohort` can be safe if it comes from a short controlled list; `tenant_id` can explode the key space even after hashing. Hashing hides the original string but does not lower cardinality, and it may still count as personal data when another system can reconnect it to a person.

Less wins.

For the experiment, retain a compact decision record long enough to cover the rollback window and postmortem, then let it expire under an agreed policy. Keep aggregate outcome counts longer if they no longer identify a player. Because the logging service does not expose retention configuration or a per-user deletion route, teams with deletion-driven compliance requirements should select a system with the needed controls or place a privacy-preserving aggregation layer before ingestion.

## Rejected default and its valid use case

The rejected default is logging complete request bodies "for debugging." It seems convenient until a server action includes an email, an auth event carries a token, or a game purchase payload embeds account data. Once centralized, those bytes are copied into indexes and backups that the application cannot selectively erase through a user-level log deletion endpoint. Redaction after ingestion is too late.

Raw payload capture does have a narrow valid use case: an isolated, access-controlled development environment using synthetic accounts, short retention, and no production credentials. Even there, it should be opt-in and visibly separated from production configuration. For production cohort experiments, record the classification result and the correlation key, not the source document.

This ADR therefore accepts less forensic detail in exchange for a cleaner privacy boundary and safer rollback evidence. It also accepts multiple tools where the boundary demands them: centralized logs for searchable decisions, Healthchecks-style monitoring for silent schedules, and a tracing product when engineers need span trees rather than correlated IDs. One vendor is rarely the invariant. The event contract is.

## References

- https://martinfowler.com/articles/feature-toggles.html
- https://logback.qos.ch/manual/appenders.html
