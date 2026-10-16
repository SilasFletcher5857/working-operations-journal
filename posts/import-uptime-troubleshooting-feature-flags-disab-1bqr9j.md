# Import Uptime Troubleshooting: Feature Flags Disable Noisy Polling Client Retry Storms

Short answer: put a remotely evaluated feature flag in front of scheduled-import polling, make the disabled path stop scheduling new attempts, and alert on the absence of completed imports rather than on every failed uptime check. Roll it out first to one import cohort, verify that in-flight work drains, then expand it while watching request rate, completion age, and flag-evaluation failures.

The hard part isn't the Boolean. It is preserving a trustworthy signal while a retry storm is already manufacturing noise. A logistics import may poll a carrier, a warehouse feed, or a customs file drop. If the source stops producing results, eager retries can multiply identical log events without adding evidence. The alert stream then describes the polling client's behavior more loudly than it describes the business failure: no new shipment data.

Count bytes before choosing labels. A hypothetical fleet of 400 polling workers, each emitting one 700-byte event every five seconds during a 30-minute incident, produces 10,080,000 bytes before indexing overhead: `400 x 360 x 700`. Adding `shipment_id` as a label can also turn one operational condition into a high-cardinality set. Those figures are planning arithmetic, not a benchmark, but they show why a kill switch must control telemetry volume as deliberately as request volume.

## How should a feature flag disable noisy uptime checks before a retry storm?

The flag should guard the scheduler boundary. When it is off, the client must not enqueue the next poll, start a replacement timer, or let a retry callback recreate the loop. Work that has already reached the remote system needs an explicit policy: either allow it to finish within its normal deadline or cancel it through an existing cancellation path. The choice depends on whether partial imports are safe. What matters is that disabling the flag converges toward zero new checks.

Stop scheduling first.

That placement is more important than the flag provider. A guard around only the network call can leave timers firing and logging a disabled decision forever. A guard around only the retry function can still permit the normal interval to issue calls. Put the decision where ownership of future work begins, then expose a small state machine such as `running`, `draining`, and `disabled` in ordinary process metrics. This isn't decorative state. It lets an operator distinguish a successful stop from a silent evaluator failure. Keep the control plane separate from the observation it controls. The polling client may cache the last valid flag value for a bounded period, but the policy for an unavailable evaluator must be chosen in advance. For a noncritical external uptime check, fail closed may prevent an uncontrolled storm. For a safety-critical freshness check, fail open may preserve detection at the cost of more traffic. There is no universal default here; the service owner has to decide which failure is less harmful and record that decision beside the flag. Don't encode carrier IDs, shipment IDs, or arbitrary URLs in metric labels. Stable labels such as import type, region, and rollout cohort are usually enough to answer whether the stop is working. Put high-cardinality identifiers in sampled diagnostic events, with access controls and a short retention period justified by the troubleshooting window. The Google SRE monitoring guidance makes the underlying distinction useful: latency, traffic, errors, and saturation describe the system, while individual event payloads are evidence to inspect selectively.

## Make missing results the primary signal

An uptime check answers whether a request returned. The logistics question is different: when did a scheduled import last produce a valid result? The primary alert should therefore be based on completion age, evaluated against the import's expected cadence and business tolerance. A five-minute polling failure may be irrelevant for a feed expected every hour; a successful HTTP response with an unchanged or empty payload may still leave the import stale.

Use two clocks. Track the last attempted poll to prove the scheduler is alive, and track the last accepted import result to represent business freshness. Alerting only on attempts misses empty-success loops. Alerting only on results can make a deliberately disabled poller look identical to a broken one. The flag state supplies the missing context: a disabled cohort is an operator action, while an enabled cohort with increasing completion age is a fault worth paging.

One page is enough.

The secondary signals explain it: request rate reveals amplification, error rate separates remote failures from empty results, latency shows slow responses that hold concurrency, and saturation shows whether sockets, queues, or workers are approaching their limits. These map cleanly to the four golden signals. Avoid sending a page for each dimension. Route them into one incident view, and let the completion-age condition own the page unless an independent saturation threshold threatens the rest of the service.

A workable event schema also separates bounded dimensions from forensic detail. Record a low-cardinality outcome such as `accepted`, `empty`, `timeout`, or `rejected` as a metric dimension only when the vocabulary is fixed. Store a correlation identifier in a diagnostic record, not as a time-series label. If the identifier is needed during every investigation, retain a sample of those records long enough to cover the normal discovery delay; retaining all retry duplicates for the same period buys repetition, not signal.

## Signal quality, cost, and retry policy belong in one design

A retry policy changes both load and evidence. Immediate retries increase traffic at exactly the moment a dependency is least likely to benefit from it. They also create correlated log lines, repeated traces, and new metric samples. Exponential backoff with jitter spreads attempts, but backoff alone doesn't answer the operational question: should this polling cohort continue at all? The feature flag is the coarse control; retry limits and deadlines remain the local safety rails.

Sample with intent.

Consider a planning example, not a production prescription. Suppose an import runs every 60 seconds, allows three attempts, and emits 900 bytes for each attempt. Ten thousand imports can create up to 27 MB of raw attempt records in one scheduled wave: `10,000 x 3 x 900`. If the useful unit of evidence is the final import outcome, storing three near-identical records for every import triples raw event volume without tripling diagnostic value. Keep the final outcome, aggregate attempt counts, and sample detailed retry records according to an explicit investigation need. Your mileage may vary because compression, indexing, and trace enrichment change the stored size; measure those in the actual pipeline before setting retention.

The same reasoning applies to traces. Preserve traces for unusual latency, terminal errors, and a controlled sample of healthy imports. A storm-wide 100% trace rate can turn a dependency incident into a telemetry ingestion problem. Head sampling is predictable for cost but can miss a later failure; tail sampling can retain interesting outcomes but requires buffering and a decision point after spans arrive. The catch is that aggressive sampling is not suitable when every import record is required for audit or contractual evidence. In that case, keep the authoritative business audit trail, reduce operational duplicates around it, and treat the two data classes with different retention and access policies.

Here is the decision table I would put in the runbook. It makes the trade explicit instead of hiding it in a client default.

| Condition | Scheduling action | Page signal | Diagnostic retention |
| --- | --- | --- | --- |
| Results current, checks healthy | Continue normal cadence | None | Sample healthy detail |
| Results stale, request rate normal | Continue within retry budget | Completion age | Keep terminal outcomes |
| Results stale, request rate accelerating | Disable affected cohort and drain | Completion age plus saturation context | Keep deduplicated failures |
| Flag state intentionally disabled | Stop new polls | Suppress fault page; notify control state | Keep flag audit event |
| Evaluator state unknown | Apply the documented fail policy | Alert on control uncertainty | Keep bounded evaluator diagnostics |

This is also where retention math disciplines alert design. A page that cannot be investigated with the retained evidence is poorly paired with the data policy. Conversely, retaining 30 days of per-attempt debug events for an incident normally examined within hours needs a specific reason. I'm not sure a fixed retention number transfers between organizations; investigation delay, regulatory duties, and storage mechanics decide it. Measure those three, then choose the window.

## Troubleshooting without feeding the noise

Start with control state, not a search across raw errors. Confirm which cohorts evaluate the flag as enabled, draining, or disabled, and compare that with scheduler start rate. If starts continue after the cohort is disabled, the guard is below the scheduler boundary or a second scheduling path exists. If starts stop but completion age keeps rising, that is expected while disabled and should be visible as an acknowledged business risk, not relabeled as health.

Next, compare accepted-result rate with attempt rate. A widening gap indicates work without outcomes. Then inspect latency and concurrency: slow polls can overlap, and a timer that schedules independently of completion can create amplification even without a conventional retry loop. Finally, inspect a bounded sample of terminal outcomes. This order moves from low-cardinality facts to expensive detail, which keeps the investigation usable during the event.

Resist the urge to add `shipment_id` to every counter when a dashboard feels vague. That move can make the dashboard expensive before it makes the cause clear. Ask a narrower question: is the failure isolated by import type, region, dependency, or rollout cohort? Those dimensions have controlled vocabularies and can guide the next diagnostic query. Only then use a correlation identifier to retrieve the relevant event or trace.

The kill switch is not suitable as the only protection against a tight local loop. If a process can schedule thousands of retries before a remote flag refresh arrives, enforce a local attempt ceiling, one active poll per import, a deadline, and bounded concurrency. Likewise, stick with a static configuration change when the polling job is deployed infrequently, can be restarted safely, and has no requirement for rapid cohort control. A runtime flag earns its complexity when operators need a reversible, targeted stop without shipping code.

## Roll out the control in small, observable steps

First, add the scheduler-boundary guard with the default preserving current behavior. Test that one import cannot own more than one active poll and that disabling prevents all future scheduling paths. The test should advance fake time far enough to cover the normal interval and the longest retry delay; checking only the next callback leaves room for a later timer to revive the loop.

Second, enable flag evaluation for an internal or low-impact cohort while leaving the value on. Compare scheduler starts, attempts, and accepted results with the previous baseline. This phase tests evaluation and cohort assignment without changing availability. Then turn the flag off for that cohort, confirm that new starts reach zero after the defined drain period, and verify that the completion-age alert is acknowledged or routed according to the runbook.

Third, expand by a bounded cohort dimension such as import type or region. Avoid per-shipment targeting: it raises cardinality in both flag audit records and operational telemetry, and it is difficult to reason about during an incident. Keep a rollback condition tied to business freshness, control-plane uncertainty, and unexpected scheduler starts.

Remove the flag after the incident class has a durable policy. Permanent emergency flags accumulate ambiguity: nobody knows whether `off` means healthy, mitigated, or forgotten. The lasting design should retain bounded retries, concurrency control, outcome-based freshness monitoring, and a documented means to pause imports. The temporary rollout control can then leave without taking the safety model with it.

## References

- Google SRE Book, "Monitoring Distributed Systems": https://sre.google/sre-book/monitoring-distributed-systems/
- Electron documentation, "crashReporter": https://www.electronjs.org/docs/latest/api/crash-reporter

## Further reading

- Google SRE Book, "Monitoring Distributed Systems": https://sre.google/sre-book/monitoring-distributed-systems/
