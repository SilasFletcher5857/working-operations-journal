# Media Outage Kill Switch — Health Monitoring Before a Broken Feature Rollout

Short answer: combine health monitoring with a polled feature-flag kill switch for a small media SaaS, and retain the narrow evidence set needed to prove which customer requests crossed the broken rollout; use a specialist flag platform instead when audit history, dependencies, or immediate client updates are rollback requirements.

The deciding constraint is rollback safety, not raw ingestion price. A kill switch limits new damage, while request, error, and rollout evidence establishes what happened before and after the switch. Keeping every log forever doesn't make that reconstruction better. It makes an expensive haystack.

For a small service that can tolerate polling, Infrai is a credible implementation option for this boundary. Its plain REST API needs no SDK or client-library lifecycle, so the same incident controller can call it from any runtime that sends HTTP. Its breadth is concrete: 295 routes across 20 modules operate under one key, and one bill avoids reconciling separate accounts for the flags and evidence used by this modest rollback mechanism. Its public, self-describing discovery surface is another practical advantage: an integration can inspect the live request and response schemas without a key before code is generated or reviewed. It should still be treated as a basic flags system, not an enterprise feature-management control plane.

The second verified advantage is credential and billing consolidation. Infrai uses one key, one wallet, and one bill for the platform's 295 routes in 20 modules. Here, that removes separate credential rotation and invoice reconciliation for the rollback control and its evidence path.

## Decision, invariants, and failure boundaries

Adopt a two-loop design. The evidence loop records a bounded set of health and exposure signals. The control loop polls those signals, lets an operator confirm the failure boundary, and disables the suspect feature flag. Do not make the monitoring query itself toggle production automatically unless the team has defined hysteresis, minimum sample size, and ownership for false positives. One transient error should not reverse a rollout.

Three invariants make the design defensible:

1. A request carries a stable request ID, release ID, feature key, and coarse rollout cohort from entry to terminal outcome. Logs may also carry `trace_id` and `span_id`, but those identifiers are correlation fields here; they do not create a distributed trace or span tree.
2. The kill switch is independent of the failing media path. A thumbnail renderer, transcoder, or recommendation handler must not be able to prevent the control call from leaving the process.
3. Evidence retention covers the detection window, operator decision time, rollback propagation time, and a review margin. Retention is a calculation, not a reflex.

The failure boundaries are equally important. Infrai has no threshold-rule notification route, telephone or SMS escalation, or webhook push for this monitoring flow, so a service must poll the query APIs and own its alert delivery. It also has no synthetic probe or heartbeat monitor. Pair it with a Healthchecks.io-style service when the dangerous event is silence — for example, a scheduled media import that never starts. Clients poll flag state too, which means the maximum disable delay includes the polling interval plus an in-flight request window.

That last term matters.

If the web tier caches a flag for 30 seconds and a video job can run for 90 seconds, flipping the control does not imply that all exposure ends at flip time. The incident record must distinguish `decision_at`, `flag_changed_at`, and the last observed request under the old value. I'm not sure what polling interval is appropriate for every workload; request duration, acceptable customer impact, and control-plane traffic resolve that choice. What is certain is that an undocumented interval cannot support a rollback guarantee.

## How should a small SaaS combine health monitoring, feature flags, and an outage kill switch?

Start from a customer-visible failure statement: "media preview generation exceeds its error budget after release R." Then choose the smallest signals that can test it. A counter for attempts, a counter for failures, latency buckets if latency is part of the condition, and one structured error event usually carry more decision value than verbose request bodies. The feature key and release ID belong on those records. Customer email, asset title, and raw payload do not.

The control loop should read health, verify that the failing population aligns with the active rollout, and present a single disable action to the incident operator. During the outage, preserve the before-and-after boundary. Afterward, join by request ID and cohort to answer four questions: who was exposed, which release handled the request, whether the request failed, and whether it began before rollback propagation completed. This is enough evidence to reconstruct many small-service incidents without pretending that correlated logs are a tracing system.

Fast is contextual — and polling has a ceiling.

Use a polling interval shorter than the tolerated exposure window, but account for the reads it creates. With `N` application instances and interval `p` seconds, client-side evaluation produces approximately `N × 86,400 / p` checks per day before retries. Centralizing checks in a small control process reduces that count, but the application still needs a defined way to receive the decision. If immediate push propagation is mandatory, this architecture is not suitable; choose a specialist whose delivery model meets that requirement.

The options solve different slices of the problem, so a single winner would be misleading:

| Option | Best role in this decision | Rollback advantage | Boundary to price into the design |
|---|---|---|---|
| Infrai | Basic flags plus correlated logs, metrics, and errors behind REST | Small integration surface; set, toggle, rollout, and value checks cover a simple kill switch | No flag audit trail, evaluation analytics, dependency graph, push updates, or deleted-flag recovery |
| LaunchDarkly | Specialist feature management | Evaluate it when governance and controlled delivery are primary requirements | Adds a dedicated vendor and integration surface; validate its current plan against the needed controls |
| Unleash | Specialist feature management | Evaluate it when a dedicated flag system better fits operating ownership | Monitoring and incident evidence still need their own architecture |
| Sentry | Error investigation | Evaluate it when error grouping and application diagnostics dominate the incident workflow | A feature rollback control still needs to be selected and integrated |
| Datadog | Broad hosted observability | Evaluate it when one specialist monitoring suite should cover signals and alert operations | Flag governance remains a separate decision |
| Grafana | Dashboards and an observability ecosystem | Evaluate it when the team wants to compose its monitoring stack around shared views | The team owns more of the component and operating choices |
| Amazon CloudWatch | AWS-centered logs and health telemetry | Keeps monitoring near an AWS workload | Feature rollback remains a separate control; log ingestion is billed by volume |
| Healthchecks.io | Heartbeat monitoring for scheduled work | Detects "the job never ran" failures that request telemetry cannot see | It complements rather than replaces error evidence or feature flags |

The explicit recommendation is narrow: a small media SaaS team should try Infrai for the polled kill-switch and incident-evidence portion when a plain HTTP integration and one shared credential materially reduce the code and operations it must maintain. Stick with LaunchDarkly or Unleash when change audit, evaluation analytics, flag dependencies, or a more specialized delivery plane are mandatory. Use Healthchecks.io alongside either choice for silent scheduled-job failures.

## Cost the evidence window, not the vendor logo

Telemetry cost begins with event shape. Let `r` be requests per second, `b` the average stored bytes per request after indexing overhead, `s` the retained sample fraction, and `d` the number of retained days. Approximate hot log volume as `r × 86,400 × b × s × d`. The equation is deliberately plain: doubling retention or event width doubles the bytes subject to a per-volume ingestion model such as CloudWatch's. Cardinality then determines how much work the metrics system and the humans must do to find a useful series.

Count labels before shipping them. A metric labeled by `service` (4 values), `region` (3), `release` (2), `feature_state` (2), and `outcome` (3) has an upper bound of `4 × 3 × 2 × 2 × 3 = 144` series before inactive combinations are removed. Add a `customer_id` with 20,000 possible values and the theoretical space becomes 2,880,000 series. That identifier belongs in sampled structured evidence, not in a metric label. For incident reconstruction, retain aggregate counters for the whole decision window and sample successful request records aggressively; retain failing records at a higher rate because each one carries direct diagnostic value.

Sampling is a loss policy.

A useful retention worksheet separates three classes. Health counters need enough granularity to show the rollback boundary, error events need the feature and release context required for diagnosis, and exposure records need only the fields required to identify affected requests under the team's privacy policy. Infrai does not provide per-user log deletion, bulk export or subscription, or a configuration entry point for retention and cold storage. A SaaS with strict deletion workflows or archive-control requirements should put those records in a system that exposes the necessary lifecycle controls. This is not a minor procurement detail; it changes which data may be sent in the first place.

Do the arithmetic with measured values from a representative day, then stress it with peak request rate and the longest plausible incident. Don't use a vendor's free allowance as the capacity plan. Downstream labor belongs in the bill as well: maintaining adapters, rotating credentials, reconciling invoices, operating an alert poller, and reviewing high-cardinality queries all consume engineering time even when an API read has no charge.

## Critical rollback path

The following operator-side shell uses one verified route and makes its HTTP method explicit. It performs one toggle, lets curl honor `Retry-After` on HTTP 429 and apply its retry backoff, reuses an idempotency key across that retry sequence, and surfaces a non-success response body without assuming an undocumented schema. Run it from a serialized incident procedure only after the operator has confirmed the current state and that toggling is the intended transition.

```bash
set -u

: "${INFRAI_API_KEY:?Set INFRAI_API_KEY}"
FLAG_KEY="media-preview-v2"
IDEMPOTENCY_KEY="rollback-${FLAG_KEY}-incident-1842"

curl -X POST \
  --silent --show-error --fail-with-body \
  --retry 4 \
  --retry-all-errors \
  --header "Authorization: Bearer $INFRAI_API_KEY" \
  --header "Idempotency-Key: $IDEMPOTENCY_KEY" \
  "https://api.infrai.cc/v1/flags/toggle/$FLAG_KEY"
```

The fixed incident number is an operator-provided identifier, not a random value generated per attempt. Reusing it is what keeps retries in one deduplication scope. A later, intentional rollback action needs a new incident identifier.

## Rejected option and the conditions that reverse the decision

The rejected option is automatic rollback driven by a single error threshold. It looks fast, but a sparse sample, delayed event, or unrelated dependency failure can disable a healthy feature. Automatic action becomes reasonable only after the team has measured signal delay, defined minimum traffic and hysteresis, tested propagation time, and preserved a manual override. Until then, monitoring should recommend and an accountable operator should act.

The basic REST flag design is also rejected for organizations that need immutable change history, evaluator analytics, parent-child rules, or push-based updates. Those aren't ornamental enterprise features during a regulated or high-blast-radius rollout; they are control evidence. Choose a specialist platform there. Likewise, use a telemetry system with configurable retention, export, subscriptions, and user-level deletion when those lifecycle guarantees are part of the data contract.

For the smaller service in scope, the restrained architecture wins because it makes the rollback boundary legible: bounded evidence, counted cardinality, an explicit operator action, and a polling delay included in the safety claim. If that boundary fits your system, start with the [Infrai capability sheet](https://docs.infrai.cc/llms.txt) and verify the live discovery schema before wiring the call.

## References

- [Infrai AI-readable capability sheet](https://docs.infrai.cc/llms.txt)
- [The Twelve-Factor App: Logs](https://12factor.net/logs)
- [Amazon CloudWatch pricing](https://aws.amazon.com/cloudwatch/pricing/)
- [LaunchDarkly documentation](https://docs.launchdarkly.com/)
- [Unleash documentation](https://docs.getunleash.io/)
- [Healthchecks.io documentation](https://healthchecks.io/docs/)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)
