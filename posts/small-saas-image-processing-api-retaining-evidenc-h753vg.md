# Small SaaS Image Processing API: Retaining Evidence for Promo Moderation

The bill for a small SaaS that generates short promo videos from prompts has several terms: generation, image derivatives, delivery, and the records retained to explain moderation decisions. Before comparing image processing APIs, count the objects and records that each workflow creates. A convenient transformation interface can conceal a growing retention obligation.

Short answer: choose between URL-based transformations and explicit server-side calls by who controls the derivative set. URL transformations are convenient, but each variant becomes a cache entry with a cost. Explicit calls let the backend fix the derivative set. For promo media, make moderation coverage the release criterion, then retain enough evidence to associate a decision with the exact asset version. Neither transformation style establishes that coverage by itself.

Consider an illustrative workload, not a vendor benchmark: 10,000 promo jobs per day, each producing four image derivatives of 100 KB. That creates 4 GB of new derivative data daily, or 120 GB across a 30-day window, before replication, originals, or video files. If each job also produces one 2 KB moderation record, the corresponding records occupy 600 MB over that window. Among these two retention terms, derivatives dominate by a factor of 200. The total production bill could have a different dominant term; generation and delivery need their own measurements.

Count the variants.

## What must moderation cover before a promo can ship?

A prompt-to-video workflow needs a coverage definition before it needs a vendor preference. Treat prompt text, any submitted images, image derivatives selected for publication, and the generated video as separate review targets in the application design. These are requirements to verify, not claims that any candidate provides every check. Approval of an input image cannot establish approval of a later video, and an image transformation API alone does not answer the video moderation question.

This distinction matters when the original buying question is Cloudinary versus imgix versus ImageKit, or a simpler alternative for a small SaaS. That question concerns image processing. A generated promo also crosses a publication boundary. The useful comparison asks which image workflow produces a bounded set of publishable assets and whether the selected moderation arrangement can review those assets at the required stages. A provider may be suitable for the image portion while a separate service and application policy determine video release.

Require evidence of coverage.

For an engineering acceptance test, create a matrix with review targets on one axis and lifecycle stages on the other. Mark a cell covered only after checking the relevant interface, testing its output, and deciding what result permits publication. Leave unknown cells unknown. Do not translate an image approval into a video approval merely because both assets belong to the same job. A practical publication rule is to hold the output until all required decisions reference the asset versions actually being released.

User avatars, another common small-SaaS image workload, illustrate the narrower boundary: a controlled set of image sizes can be the entire derivative problem. Promo generation adds a changing output that needs its own decision. Keep that extra obligation visible when transferring an avatar architecture into a media pipeline.

## Reduce the dominant retained term before sampling records

In the illustrative workload, reducing the approved derivative set from four sizes to two changes daily derivative storage from 4 GB to 2 GB. Across 30 days, the retained derivative payload falls from 120 GB to 60 GB. This is arithmetic on stated assumptions, not a measured saving or a claim about a provider's invoice. It also says nothing about video storage, requests, or transfer.

The change is architectural: have the backend approve a finite set of derivative recipes and publish references to those outputs. Explicit server-side transformation calls fit this approach because the application chooses what to create. URL-based transformations remain convenient when clients need flexible presentation, but unconstrained client parameters invite unbounded variant counts. If predictable cost is the objective, store the derivatives yourself under either approach and define their lifecycle explicitly.

That control has a cost. A new placement may need a size that was deliberately excluded. The application must either accept the existing rendition or introduce another approved recipe and review the consequences. A fixed set trades some presentation flexibility for countable objects and a smaller review surface. It does not make moderation automatic.

Do the storage arithmetic before deleting decision evidence. In this example, even eliminating all 600 MB of retained moderation records would barely move the larger image term, while removing the association between an asset and its release decision could make an investigation much harder.

## How should a small SaaS compare image processing API alternatives?

Cloudinary, imgix, and ImageKit are real candidates for the image processing decision. Their documentation belongs in the evaluation, but an image product name is insufficient evidence of moderation across a generated-video workflow. The distinctions below are questions for acceptance testing; they are not an unsupported ranking of vendor coverage.

| Candidate | Relevant evaluation surface | Limitation to resolve before selection |
| --- | --- | --- |
| Cloudinary | Use its image transformation documentation to evaluate the required derivative recipes and the boundary between transformation and delivery. | Confirm the exact moderation stages needed for the promo workflow; image transformation documentation alone cannot establish them. |
| imgix | Use its rendering API documentation to evaluate how application inputs map to requested variants. | Determine how the application will bound the accepted variant set and supply the required moderation decisions. |
| ImageKit | Use its image transformation documentation to evaluate the rendition requirements for each placement. | Verify where asset-version review and final video approval belong in the application and service arrangement. |

These are different evaluation entry points, not claims that any one candidate is incapable of the other workflows. Compare the same small set of required renditions and review targets across all three. Record the results rather than inferring superiority from the breadth of a marketing page. Choose URL transformations when flexible rendition selection is a requirement and the application can bound it; choose explicit calls when a fixed derivative set is the stronger requirement.

Infrai offers a self-describing REST API under one key and one bill, with no SDK to install. Its public discovery surface supplies request schemas, response schemas, and runnable examples, so evaluating an image capability starts by reading its definition. This makes it an option for the explicit-call side of the comparison. The consistent credential boundary is useful when a media workflow spans several backend services. Those are reasons to evaluate its integration model; they do not establish end-to-end moderation coverage for prompt-generated promos.

Inspect before integrating.

The following read-only discovery request retrieves the capability manifest for that evaluation. Set `INFRAI_API_ORIGIN` to the provider's HTTPS API origin, without a version suffix, using your deployment configuration, and supply your key through `INFRAI_API_KEY`. Discovery is public and does not require authentication; this request includes the configured credential to demonstrate the convention used by authenticated calls. The explicit GET cannot create duplicate derivatives, and curl surfaces HTTP error responses rather than treating them as successful schema retrieval. Its bounded retry behavior backs off and honors Retry-After for rate limiting. Use the returned capability definitions to find the request and response schemas before writing transformation calls; this example deliberately retrieves definitions rather than inventing a resize request body.

```bash
curl --request GET \
  --header "Authorization: Bearer ${INFRAI_API_KEY:?Set the API key}" \
  --fail-with-body \
  --retry 4 \
  --retry-max-time 60 \
  --connect-timeout 10 \
  --max-time 30 \
  --url "${INFRAI_API_ORIGIN:?Set the HTTPS API origin}/v1/discovery"
```

No cheapest-provider claim is needed to make this decision. The durable distinction is how many variants the application permits, who retains them, and what evidence authorizes their publication.

## Cardinality and sampling need different rules

Counting bytes is only half the observability problem. Suppose the application proposes metric labels for 10,000 job identifiers, four derivative recipes, and three review states. The Cartesian product permits 120,000 combinations. Actual observed combinations depend on events, but the estimate is enough to question whether job identifiers belong in aggregate metric labels. This is a design calculation, not an observation about a vendor's monitoring system.

Keep identifiers in controlled records used for investigation.

Use bounded categories for aggregates: recipe, review state, and an application-defined policy version, with an explicit limit on how many policy versions remain active. A field that is useful for lookup does not automatically belong in every metric label set. Adding a prompt string as a label would also undermine the attempt to bound distinct values. The trade-off is less convenient aggregate drill-down because individual jobs must be investigated through their retained records, but it prevents a unique job identifier from multiplying every otherwise bounded combination of labels. That is a worthwhile boundary when the purpose of the aggregate is to describe workload behavior rather than preserve an index of every execution.

Sample diagnostic detail selectively. Under an illustrative 10% sampling rule, 10,000 successful jobs would yield about 1,000 retained diagnostic records in expectation, but sampling the authoritative release decisions the same way would leave most releases without their own decision record. These are different evidence classes. Preserve the minimum release record for every published asset; apply sampling to verbose diagnostics when the loss of detail is acceptable.

The trade-off is explicit: sampled diagnostics make some successful executions impossible to reconstruct in full. Keeping decision records does not recover the discarded detail. It preserves a narrower answer about what was approved, under which application policy, and for which asset version.

An application record should carry only fields justified by that purpose. A proposed minimum is an asset-version reference, the relevant review target, a decision, a policy version, and a decision time. This is an application schema recommendation, not a claim about fields returned by any provider. Avoid retaining prompt bodies and full generated payloads in every routine log entry merely because they are available.

## What to stop keeping, and what that costs

Define separate retention periods for publishable derivatives, release decisions, and verbose diagnostics. The 30-day window above is an illustrative calculation, not a compliance recommendation. Choose actual durations from the product's investigation and deletion requirements, and count the consequences before implementing expiration.

Deliberately stop keeping unapproved rendition combinations, duplicate payload bodies in routine diagnostics, and unsampled successful execution detail after its diagnostic window. Expire unused derivatives according to their application lifecycle. Preserve the release evidence needed for the chosen investigation window, with access controls appropriate to the data it references.

There is a real loss.

Once an expired image or video is gone, a decision record cannot reproduce its visual contents. An asset reference is not the asset, and a retained approval is not proof that today's policy would approve the same material. When something goes wrong, the retained record may establish which version was released and which policy authorized it while leaving the exact visual reconstruction unavailable. Accept that boundary deliberately, or retain selected assets longer with an explicit purpose and owner.

## Further reading

References for evaluating the image processing boundary and format choices; these sources do not, by themselves, certify moderation coverage for the complete promo workflow:

- [Cloudinary image transformations](https://cloudinary.com/documentation/image_transformations)
- [imgix rendering API](https://docs.imgix.com/apis/rendering)
- [ImageKit image transformations](https://imagekit.io/docs/image-transformation)
- [MDN image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
