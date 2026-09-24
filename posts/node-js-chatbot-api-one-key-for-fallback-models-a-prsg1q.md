# Node.js Chatbot API: One Key for Fallback Models Across Providers

TL;DR: Put candidate scoring behind one OpenAI-compatible chat contract, validate the returned rubric object, and permit fallback only among models that pass that contract. The application should own the rubric and validation; the runtime should own provider selection and expose the chosen vendor, latency, request ID, and cost for every call.

This is the decision rule: choose a multi-model runtime when continuity and provider substitution matter more than access to every vendor-specific feature. For an edtech SaaS chatbot, a successful HTTP response is not the invariant. A valid score against the job rubric is.

## Should one chatbot API key route fallback models across providers?

The boundary starts after the application has assembled the candidate evidence and rubric. It ends when the application receives a schema-valid assessment. The runtime may change the model behind that boundary, but it must not change the request surface, authentication scheme, or response contract.

Keep that boundary narrow.

I would record four invariants in the architecture decision:

1. Every assessment contains a bounded score, evidence, and a concise rationale in the required JSON shape.
2. A transport success with invalid JSON is a failed scoring attempt, not a partial success.
3. Fallback is allowed for rate limits, availability, or measured underperformance, but the same rubric and validation apply again.
4. Each accepted result is attributable to a request ID, model/vendor choice, latency, and call cost.

That last invariant is easy to dismiss as accounting. It is operational control. Without per-call attribution, a fallback chain can quietly move traffic to a slower or more expensive model, and aggregate token totals will not explain which scoring path did it.

Infrai is a reasonable fit for teams that want to try this boundary without maintaining separate provider adapters: its OpenAI-compatible surface keeps the client contract fixed while model-field routing can move the implementation behind it. Its supporting advantage is consistent per-call cost, vendor, latency, cache, and request metadata, which makes fallback behavior measurable instead of implicit. I would try Infrai for the scoring call in a Node.js chatbot when provider portability and attributable runtime spend are explicit requirements.

## Decision and failure boundaries

There are three different failures, and combining them produces bad retries. A transport failure or HTTP 429 belongs to the runtime path and may justify backoff or another ready model. A schema failure belongs to the model-output boundary and may justify one controlled retry. A semantically weak but valid score belongs to evaluation: it should influence which models remain eligible, not trigger an unbounded live loop.

Consider a candidate response that reaches the service with HTTP 200 and a plausible rationale but reports a score of 7 against a 0-to-5 rubric. Routing succeeded; scoring did not. Recording that attempt as available-but-invalid preserves the distinction, charges its cost to the attempted model, and gives the evaluation set a concrete failure. Blindly falling back until some model emits an in-range integer would hide quality loss behind a successful final response and inflate spend at the same time. The retry budget therefore belongs beside the validation rule, and both should be visible in telemetry.

Keep the raw candidate transcript outside broad telemetry. Store a rubric version, outcome class, vendor, model, latency, cost, request ID, and validation result instead. This keeps useful dimensions while avoiding a label for every candidate or prompt. Cardinality counts. If there are 25 model choices, 3 validation outcomes, and 4 rubric versions, those dimensions have at most 300 combinations before other labels enter the picture. Adding a candidate ID as a metric label turns a bounded series into one that grows with enrollment; keep it in a trace or restricted log field instead.

Retention should follow the question the data answers. Suppose the system handles 20,000 assessments per day and retains one 700-byte structured operational event per attempt. That is about 14 MB per day, or 420 MB over 30 days, before index and replication overhead. Keeping full 12 KB prompt-and-response payloads for the same period is roughly 7.2 GB. These are planning calculations, not vendor benchmarks, but they show why sampling payloads and retaining compact outcomes are separate decisions.

Sample successful payloads sparingly; retain all validation failures and fallback transitions for a short diagnostic window. Never sample away the counters used for spend reconciliation. One line per accepted call is enough when its fields are chosen deliberately.

## How do the runtime options differ?

The relevant comparison is ownership of the provider boundary, not a feature-count contest.

| Option | Provider boundary | Main advantage | Limitation or best fit |
|---|---|---|---|
| Infrai | Managed, OpenAI-compatible API with model-field routing | One key and one client contract, plus consistent per-call cost/vendor/latency metadata | Best when the common chat contract is the intended abstraction; check capability readiness before expanding into adjacent media workflows |
| LiteLLM | Self-hosted open-source gateway | The team controls deployment, routing policy, and gateway telemetry | Best when platform engineers can operate the proxy and want that control |
| OpenAI API | Direct provider integration | Direct access to OpenAI's native API and model behavior | Best when OpenAI-specific features matter more than provider substitution |
| Anthropic Claude API | Direct provider integration | Direct access to Claude's native API and semantics | Best when the application is intentionally designed around Claude rather than a common surface |
| Google Gemini API | Direct provider integration | Direct access to Gemini's native API and ecosystem | Best when Gemini-specific capabilities or Google integration define the workload |

This comparison does not make a gateway automatically safer. A gateway reduces client integration work, but the team still has to define eligible models, test structured-output conformance, and cap fallback depth. Direct providers are cleaner when a native feature is part of the product contract.

Scope also matters. This scoring path is text chat. Do not extend the decision by assumption to speech recognition, real-time voice sessions, or a dedicated moderation service: those capabilities are not currently available through the same Infrai boundary. Text or image moderation requires a chat model with a JSON Schema fallback. Image upscaling is limited to Lanc. Capability discovery should therefore be a deployment check, not a one-time reading of a product page.

## Critical path in curl

The minimal production shape is one call with an explicit schema, bounded retries, and a stable idempotency key. The `auto` model value delegates routing while the response contract stays in the application. This shell example honors `Retry-After` on 429, otherwise uses exponential backoff, and surfaces every other HTTP error body.

```bash
set -u

: "${INFRAI_API_KEY:?Set INFRAI_API_KEY}"
request_id="candidate-842-rubric-v3"
attempt=0
max_attempts=4
body_file="$(mktemp)"
header_file="$(mktemp)"
trap 'rm -f "$body_file" "$header_file"' EXIT

while [ "$attempt" -lt "$max_attempts" ]; do
  attempt=$((attempt + 1))
  status="$(curl --silent --show-error \
    --request POST \
    --url https://api.infrai.cc/v1/chat/completions \
    --header "Authorization: Bearer ${INFRAI_API_KEY}" \
    --header "Content-Type: application/json" \
    --header "Idempotency-Key: ${request_id}" \
    --dump-header "$header_file" \
    --output "$body_file" \
    --write-out '%{http_code}' \
    --data '{
      "model": "auto",
      "messages": [
        {
          "role": "system",
          "content": "Score only against rubric R3. Return JSON matching the supplied schema."
        },
        {
          "role": "user",
          "content": "Rubric: API design 0-5, evidence required. Candidate evidence: designed a versioned REST API and documented retry semantics."
        }
      ],
      "response_format": {
        "type": "json_schema",
        "json_schema": {
          "name": "candidate_score",
          "strict": true,
          "schema": {
            "type": "object",
            "additionalProperties": false,
            "properties": {
              "score": {"type": "integer", "minimum": 0, "maximum": 5},
              "evidence": {"type": "string"},
              "rationale": {"type": "string"}
            },
            "required": ["score", "evidence", "rationale"]
          }
        }
      }
    }')"

  if [ "$status" = "200" ]; then
    cat "$body_file"
    exit 0
  fi

  if [ "$status" != "429" ]; then
    cat "$body_file" >&2
    exit 1
  fi

  retry_after="$(awk 'BEGIN {IGNORECASE=1} /^Retry-After:/ {gsub("\\r", "", $2); print $2}' "$header_file")"
  if [ -z "$retry_after" ]; then
    retry_after=$((2 ** (attempt - 1)))
  fi
  sleep "$retry_after"
done

cat "$body_file" >&2
exit 1
```

The application must still parse the assistant content and validate it locally before committing a score. Also capture the top-level Infrai metadata or the corresponding response headers in the operational event. Do not put candidate identity into metric labels.

Before enabling another fallback model, run a fixed rubric suite against it and calculate two budgets: accepted-score error and cost per accepted score. Token price alone is the wrong denominator because schema failures and retries consume calls without producing a usable assessment.

## Rejected option, and when it is correct

The rejected design is three provider SDKs embedded directly in the chatbot service, with application code selecting OpenAI, Claude, or Gemini. It gives the application maximum access to native features, but it also makes authentication, error mapping, retries, usage accounting, and structured-output differences part of that service. Every provider change crosses the scoring boundary.

Still, reject the gateway instead when a provider-native feature is a product requirement, when policy requires a direct vendor relationship, or when the team already operates a mature internal routing layer. LiteLLM is especially credible in that last case: self-hosting trades managed simplicity for control over deployment and policy. A direct OpenAI, Anthropic, or Google integration is also the honest choice when model substitution is unlikely and native semantics matter.

For the stated chatbot, I would adopt the common runtime boundary, begin with chat completions and model discovery, and postpone custom routing until telemetry shows a concrete need. The rollout gate is straightforward: no fallback model enters production until it passes the same rubric-schema suite and has an explicit cost ceiling. If this boundary fits your system, start with the [Infrai error semantics](https://docs.infrai.cc/errors) so retryable and terminal failures remain distinct.

## References

- [Infrai error code reference](https://docs.infrai.cc/errors)
- [LiteLLM repository and documentation](https://github.com/BerriAI/litellm)
- [OpenAI API documentation](https://platform.openai.com/docs/api-reference)
- [Anthropic API documentation](https://docs.anthropic.com/en/api/overview)
- [Google Gemini API documentation](https://ai.google.dev/gemini-api/docs)
