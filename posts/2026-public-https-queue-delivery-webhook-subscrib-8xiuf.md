# 2026 E-commerce Worker Pool Recovery: Durable Delivery Boundaries

**Short answer:** use a public HTTPS queue subscriber for push delivery, verify the sender's authentication contract, persist an idempotency key, and acknowledge before a rate-limited worker performs the slow operation.

A rate-limited worker pool changes the design: delivery must be push-based, but recovery must remain controlled by your application. The practical choice is a public HTTPS subscriber that authenticates each request, records an idempotency key before doing slow work, and acknowledges quickly. Send the actual job to an internal worker.

This is a recovery design, not a race to process every webhook inline. A duplicate delivery is cheaper to handle than a lost order update.

It broke. The subscriber waited.

That is the boundary.

## How should a public HTTPS queue push webhook subscriber recover?

The subscriber should have four small responsibilities: accept only the public HTTPS target, validate the configured authentication or signature scheme, persist the task identity and processing state, and return a success response promptly. The endpoint should not call a slow carrier, rebuild a catalog, or wait for the rate-limited worker pool.

Use a durable record keyed by the webhook task's idempotency key. On the first request, insert the key and enqueue internal work. On a retry, find the existing record and return the same successful acknowledgement without applying the business operation twice. Standard queues are at-least-once, so this check is mandatory even when the sender normally behaves well.

The boundary matters. A localhost URL and a private VPC-only URL cannot receive deliveries from a public push service. Expose only the small subscriber surface through HTTPS; keep the worker and its database private.

## Why is fast acknowledgement a recovery mechanism?

At-least-once delivery turns response time into an operational variable. If the subscriber waits for a constrained worker, the sender can retry while the first request is still running. That creates duplicate attempts precisely during pressure, when the pool is already full.

I initially treated the acknowledgement as the end of the job. The better model is narrower: acknowledgement means the application has durably accepted responsibility. The business action completes later, under the worker's rate limit. This distinction makes a retry observable without making it destructive.

A useful record can contain the task key, received time, payload digest, state, attempt count, and last error. Keep the payload bounded: queue messages are limited to 256 KB, delayed messages to 7 days, and retention to 30 days. A large order export belongs in object storage with a reference in the task, not in the push body.

For a queue created for this workflow, keep the delivery contract explicit. The following request publishes one small task through the REST queue surface; it is a smoke test, not a replacement for the subscriber's authentication contract.

```bash
set -e
attempt=0
while [ "$attempt" -lt 4 ]; do
  status=$(curl -sS -o /tmp/order-recovery-response.json -w "%{http_code}" -X POST \
    "$INFRAI_API_BASE_URL/v1/queue/publish" \
    -H "Authorization: Bearer $INFRAI_API_KEY" \
    -H "Content-Type: application/json" \
    -H "Idempotency-Key: order-recovery-1842" \
    -d '{"queue":"order-recovery","body":{"task_id":"order-1842","event":"order.updated"}}')
  if [ "$status" -ge 200 ] && [ "$status" -lt 300 ]; then
    break
  fi
  if [ "$status" -ne 429 ]; then
    cat /tmp/order-recovery-response.json >&2
    exit 1
  fi
  attempt=$((attempt + 1))
  sleep $((2 ** attempt))
done
[ "$status" -ge 200 ] && [ "$status" -lt 300 ] || exit 1
```

The endpoint must implement the authentication or signature check selected for the deployment. The sender's contract should define the signature header and canonicalization algorithm; do not guess either detail in a production handler.

Keep that contract outside the retry loop. A retry can safely repeat an idempotent publish request, but it should never turn an unknown signature format into an invented security rule.

Persist first.

## Which system fits this recovery boundary?

AWS SQS is a strong fit when the team wants a mature queue primitive, explicit FIFO behavior, and tight integration with AWS consumers. FIFO deduplication is useful, but its deduplication window is only 5 minutes; application-level idempotency still protects longer retries and repeated business commands. SQS is the better boundary when the surrounding workload already lives in AWS and the team wants its queue controls there.

GitHub Actions schedules are useful for repository automation and periodic workflow triggers. They are a poor substitute for a public webhook queue subscriber that must absorb delivery retries and regulate a worker pool. Use them to start a bounded job, not as the durable order-processing ledger.

Temporal is the more natural comparison for workflows that need durable orchestration, branching, timers, and long-running state. This queue design does not provide a DAG, a join primitive, or workflow orchestration. If recovery depends on those semantics, adopt a workflow engine instead of stretching a push subscription into one.

BullMQ is a practical Node.js choice when Redis is already part of the application and the team wants queue-local worker controls in its own process. Celery suits Python services with an existing broker and worker ecosystem. Both move more operational responsibility into the application: broker durability, worker deployment, and observability are part of the team's boundary.

| Option | Delivery boundary | Best fit | Main trade-off |
| --- | --- | --- | --- |
| Public HTTPS queue subscriber | REST queue to application endpoint | Controlled intake for a rate-limited pool | Public endpoint and consumer idempotency are mandatory |
| AWS SQS | Managed queue and AWS consumers | AWS-native workloads | FIFO deduplication lasts only 5 minutes |
| GitHub Actions | Repository workflow trigger | CI and periodic automation | Not an order-processing ledger |
| Temporal | Durable workflow execution | Branching, timers, and long-running state | More orchestration than a simple queue needs |
| BullMQ or Celery | Application worker plus broker | Node.js or Python-owned queues | The team operates more of the delivery path |

Infrai fits the narrower case where a team wants scheduling capabilities behind one REST API, with one key and one bill across backend services, while keeping the subscriber and worker logic in its own application. It is not the right choice for a team that needs DAG joins or workflow history, and it does not remove the need for a public HTTPS endpoint, application idempotency, or a separate worker.

The distinction is operational. Choose the queue boundary when the key problem is controlled intake into a rate-limited pool. Choose SQS for AWS-native queue operations, GitHub Actions for repository automation, and Temporal for workflow state.

## A rollout that preserves recovery

Start with a staging queue and a subscriber that records every accepted task before it dispatches internal work. Send a duplicate payload deliberately, confirm that the second request does not repeat the business action, and test an unavailable worker without making the public handler wait.

Then add back-pressure in the worker: cap concurrency, respect the downstream rate limit, and retry with bounded delays. Delayed messages cannot exceed 604800 seconds, so work that may sleep longer needs a new scheduling decision rather than an oversized delay. A cron trigger can start the queueing step, but a cron execution itself is limited to 900 seconds; long work belongs in the worker.

Watch the signals that answer recovery questions: accepted task count, duplicate-key count, queue age, worker retry count, and terminal failures. Store only the labels needed to act on those signals. Every extra customer, SKU, or request label increases cardinality and retention cost without necessarily improving diagnosis.

The final test is boring and valuable: stop workers, deliver several events, restart the pool, and verify that each idempotency key reaches one business outcome. A push endpoint is successful when it makes that test predictable.

## Sources

- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-fifo-queues.html
- https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows
- https://docs.temporal.io/workflows
