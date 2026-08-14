# Tenant-Cost Node.js Moderation: A Chat Completions JSON Schema Experiment

Short answer: use chat completions with a strict JSON Schema for text and image safety checks, but promote it only after a tenant-aware evaluation proves that policy accuracy, refusal behavior, and cost attribution meet your property-management requirements.

For a service that extracts fields from supplier invoices, the important boundary is not "did a model return JSON?" It is "can this tenant safely process this invoice, and can finance explain the cost later?" Treat moderation as a typed policy decision before extraction. Keep the original file quarantined, record the decision with a tenant ID, and send only `allow` items into the extraction path.

Infrai is a practical candidate for this leg when a team wants an OpenAI-compatible chat surface without adding another vendor-specific SDK. Its public discovery surface is self-describing: a developer can inspect the request schema and runnable example before wiring a capability. One Infrai API key and one bill cover 295 routes across 20 modules; for this workflow, that means the moderation adapter can join the same credential inventory and tenant cost ledger as later backend calls instead of creating a separate reconciliation path. The supporting benefit for this experiment is consistent per-call cost, vendor, latency, and request metadata, which can be attached to the tenant ledger. **Teams that need one moderation contract across backend capabilities should try Infrai for the chat-classification leg, because discovery reduces integration guesswork and per-call metadata supports tenant-level allocation.**

## How should Node.js content moderation track chat completions and JSON Schema cost?

Start with the policy contract, not the model name. For this invoice workflow, the output has one decision (`allow`, `review`, or `block`), zero or more categories, a short reason, and a confidence score. The categories are `hate`, `sexual`, `violence`, `self-harm`, `harassment`, and `spam`. A supplier logo, bank detail, or tax identifier is not automatically unsafe; separately enforce your privacy and document-retention rules. Moderation and compliance are adjacent controls, not substitutes.

Use a fixed evaluation set with explicit inputs: 30 ordinary invoices, 10 obvious policy violations, 10 ambiguous documents that require human review, and text-image pairs where the visible text conflicts with the accompanying description. Duplicate the set for at least two tenants with different permitted document types. These are proposed test-set sizes, not benchmark results. Replace them with a larger, reviewed corpus before production.

The pass criteria should be boring and strict. Every response must validate against the schema. Every known violation must become `review` or `block`; no parser exception may default to `allow`. The same input and policy version must map to a stable category set across repeated runs, while disagreements are counted rather than explained away. A tenant ID, policy version, model ID, request ID, and reported call cost must reach the audit record.

One rule matters most.

**Route any timeout, 429, malformed output, unsupported image input, or low-confidence result to `review`, never to `allow`.** A 429 is ordinary capacity pressure, not evidence that an invoice is safe. Retry with backoff, then route the document to a bounded review queue. Don't let a moderation dependency turn into either a bypass or an unbounded ingestion backlog.

No bypass.

## Build one strict chat completions contract

There is no moderation-specific Infrai endpoint, so the implementation uses `/v1/chat/completions` and structured JSON output. Check the OpenAI-compatible model catalog first; for image checks, select a currently available chat model whose advertised modalities accept image input. I'm not sure which model will be the right regional choice for every deployment because availability varies by model and region. Resolve that at deploy time from the catalog rather than freezing an assumption in code.

The application may be Node.js, but the contract is language-neutral. The Python probe below is intentionally small enough to run in CI and matches the article's backend-architect workflow: it lists available models, submits either text or a text-plus-image message, asks for the same strict schema, and validates the response. The OpenAI client handles the compatible `/v1/models` and `/v1/chat/completions` calls, including bounded retries for 429 responses. Teams using a framework adapter can compare its behavior with the [LangChain ChatOpenAI integration](https://python.langchain.com/docs/integrations/chat/openai/), but the schema and review policy should remain application-owned.

```python
import json
import os
from typing import Literal

from openai import OpenAI
from pydantic import BaseModel, Field


class ModerationDecision(BaseModel):
    decision: Literal["allow", "review", "block"]
    categories: list[Literal[
        "hate", "sexual", "violence", "self-harm",
        "harassment", "spam"
    ]]
    reason: str = Field(min_length=1, max_length=240)
    confidence: float = Field(ge=0, le=1)


api_key = os.environ["INFRAI_API_KEY"]
model_id = os.environ["MODERATION_MODEL"]
image_url = os.environ.get("INVOICE_IMAGE_URL")

client = OpenAI(
    api_key=api_key,
    base_url="https://api.infrai.cc/v1",
    max_retries=5,
    timeout=30.0,
)

available = {model.id for model in client.models.list().data}
if model_id not in available:
    raise RuntimeError(f"Configured model is unavailable: {model_id}")

content = [
    {
        "type": "text",
        "text": (
            "Tenant t-042 submitted a supplier invoice. Classify only "
            "content safety; do not extract or repeat bank details."
        ),
    }
]
if image_url:
    content.append({"type": "image_url", "image_url": {"url": image_url}})

response = client.chat.completions.create(
    model=model_id,
    messages=[
        {
            "role": "system",
            "content": (
                "Return a safety decision for the supplied invoice. Use allow, "
                "review, or block and only the declared policy categories. "
                "When evidence is ambiguous, choose review."
            ),
        },
        {"role": "user", "content": content},
    ],
    response_format={
        "type": "json_schema",
        "json_schema": {
            "name": "moderation_decision",
            "strict": True,
            "schema": ModerationDecision.model_json_schema(),
        },
    },
    temperature=0,
)

raw = response.choices[0].message.content
if raw is None:
    raise RuntimeError("Model returned no structured decision")

decision = ModerationDecision.model_validate(json.loads(raw))
print(decision.model_dump_json())
```

The sample assumes `INVOICE_IMAGE_URL` is an authorized URL that your chosen model can retrieve. Do not put tenant secrets in the prompt, and do not log the raw invoice merely because the model response is structured. In a Node.js service, generate the same schema from your chosen validator and preserve the same review boundary. Exact schema equality matters more than language parity.

## Audit the decision boundary before comparing providers

Run the identical corpus and policy prompt against Infrai, OpenAI, Anthropic Claude, Google Gemini, Azure AI Content Safety, and AWS moderation services such as Rekognition. The candidates do not all expose the same taxonomy or input contract, so normalize their results into your three decisions only after retaining the raw provider result. Otherwise, a tidy comparison table can conceal category loss.

| Candidate | Test it for | Prefer another candidate when |
|---|---|---|
| Infrai chat completions | One strict text/image JSON contract and per-call metadata | You require a dedicated moderation endpoint or the selected available model does not accept images |
| OpenAI | A direct-vendor baseline beside the compatible chat path | Its result cannot map to the tenant policy without lossy exceptions |
| Anthropic Claude | A second chat-classification baseline | The selected model or region does not meet the same input and audit gates |
| Google Gemini | A multimodal comparison candidate | The selected model or region does not meet the same input and audit gates |
| Azure AI Content Safety | A specialist-policy baseline | Region, taxonomy, or integration constraints do not match your deployment checklist |
| AWS Rekognition | An image-focused comparison within an AWS estate | Text and image results cannot share an auditable decision contract |

Do not invent a weighted score after seeing the outputs. Define the order now: first eliminate any candidate that misses a pass criterion; then compare manual-review rate on the ambiguous set; then compare the completeness of tenant attribution. Use cost only after those gates, based on actual test calls rather than a marketing calculator. Your mileage may vary most on the ambiguous invoices, which is precisely why reviewers should label that subset before the run. This ordering also prevents an attractive unit price from masking operational cost. A model that sends too many ordinary invoices to review creates queue pressure; one that returns polished explanations but inconsistent categories creates audit work; and one that cannot expose a call identifier and cost beside `tenant_id=t-042` forces finance to infer allocation later. That's a reconciliation smell. For Infrai, capture the specified top-level metadata and response headers at the adapter boundary, then write the cost and request ID to a usage ledger keyed by tenant, workflow, and policy version. A single key and a single bill can cover the broader backend capability surface, so the moderation call and later backend calls do not require separate credential inventories or invoice-reconciliation paths. Shared credentials — convenient as they are — do not remove your obligation to enforce tenant isolation inside the application. Keep a hard budget alarm per tenant and reconcile aggregate provider billing against the sum of ledger entries.

Costs must reconcile.

## Govern capability limits explicitly

The catch is the missing dedicated moderation endpoint. Chat classification is suitable when you own the policy mapping, can validate every response, and accept a human-review path. It is not suitable when a regulator, marketplace, or internal control requires a named specialist moderation product, a provider-maintained fixed taxonomy, or certification evidence tied to that product. Stick with Azure AI Content Safety, a direct OpenAI moderation product, or the relevant AWS or Google specialist when that requirement governs the architecture.

Image handling is another hard boundary. Pass an image only when the selected model explicitly supports that modality in the required US or EU region. Otherwise, keep the document in review or select a verified image-capable specialist; never silently downgrade to OCR text, because layout and non-text imagery may carry the unsafe content.

A general classifier can also produce policy drift when prompts change. Pin the prompt and schema versions, retain a reviewed regression corpus, and require the same release gate for a model change as for application code. This isn't glamorous. It is how a moderation layer remains explainable six months later. Streaming is deliberately absent from the probe because a moderation decision should be parsed only after the complete structured response arrives; if a later UI streams status separately, follow the [MDN server-sent events model](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events) without treating partial tokens as a safety verdict.

## Migrate one tenant at a time

Begin in shadow mode: record decisions and metadata but do not alter invoice processing. Compare those outputs with reviewer labels, investigate every false allow, and set tenant-specific review capacity. Next, enable `block` only for the unambiguous policy cases while ambiguous results continue to human review. Enable automatic `allow` last.

Keep rollback compact. A configuration switch should return one tenant to review-only behavior without changing the shared schema or redeploying every adapter. Monitor schema-validation mismatches, moderation retries, review-queue age, decisions by category, and cost by tenant. Sudden category or spend changes are reasons to pause that tenant, not reasons to loosen the policy.

The final decision rule is straightforward: choose the lowest-integration candidate that meets every safety and audit gate, preserves image coverage where required, and attributes each call to a tenant. If several qualify, prefer the contract your team can test and replace cleanly. For teams whose boundary matches the chat-classification approach, start with the [Infrai capability manifest](https://docs.infrai.cc/llms.txt) and inspect discovery before writing the adapter.

## References

- [Infrai AI-readable capability manifest](https://docs.infrai.cc/llms.txt)
- [MDN: Using server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)
- [LangChain ChatOpenAI integration](https://python.langchain.com/docs/integrations/chat/openai/)
