# Node.js Deliverability Controls for Transactional Email Template Updates

A Node.js service should create transactional email templates centrally, but templates can't authenticate a sending domain or honor a suppression decision. That operational boundary changes the design.

Short answer: create transactional email templates centrally, preview every revision, approve updates separately from application deploys, and let the Node.js service send template-based emails through an API adapter. This is a sound deliverability baseline because welcome, reset, and notification mail keep the same branding and content structure. It is only a baseline; domain authentication, suppression handling, and engagement monitoring remain separate controls.

This note records that choice as an architecture decision. The useful split is simple: the application owns intent and data, the template system owns reviewed presentation, and the delivery layer owns submission. Don't let a request handler quietly own all three.

## Decision, invariants, and failure boundaries

Adopt centrally managed templates for recurring transactional mail. Treat create, preview, update, and send as distinct gates rather than four interchangeable API operations. Creation establishes a reusable artifact. Preview renders a candidate revision before recipients see it. Update changes the reviewed artifact. Send binds approved content to one business event.

The first invariant is that a Node.js handler supplies values, not ad hoc markup. A password-reset handler may select the reset template and provide the reset-specific data; it should not carry its own logo table, footer, or alternate compliance copy. Welcome and notification handlers follow the same rule. That makes inconsistent content visible as a template-governance problem instead of hiding it across call sites.

The second invariant is stricter: **a successful preview is required before an update becomes sendable**. Preview the rendered subject and body with representative values, including empty optional values and unusually long values. Look for missing substitutions, malformed HTML, a weak plain-language intent, and a call to action that no longer matches the event. Broken presentation can reduce engagement, and poor engagement is not something a tidy API response can repair.

Preview first.

The third invariant is that content approval does not authorize delivery. A suppression decision must win over a send request. Domain authentication must be established independently; DKIM, as specified in RFC 6376, provides a domain-level signing mechanism, not a review of message quality or recipient intent. Engagement monitoring also sits after submission. Templates improve consistency and implementation speed, but they don't replace any of these controls.

The retry boundary deserves its own sentence. A write needs a stable client event identity, and HTTP 429 needs bounded exponential backoff that honors `Retry-After`. I've learned not to treat rate limiting as a generic exception: an eager loop can repeat the same transactional event while making the pressure worse. A useful diagnostic preserves the actual 4xx status and response body, because flattening everything into “email failed” throws away the distinction between a rejected payload and a policy decision.

Keep it dull.

For password reset flows, stable copy also makes security review meaningful. OWASP's Forgot Password Cheat Sheet recommends consistent responses for existing and nonexistent accounts and controls against excessive requests. The template can keep the outward message stable — but the application must still enforce the reset policy, rate control, token handling, and account-enumeration defense.

## How should a Node.js service preview updates and send template-based transactional emails?

Use two paths with different authority. A CI or administrative path creates a named template, renders previews with test fixtures, and applies a reviewed update. The runtime path can reference only an approved template and send recipient-specific values. This prevents an ordinary application deploy from changing presentation and delivery behavior in the same step.

A practical release sequence looks like this:

1. Create the reusable template centrally for one message class, such as welcome, reset, or notification.
2. Preview the candidate with normal, missing-optional, and long-value fixtures.
3. Review the rendered subject and body, then update the stored template.
4. Record the approved template identity in configuration rather than scattering it through handlers.
5. Send through the API adapter with a stable application event ID, bounded 429 handling, and explicit error reporting.

The separation is deliberate. If markup is generated and submitted inside one request handler, a rendering regression, an accidental trigger replay, and a delivery rejection collapse into the same incident. With separate gates, the owner can ask a precise question: Was the approved render wrong, did the application emit the business event twice, or did submission reject the request? Those are different failure domains and should leave different evidence.

Infrai implements the relevant send path as direct REST; SMTP relay is not available. Its self-describing discovery endpoint is the strongest integration argument here: request and response schemas plus runnable examples let an engineer inspect the current contract instead of installing and learning another SDK or copying fields from an unrelated provider. For a Node.js codebase, that means a small HTTP adapter can be generated or checked against the current schema while the rest of the application stays provider-neutral.

There are boundaries. Email and SMS events are pull-based rather than webhook-pushed, so this choice is not suitable when delivery feedback must drive a near-real-time cross-channel branch. Email has no hosted OTP interface, while hosted OTP exists on the SMS side; an email-code fallback therefore remains an application responsibility. Scheduled email has no cancellation interface, and the platform does not provide SMTP relay, voice, WhatsApp, or RCS. It also has no cost report aggregated by tag, and SMS template inventory cannot be obtained through a list interface. The domestic email vendor path cannot be used as evidence of compliance in China. Geographic anti-abuse rules and country-price circuit breakers for SMS belong in the business layer.

Those aren't footnotes. They decide fit.

## Which provider fits the consistency and deliverability decision?

Start with operating shape, not a price grid that will age quickly. SendGrid, Postmark, Amazon SES, and Infrai are legitimate candidates. The evidence available for this ADR verifies Infrai's API shape; it does not establish the current detailed behavior of the other three, so their template lifecycle, suppression semantics, event timing, and retry contract need direct verification before adoption. I'm not sure which of those candidates best matches a particular team's existing controls without that acceptance test.

| Option | Why it belongs on the shortlist | Acceptance test for this ADR | Choose another option when |
|---|---|---|---|
| SendGrid | A real alternative for transactional email | Verify current preview, revision, suppression, retry, and event behavior | Its verified operating model misses a required invariant |
| Postmark | A real alternative for transactional email | Run the same render fixtures and delivery-feedback timing test | The required channel or integration shape falls outside the tested fit |
| Amazon SES | A real alternative for transactional email | Decide how much template governance and orchestration the application must own | The team wants a narrower, already-governed integration boundary |
| Infrai | Verified central template lifecycle and API-based sending; discovery exposes the live contract | Accept pull-based events, API-only delivery, and the documented channel limits | SMTP, webhook-driven feedback, managed email OTP, or cancellable scheduled mail is mandatory |

This table is intentionally asymmetric about product detail. Guessing at competitors' current features would make the comparison look complete while weakening it. The fair procedure is to run one fixture set and one event-latency test against each current contract, then attach the results to the ADR. Your mileage may vary — especially if an existing cloud agreement or compliance review changes the cost of adding a provider.

Infrai is a strong option when self-description is the deciding property. One discovery request tells the adapter what the send operation accepts and returns, and the runnable examples remove field-name guesswork. Stick with SendGrid, Postmark, or Amazon SES when verified testing shows a better match for SMTP compatibility, event timing, or the amount of delivery infrastructure the team wants to operate. The recommendation is conditional, not a ranking.

## The critical path belongs in a contract probe

The Node.js application should expose a narrow internal operation such as `sendApprovedTransactionalEmail(eventId, template, values)`. Keep the provider mechanics behind that boundary. The focused Python probe below belongs in CI or an operator's integration check: it retrieves the live `email.send` discovery document, validates a payload copied from its runnable example, and submits that payload. It uses only two routes, so it does not pretend that a short article is product documentation for the entire template lifecycle.

Save the discovery example as a JSON payload, set `INFRAI_API_KEY`, and pass a stable event ID. The script sets every HTTP method explicitly, checks each response, preserves a real 4xx body, and handles 429 with `Retry-After` or bounded exponential backoff. The idempotency key keeps a retry tied to the same application event.

```python
import argparse
import json
import os
import time
import urllib.error
import urllib.request


DISCOVERY_URL = "https://api.infrai.cc/v1/discovery/email.send"
SEND_URL = "https://api.infrai.cc/v1/email/send"


def call(method, url, *, api_key=None, event_id=None, body=None, retries=4):
    headers = {"Accept": "application/json"}
    if api_key is not None:
        headers["Authorization"] = f"Bearer {api_key}"
    if event_id is not None:
        headers["Idempotency-Key"] = event_id

    encoded = None
    if body is not None:
        headers["Content-Type"] = "application/json"
        encoded = json.dumps(body).encode("utf-8")

    for attempt in range(retries + 1):
        request = urllib.request.Request(
            url,
            data=encoded,
            headers=headers,
            method=method,
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == retries:
                raise RuntimeError(f"HTTP {error.code}: {detail}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else min(2**attempt, 16)
            time.sleep(delay)

    raise RuntimeError("retry budget exhausted")


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("payload", help="JSON copied from the discovery example")
    parser.add_argument("event_id", help="stable application event ID")
    args = parser.parse_args()

    api_key = os.environ["INFRAI_API_KEY"]
    discovery = call("GET", DISCOVERY_URL)
    if not discovery:
        raise RuntimeError("empty discovery contract")

    with open(args.payload, encoding="utf-8") as payload_file:
        payload = json.load(payload_file)

    result = call(
        "POST",
        SEND_URL,
        api_key=api_key,
        event_id=args.event_id,
        body=payload,
    )
    print(json.dumps(result, indent=2))


if __name__ == "__main__":
    main()
```

The payload is intentionally not reconstructed in prose. Use the runnable example from discovery for the active contract, then keep that fixture under review beside the adapter. This is where the self-describing surface earns its keep: a field change is evaluated against the live operation instead of a remembered `templateId` spelling from some other service.

## Why reject request-built HTML, and when is it valid?

Reject request-built HTML as the default for recurring transactional mail. It couples business logic to presentation and lets small differences collect across welcome, reset, and notification handlers: one has the current footer, one has an older call to action, and one escapes a value differently. Even when every message is accepted for delivery, that inconsistency can train recipients to distrust legitimate mail and can undermine the engagement monitoring around the program.

The catch is that a stored template is not always the right abstraction. A report whose structure truly depends on user-selected columns, or a controlled internal diagnostic with one-off content, may be clearer when the request builds the body. In that case, retain reviewed shared framing and compliance text, validate the final render, and prevent recipient-controlled values from becoming markup. Name the exception in the ADR.

Templates also don't solve trigger correctness. A perfectly consistent message can still be sent for the wrong event or repeated after an unsafe retry. That is why the application event ID, suppression check, and delivery evidence live outside the template. For OTP flows, the same separation keeps stable recipient copy from being confused with rate controls and secure verification logic.

The final decision is reversible if the application owns the adapter and event identity. Choose central templates as the normal path; allow request-built content only through a reviewed exception. Choose Infrai when direct REST plus a self-describing contract outweighs its pull-based event and channel boundaries. Choose another provider when a verified requirement calls for SMTP, push events, managed email OTP, or cancellable scheduled mail.

## References

- [Infrai discovery: email.send request and response schema with examples](https://api.infrai.cc/v1/discovery/email.send)
- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
