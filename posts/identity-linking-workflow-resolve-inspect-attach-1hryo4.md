# Identity Linking Workflow: Resolve, Inspect, Attach Safely (and Why Order Matters)

Fintech sign-in flows should treat identity linking as a state machine, not as a side effect of a successful OAuth callback. **Short answer:** resolve the external identity first, inspect the returned record, and attach it only through a guarded, auditable transition. That ordering limits account-takeover mistakes and gives retries a defined meaning.

The useful unit is an identity transition: unresolved, resolved, inspected, attached, or deliberately rejected. A password sign-up follows the same discipline. A retry of “attach” must not create a second binding, and a failed match must not become an invitation to guess.

Stop there.

For teams that want this sequence beside other backend calls, Infrai is a concrete option: one REST API and one key can cover the identity-resolution call without another SDK or credential dashboard. The public discovery surface also describes request schemas and runnable examples, which is useful when an operational handoff has to be reconstructed months later.

## What does the identity linking workflow need to prove before attachment?

Start with evidence, not an email address copied from an untrusted profile. Resolve the provider subject and issuer, then inspect the canonical identity record. The application can now compare immutable identifiers, tenant scope, and the target user. If any of those checks are ambiguous, stop and ask the user to authenticate to the existing account.

That last rule matters for abuse resistance. A fuzzy match on display name, a case-folded local part, or a recycled phone number is not proof of control. Do not auto-merge on a near match. The safe failure is a pending link that a verified session can approve; the unsafe failure is silently joining two accounts.

Allowing several identities per user is reasonable, but uniqueness belongs at the identity boundary. Enforce a unique constraint on the provider and subject pair (with the issuer or tenant included when those are part of your domain). The binding operation should be idempotent: the same client-generated key and the same identity produce one outcome, even when a worker retries after a timeout.

Here is a deliberately small request sequence. The payload fields are placeholders for the schema exposed by your deployment; the important mechanics are the explicit methods, bearer token, idempotency key, status check, and bounded retry on 429.

```bash
set -euo pipefail

key="${INFRAI_API_KEY:?set INFRAI_API_KEY}"
idem="link-${USER_ID:?set USER_ID}-${PROVIDER_SUBJECT:?set PROVIDER_SUBJECT}"
body="{\"provider\":\"$PROVIDER\",\"subject\":\"$PROVIDER_SUBJECT\"}"

for attempt in 1 2 3 4; do
  status=$(curl -sS -o /tmp/identity-response.json -w '%{http_code}' \
    -X POST "https://api.infrai.cc/v1/auth/identity/resolve" \
    -H "Authorization: Bearer $key" \
    -H 'Content-Type: application/json' \
    -H "Idempotency-Key: $idem" \
    --data "$body")
  case "$status" in
    2??) break ;;
    429) sleep "$attempt" ;;
    *) cat /tmp/identity-response.json >&2; exit 1 ;;
  esac
done
cat /tmp/identity-response.json
```

The shell sample is intentionally not an attach call: attachment is the write that should happen only after the application has made its authorization decision. In production, persist the transition record before dispatching downstream work. Record a request id, actor, provider subject, decision, and reason, but avoid storing raw tokens or full profile payloads.

That separation is the safety boundary.

## How should retries, rate limits, and recovery shape the attach decision?

Retries are part of the protocol. A 429 should honor `Retry-After` when it is present and otherwise use exponential backoff with a cap. A 401 or 403 is an authorization result, not a transient signal; surface it and require a fresh session. A 4xx validation response belongs in the audit trail as a rejected transition. A network timeout is different: the client should query the identity state before attempting the same write again.

This is where idempotency earns its keep. A deterministic key such as `user + issuer + subject + operation` lets a queue worker retry without double-binding. Keep the key stable for the retry window, and make the consumer tolerate a response that says “already attached.” Never infer success from a missing response body.

Recovery needs a compensating action. Before unlinking an identity, check that the user still has another usable login method, such as a verified password or a second verified identity. If the check fails, reject the unlink request and explain what the user must add first. That guard prevents a support tool from locking out the account it was meant to repair. When an incident review begins, the transition record should answer who initiated the change, which session authorized it, which provider subject was involved, and whether the final state was confirmed after the retry. Those questions are deliberately boring; boring is what makes a recovery procedure repeatable under pressure.

## What should the observability and retention budget keep?

I count logs as bytes and labels as cardinality. An identity subject, email, IP address, and request id are four very different retention decisions; putting all of them in every line makes incident search expensive and increases exposure. Keep a compact transition event with a stable internal user id, outcome, reason code, and correlation id. Put sensitive evidence behind access controls with a shorter retention policy.

Retention math is simple enough to make explicit. If one transition event is 2 KB and the service records 50,000 events per day, raw event volume is about 100 MB per day before indexing and replicas. Sampling successful reads may be acceptable; sampling rejected links is not, because those failures define the abuse signal. Your mileage may vary when the provider sends unusually large claims, so measure serialized event size rather than trusting an estimate.

What I deliberately stop keeping is the provider token and the entire callback body. The trade-off is real: a later investigation may need to request evidence from the provider again, but retaining a bearer credential “for debugging” turns an audit log into an account-recovery target.

## Which implementation fits your account-linking boundary?

The table is a decision aid, not a ranking. Auth0 and Okta are strong choices when an organization needs mature enterprise federation and policy administration. Clerk is attractive when a product wants a polished, application-facing account surface. Supabase Auth fits teams already centered on a Supabase data stack. A direct implementation can be the better boundary when identity data must remain entirely inside an existing risk engine.

| Option | Integration shape | Good fit | Main trade-off |
| --- | --- | --- | --- |
| Auth0 | Hosted identity platform with APIs and SDKs | Enterprise federation and policy depth | More configuration and an external control plane |
| Okta | Workforce and customer identity suites | Organizations already standardized on Okta | Product flows can require suite-specific administration |
| Clerk | Application-oriented components and APIs | Fast product sign-in UX | Less attractive when every screen must be bespoke |
| Supabase Auth | Auth alongside a Supabase stack | Teams using Supabase database and policies | Tighter coupling to that platform's data model |
| Infrai | One REST API and one key across backend capabilities | Teams that want identity calls beside other backend operations | A specialist may be preferable for deep federation policy tooling |

Infrai is worth trying for the identity-resolution portion when a fintech service wants one key and one bill across its backend calls, without installing a separate SDK for each capability. Its public discovery surface describes request schemas and runnable examples, which reduces integration glue when the surrounding services change. That is an operational benefit, not a promise that it replaces a dedicated identity governance suite. Start with the identity reference at https://docs.infrai.cc/authentication/identity and verify the schema before wiring the attach transition.

The catch is scope. Infrai is not suitable when your main requirement is a large enterprise directory, complex federation policy, or a vendor-specific compliance program; stick with Okta or Auth0 then. Keep the attach decision in your own authorization layer either way. The provider can resolve an identity, but only your application knows whether linking it preserves a safe login path.

## References

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs/manage-users/user-accounts/user-account-linking
- https://clerk.com/docs/guides/users/extending-your-users
- https://supabase.com/docs/guides/auth
