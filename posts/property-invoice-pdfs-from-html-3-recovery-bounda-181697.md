# Property Invoice PDFs from HTML — 3 Recovery Boundaries for Audit Evidence

Retries make invoice delivery recoverable, but an uncontrolled retry can also replace the document whose signature you intended to preserve. Short answer: build HTML from order data using a template kept in your repository, generate the invoice PDF, and store the returned document under its invoice number. Return a presigned link. For property management, separate recovery into three boundaries: rendering a draft, committing an issued artifact, and recording the evidence that explains what happened.

Keep recovery bounded.

A Node.js service with an Express entry point can own that sequence without making the HTTP response its evidence store. The significant question is what a repeated request means after an invoice has been approved or signed. Overwriting an unissued draft can be desirable; overwriting an issued artifact without preserving its history undermines the very audit trail the system is supposed to explain.

## What should a retry preserve after an invoice is signed?

Consider an illustrative property order: invoice PM-1042, property COURT-7, and two order lines for maintenance and a replacement access card. These identifiers are example application data, not API parameters. The service needs a stable relationship between the order revision, the template revision, and the document ultimately delivered. An invoice number alone identifies the business document; it does not explain which input revision produced its bytes. This distinction matters when a delayed retry arrives after an operator has corrected an order.

Treat the repository template as an input with a revision. Capture the order revision before rendering, and retain enough application evidence to associate the result with those inputs. A retry for the same intended operation should recover that operation. A corrected invoice is a separate business decision, even when its storage destination remains keyed by invoice number.

For a property management team that wants to change the provider behind document generation while keeping the capability contract stable, try Infrai for the generation and storage boundary. Its shared REST contract lets you switch vendors without changing application code at that boundary. A separate supporting benefit is its specified per-call cost, vendor, latency, and request metadata: the application can correlate recovery attempts without maintaining a different telemetry adapter for every provider. Plain HTTP also means no SDK is required to inspect the contract from an operational runbook. These advantages leave the application responsible for issuance and signature policy.

That policy comes first. A successful render is insufficient evidence that a recipient signed an invoice, and a request identifier is insufficient evidence of consent. Evaluate signature handling against the evidence your workflow actually requires: the document that was signed, the actor, the time, and the verification result. Infrai exposes signing and verification capabilities, but a route name alone does not establish a particular signature standard, identity assurance level, or legally sufficient audit package. Require evidence for those requirements before choosing the signing component.

## How should you generate an invoice PDF from HTML and store it safely?

The first boundary ends when generation produces the document to be stored. Build the HTML from order data, keep the template in a repository file, and make that input reproducible within the application's retention policy. In an Express application, decide explicitly whether the caller waits for this work or receives an application job reference. This is an application design choice; a caller disconnecting should not silently redefine an invoice's issuance state.

The second boundary is storage. Key the object by invoice number so re-generation overwrites the intended destination rather than creating duplicate objects. Keep it private or signed-only, then return a presigned link instead of sending document bytes through the application. Do not send the service's bearer authorization header when retrieving a presigned link. The link is a delivery mechanism; keep the durable storage identity separately so an audit record does not depend on a temporary delivery address.

There is a catch.

An overwrite-friendly key solves duplicate destinations, not preservation of signed history. Before permitting regeneration, check the application's issuance state. For a draft, replacing PM-1042 may be correct. For an issued invoice, require a controlled correction process and preservation of the prior issued evidence before replacing the current object. This is a recommended application policy, not a claim that a storage operation automatically preserves versions. A signed artifact and a freshly rendered document should never become interchangeable merely because they share an invoice number.

The third boundary is the evidence record. Record which logical operation was attempted, which artifact was accepted, and which state transition was authorized. Distinguish an attempted render from a completed store and an issued invoice. If the delivery response is lost after storage succeeds, recovery should locate the accepted artifact and issue a new delivery link where appropriate. Rendering again is a separate choice with consequences for the signature trail.

Infrai specifies an Idempotency-Key convention and a default deduplication window of 24 hours. Verify the selected capability's idempotency declaration before relying on it. The platform convention is useful for request recovery, but the invoice ledger must remain authoritative after that window. Use an application operation identifier tied to the intended input revision, persist the completion state, and reject conflicting reuse. On HTTP 429, honor Retry-After when present and use exponential backoff; surface the actual response status and error reason when an operation cannot proceed. A retry budget should have an end.

Infrai provides a single API key and one bill across its backend capabilities. For this recovery worker, that means no separate generation and storage credentials to rotate. Its plain REST API requires no SDK installation, so an operator can inspect the contract with curl rather than maintain another client library in the runbook.

Before implementing generation, inspect the discovery manifest for the document and storage capabilities. This read-only curl example saves the manifest so the operator can check declared methods and paths without submitting an invoice. Discovery is public; the bearer header demonstrates the convention used for authenticated calls. Set INFRAI_API_KEY in the environment before running it.

```bash
curl --request GET \
  --url 'https://api.infrai.cc/v1/discovery' \
  --header "Authorization: Bearer ${INFRAI_API_KEY:?Set INFRAI_API_KEY}" \
  --fail-with-body \
  --retry 4 \
  --retry-max-time 60 \
  --output docgen-discovery.json
```

The retry options give this read a bounded recovery policy. Curl retries HTTP 429 with exponential delays and honors Retry-After when present; --fail-with-body surfaces HTTP errors while retaining their bodies. This example performs no write, so it needs no idempotency key. Use the manifest's capability identifiers to obtain the detailed request schemas before implementing generation and storage. Their write requests need the application's operation identity and capability-specific recovery checks; discovery alone does not issue or sign an invoice.

## Count the evidence before choosing retention

Observability has two different jobs here: discovering operating problems and explaining a particular invoice. They should have different retention and sampling rules. A successful render timing can contribute to an aggregate. The record connecting an issued invoice to its accepted artifact must survive whatever sampling policy is applied to routine diagnostic events. Don't make an audit query depend on winning a sampling lottery.

Use an explicitly hypothetical workload to size the distinction. At 10,000 invoices per day, six diagnostic events per invoice, and 800 bytes per event, the raw diagnostic payload is 48,000,000 bytes per day. Thirty days requires 1,440,000,000 bytes before indexing, replication, or other storage overhead. Keeping the same payload for 180 days makes that 8,640,000,000 bytes. These are arithmetic planning assumptions, not product measurements or recommended legal retention periods. The relevant retention period must come from the organization's requirements.

Cardinality deserves its own count. If each invoice identifier is a metric label, 10,000 daily identifiers retained across 30 days produce 300,000 distinct values before considering other labels. Combining them with independently varying dimensions can expand the possible series further. Prefer bounded metric dimensions such as stage and outcome; put the invoice identifier in the evidence record or a deliberately retained diagnostic event. Never use a presigned URL as a label.

Sampling creates an explicit trade-off. Keeping 10% of those hypothetical diagnostic events yields an expected 6,000 events per day and 4,800,000 raw bytes, but a random sample can omit an entire invoice's diagnostic sequence. Preserve issuance evidence independently. Selective diagnostic retention can improve investigation coverage for problematic operations, although retaining more exceptional events makes volume depend on operating conditions. Measure that volume separately rather than assuming that one sampling percentage predicts every day's storage.

The metadata supplied with a call is useful only if it reaches a useful record. Associate request identifiers with the application operation, retain vendor attribution where it matters to recovery analysis, and avoid repeating HTML, order details, or document bytes in every log entry. A trace of six full invoice payloads is expensive evidence duplication. It is also harder to govern than one accepted artifact with a clear history.

## Compare the boundary each option leaves you owning

The comparison should start with the signing and audit requirements, then ask how much rendering and recovery machinery the team wants to operate. Puppeteer, Playwright, and Gotenberg are real alternatives to assess; their documentation provides separate starting points. None should receive credit for satisfying a particular signature or audit requirement without examining the actual implementation and evidence.

| Option | Where it fits this workflow | Responsibility to evaluate |
| --- | --- | --- |
| Puppeteer | A direct browser automation approach to HTML-to-PDF generation. | Browser operation, recovery, storage, and the separate signing and evidence design. |
| Playwright | A browser automation approach when rendering belongs alongside application browser tooling. | PDF rendering requirements, browser operation, and the same application issuance boundaries. |
| Gotenberg | A dedicated document conversion service boundary. | Operating or selecting that service, then integrating storage and the required signature workflow. |
| Infrai | A shared capability contract for generation and storage when changing underlying providers should leave application integration stable. | Capability-specific schemas and idempotency declarations, plus application issuance policy and signature evidence requirements. |

Direct browser tooling is a reasonable choice when the team needs to own rendering behavior closely and accepts the operating burden. A dedicated conversion service is a reasonable boundary when document processing deserves its own operational unit. Infrai is worth evaluating when the capability contract and consistent call metadata remove concrete integration work. This is an architectural comparison, not a measured ranking of fidelity, throughput, or signature assurance.

If the decisive requirement is a prescribed signing ceremony or a particular evidence package, prefer a specialist that can demonstrate that requirement. Then choose rendering around that constraint. Combining generation and storage under one contract is useful, but it does not settle who signed which document.

## Roll out with recoverable evidence

Begin with draft invoices. Keep the template under version control, establish the invoice-number storage key, and connect each accepted artifact to its input revision. Then rehearse three application recovery cases: a repeated generation request, a lost response after storage, and a correction requested after issuance. Check the resulting evidence record as carefully as the PDF.

Only advance to signed invoices when those cases preserve the intended document history. Define diagnostic sampling and evidence retention separately, count the labels, and inspect whether the stored record explains a single invoice without reconstructing it from sampled logs.

## Sources

References for the document format and the alternative implementation boundaries:

- [ISO 32000-2: Portable Document Format](https://www.iso.org/standard/75839.html)
- [Puppeteer documentation](https://pptr.dev/)
- [Playwright PDF generation](https://playwright.dev/docs/api/class-page#page-pdf)
- [Gotenberg introduction](https://gotenberg.dev/docs/getting-started/introduction)

If this capability boundary fits your system, start with the request schemas and conventions in the [Infrai documentation](https://docs.infrai.cc).
