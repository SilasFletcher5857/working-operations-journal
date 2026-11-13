# Healthtech Summary JSON API: Why I Chose Structured Output With Bounded Telemetry

Choose a structured summary JSON output API with bounded telemetry for questions over a private healthtech knowledge base. The deciding constraint is whether an answer preserves the retrieved evidence within the response deadline. A valid JSON object alone cannot establish that.

Short answer: for a Node.js backend, select an output API by testing schema enforcement, evidence fidelity, and end-to-end latency together. Require a title, bullets, and key takeaways; validate the completed response before releasing it. Keep routine telemetry small enough to retain useful operational evidence without routinely copying private source text into logs.

This architecture decision record describes a proposed design. My choice is an engineering judgment, not a report of a production deployment or measured performance. The workload is administrative knowledge retrieval in healthtech: staff ask about internal procedures, and the service summarizes authorized excerpts. Clinical advice is outside this example.

## How should a structured summary JSON API balance output quality and latency?

The contract has three required fields: `title`, `bullets`, and `key_takeaways`. In this proposed schema, the title is a nonempty string, and both collections contain nonempty strings. Extra properties are rejected. Those requirements make the result predictable for the application that renders it; they do not establish that the content follows the source. JSON Schema provides vocabulary for expressing these structural constraints.

Preserve the qualifiers.

A hypothetical policy saying that an exception requires approval must not become a takeaway saying the exception is always permitted. Both statements can occupy exactly the same JSON field and pass exactly the same string validation. For this workload, a quality test therefore needs the retrieved passage, the question, and the proposed summary together. It should examine whether the answer preserves conditions, distinguishes missing information from a negative answer, and stays within the authorized material.

The invariants are authorization before retrieval, evidence-bound generation, local validation before release, and a deadline that covers the entire request. Authorization failures stop the request. Empty or insufficient evidence produces an explicit insufficient-evidence outcome in the application envelope. A refused, truncated, or invalid generation produces no successful summary. These are proposed application behaviors, not claims about a particular provider.

The failure boundary matters. The Node.js service owns the public response contract even if another component constrains generation. It should check the completed object before exposing it downstream. A partial response is still partial; a plausible title does not justify releasing unvalidated bullets.

## Decision: retain operational evidence with a bounded vocabulary

The telemetry design records a schema revision, a bounded outcome category, latency measurements, and size counters. Suggested outcome categories are `accepted`, `insufficient_evidence`, `invalid_output`, and `deadline_exceeded`. An authorization rejection belongs to the access-control boundary and should not be counted as a model quality failure. Raw questions, retrieved excerpts, and generated summaries are excluded from routine telemetry in this proposal.

Count cardinality before adding labels.

Suppose an illustrative deployment has 4 outcome values, 3 schema revisions, and 2 deployment environments. Their Cartesian product permits 24 label combinations for one metric before additional dimensions. Adding a distinct request identifier as a label destroys that useful bound. Put correlation identifiers in appropriately controlled event records when needed, and keep them out of metric dimensions. The numbers here describe a hypothetical label budget, not observed traffic. Retention needs equally explicit arithmetic. For an illustrative volume of 100,000 requests per day, one 600-byte event per request produces 60,000,000 raw bytes daily. Fourteen days would retain 840,000,000 raw bytes. These figures exclude indexing, replication, compression, and storage overhead; they are a payload budget rather than a storage forecast. Doubling the event size doubles this raw volume without making the summary more accurate. Adding another unbounded dimension can also defeat the label budget even when event retention is short. These are separate controls: limiting how long records remain does not limit how many distinct metric series a request can create.

Budget both dimensions.

I chose bounded telemetry because routine payload retention would expand the private-data footprint without establishing answer quality. I accept fewer routine details, with a separate, access-controlled path for deliberate quality review. A small, approved evaluation set can preserve source-to-summary comparisons without turning every production request into a retained document. Its retention and access rules should be defined independently of operational metrics.

Sampling changes the evidence available to investigators. Uniformly sampling one request in ten would, in expectation, retain 10% of request events; it does not ensure coverage of rare failures. Retaining bounded failure events at a higher rate can improve diagnosis, but the resulting sample cannot directly estimate population failure rates without accounting for the sampling policy. Keep aggregate request and outcome counts independent of sampled diagnostic records. Otherwise a change in sampling can masquerade as a quality regression.

## Compare the paths on evidence and delay

The following options describe application architectures. They make no assumptions about a commercial API's supported schema vocabulary or latency. Those details require verification against the candidate interface and measurement on the actual workload.

| Output path | Evidence and structure boundary | Latency trade-off | Telemetry consequence |
| --- | --- | --- | --- |
| Constrained generation, then local validation | Check the required fields locally; assess evidence fidelity separately | One generation attempt on the normal path; constraint overhead must be measured | Small outcome vocabulary and explicit validation failures |
| Prompt for JSON, then validate and retry | Accept only a validated object; another attempt may still misstate the evidence | Retries consume the remaining deadline | Record attempt counts so latency and failures remain interpretable |
| Stream fragments directly into the interface | Final validity remains unknown while fragments arrive | Earlier visible text, with a separate time to accepted result | Distinguish first fragment from completed, accepted answer |

I choose the first path when the candidate supports the required constraints and meets the evidence tests within the workload's latency budget. That is a conditional decision. A schema feature that the candidate does not support is a selection boundary, even if another interface accepts the same schema document.

Measure the delay users actually experience: retrieval, generation, validation, and any permitted retry through to acceptance.

Also inspect the tail.

A favorable average can hide requests that spend their entire deadline waiting for a repair attempt, and a fast first fragment can precede a final object that the application cannot accept.

## The critical path carries the contract

This curl request illustrates a proposed local application interface, not an existing provider endpoint. It assumes a service listening on port 8080 that implements the contract below. The deliberately fictional excerpt concerns an administrative procedure and contains no patient information.

```bash
curl --fail-with-body --silent --show-error \
  --max-time 10 \
  --request POST 'http://localhost:8080/summaries' \
  --header 'Content-Type: application/json' \
  --data-binary '{
    "question": "What does the fictional access-review procedure require?",
    "authorized_excerpt": "Fictional policy: A reviewer must approve an access exception before it takes effect.",
    "schema_revision": "summary-v1",
    "output_schema": {
      "type": "object",
      "properties": {
        "title": {"type": "string", "minLength": 1},
        "bullets": {
          "type": "array",
          "minItems": 1,
          "items": {"type": "string", "minLength": 1}
        },
        "key_takeaways": {
          "type": "array",
          "minItems": 1,
          "items": {"type": "string", "minLength": 1}
        }
      },
      "required": ["title", "bullets", "key_takeaways"],
      "additionalProperties": false
    }
  }'
```

Here the client timeout is 10 seconds, an illustrative test setting rather than a recommended service objective. The service needs its own deadline propagation and cancellation policy. A client stopping its wait does not establish that all downstream work has stopped.

In production, the backend should obtain authorized excerpts through its retrieval boundary; an arbitrary caller-supplied excerpt is not an authorization mechanism. Likewise, the server should select a reviewed schema revision rather than trust arbitrary client constraints. The example exposes both inputs to make this test contract inspectable.

For the Node.js implementation, the critical sequence is authorized retrieval, generation under the selected contract, parsing, schema validation, and acceptance. Instrument the stages with bounded names. Keep validation errors useful but redact payload values: an error message that embeds a rejected string can quietly reintroduce private content into otherwise minimal logs.

Before deployment, exercise empty retrieval, unsupported schema requirements, invalid output, and deadline exhaustion. Include a semantically wrong but structurally valid answer in the evaluation set. Release gates should consider evidence fidelity and latency together, using the same cases across candidate paths. Define rollback against those measurements, and retain the schema revision so a changed contract does not get mistaken for changed model behavior.

## Rejected option: repair every failed summary

An unconditional repair loop is rejected for this workload. It adds work precisely when the normal path has already failed, and another syntactically valid answer can still discard the condition that made the source useful. A repair budget must therefore include both remaining time and another evidence check. Cheap-looking individual attempts are a poor basis for this decision.

A bounded repair attempt has a valid use case in asynchronous administrative summarization, where a later accepted result is useful and the workflow can wait. Record the original failure category and the attempt count. Do not let eventual success erase the operational evidence of repeated invalid outputs.

For interactive private-knowledge answers, my decision remains conditional on evidence fidelity within the deadline. Keep the title, bullets, and key takeaways contract small; retain enough telemetry to distinguish quality loss from waiting; expand diagnostic detail only when a specific investigation justifies its retention. The best fit is the path that meets those conditions on the evaluation set.

## References

- JSON Schema validation specification: https://json-schema.org/draft/2020-12/json-schema-validation
- Prometheus guidance on metric labels and cardinality: https://prometheus.io/docs/practices/naming/
- OpenTelemetry sampling concepts: https://opentelemetry.io/docs/concepts/sampling/
- OWASP Logging Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
- curl command-line reference: https://curl.se/docs/manpage.html
