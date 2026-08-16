# Practical Startup App Moderation Categories for Harassment, Self-Harm, Spam, and PII

Short answer: use seven stable business labels—harassment, sexual content, self-harm, violence, illegal activity, spam, and privacy/PII—then map each label separately to allow, review, or block for the product context. For a media startup that turns sales-call transcripts into CRM actions, keep that contract in your own code so changing the model provider doesn't change storage, reviewer queues, or CRM writes.

The important boundary is the CRM write. A summary can be useful while a proposed action still contains a phone number that should not be copied, a threat that needs review, or spam that should be dropped. Classify the proposed action before persistence; don't ask downstream CRM code to interpret a provider's safety vocabulary.

## Failure containment at the CRM write boundary

The architecture decision is to own a compact `category`, `severity`, and `action` schema at the application boundary. Model output is evidence for that contract, not the contract itself. Seven categories are enough to launch, and additions should follow observed policy gaps rather than every distinction a model can produce.

Seven in, three out.

The invariant is that the same input and business policy produce one of three operational outcomes: `allow`, `review`, or `block`. Labels describe content; outcomes describe what this startup does about it. A sales tool might review a transcript fragment mentioning self-harm because a human needs context, while a youth community might block publication of materially similar content. The category has not changed. The business rule has.

The failure boundary sits before any CRM mutation. Parse and validate the structured response, retain the matched labels and policy version for audit, and only then enqueue an approved action. If classification cannot produce valid structured output, fail closed into human review rather than guessing. I'm not sure a universal severity scale is defensible across every media product; calibration samples, reviewer disagreement, and applicable policy would resolve that for a specific deployment.

## How should a startup app define harassment, self-harm, spam, and PII categories?

Start with definitions that a reviewer can apply in one pass. Harassment covers targeted abuse or intimidation. Sexual content covers sexual material whose treatment depends on audience and product policy. Self-harm covers encouragement, intent, or instructions involving harm to oneself. Violence covers threats, encouragement, or graphic descriptions involving harm to others. Illegal activity covers facilitation or solicitation of prohibited acts. Spam covers unsolicited, repetitive, deceptive, or irrelevant promotion. Privacy/PII covers exposed personal data that the workflow should not propagate.

Those definitions are deliberately broad. The first policy version should resolve the common case without turning the prompt into a compliance manual. Add subcategories only when they change an action or a reviewer route; otherwise they create brittle prompts and a queue full of distinctions nobody uses.

For a sales-call summarizer, apply the taxonomy to transcript-derived CRM actions, not merely the raw transcript. Consider this input: a caller gives a private mobile number, insults an account manager, and asks for a follow-up. The source can legitimately contain all three facts, but the proposed CRM action should label harassment and PII, remove the private number from automatic persistence, and route the action for review. A single `unsafe: true` flag can't explain which transformation was required, and a provider-native label can't be trusted to remain stable after a provider switch.

Keep policy rules explicit:

| Category | Example signal in a proposed CRM action | Typical starting outcome |
|---|---|---|
| Harassment | Targeted abuse toward a named person | Review |
| Sexual content | Sexual material outside the sales task | Review |
| Self-harm | Intent, encouragement, or instructions | Review |
| Violence | A credible threat or violent instruction | Block |
| Illegal activity | Facilitation or solicitation | Block |
| Spam | Repetitive or deceptive promotion | Block |
| Privacy/PII | A private phone number or personal identifier | Review |

These outcomes are examples, not universal law. Age, jurisdiction, CRM access controls, and the app's publication model can move any row. Your mileage may vary—especially for quotations in journalism, where preserving source material and distributing it are different actions.

## Privacy and audit controls during provider migration

A provider change passes only if the stored category values, action mapping, reviewer queue payload, and CRM writer interface remain unchanged. The adapter and its contract tests may change. Product storage and policy history may not.

Use a fixed corpus of representative call excerpts to test that boundary before choosing an integration. Include an action with both harassment and PII, an ambiguous quotation, harmless contact details supplied for a legitimate follow-up, and content with too little context. The acceptance check is structural and operational: valid schema, known label, explicit outcome, and the same downstream route. It isn't a claim that two models will agree on every judgment.

## Where should provider portability live?

Portability lives in the adapter that turns a provider response into the startup's schema. It does not live in prompt prose alone. Pin the JSON shape, validate every response, version the business mapping, and keep the provider client behind one interface.

| Option | Contract owned by | Portability consequence | Best fit |
|---|---|---|---|
| Direct OpenAI integration | Provider client plus application adapter | Switching requires another adapter | Teams already standardized on one provider |
| Direct Anthropic integration | Provider client plus application adapter | Switching requires another adapter | Teams that accept provider-specific integration work |
| Direct Google Gemini integration | Provider client plus application adapter | Switching requires another adapter | Teams committed to that provider boundary |
| Infrai OpenAI-compatible surface | Application JSON schema behind one API surface | The application contract can stay put while routing behind it changes | Small teams prioritizing provider portability across backend capabilities |

The last option has a concrete advantage here: model routing can change behind one OpenAI-compatible REST surface without changing the moderation schema or CRM code, and the same key and bill cover the broader capability surface. The catch is that it has no dedicated moderation endpoint; text or image moderation must use a chat model with a `json_schema` response. Stick with a dedicated provider product when its native moderation taxonomy, policy controls, or compliance arrangement is a firm requirement. Direct integration is also reasonable when the team has one approved provider and no realistic switching need.

No option removes governance work. A portable bad policy is still a bad policy.

## Rate-limit recovery in the classification adapter

The focused path below classifies a proposed CRM action before writing it. It uses the verified OpenAI-compatible chat route, reads the key from the environment, requests a strict schema, and retries HTTP 429 with `Retry-After` when available. I've learned from rate-limited delivery flows that a 429 is an instruction, not permission to spin in a tight loop—the same rule belongs here.

```python
import json
import os
import time
from typing import Any

from openai import OpenAI, RateLimitError


client = OpenAI(
    api_key=os.environ["INFRAI_API_KEY"],
    base_url=os.environ["OPENAI_BASE_URL"],
    max_retries=0,
)

CATEGORIES = [
    "harassment",
    "sexual",
    "self_harm",
    "violence",
    "illegal",
    "spam",
    "pii",
]

MODERATION_SCHEMA: dict[str, Any] = {
    "name": "crm_action_moderation",
    "strict": True,
    "schema": {
        "type": "object",
        "properties": {
            "category": {"type": "string", "enum": CATEGORIES},
            "severity": {"type": "string", "enum": ["low", "medium", "high"]},
            "action": {"type": "string", "enum": ["allow", "review", "block"]},
            "reason": {"type": "string"},
        },
        "required": ["category", "severity", "action", "reason"],
        "additionalProperties": False,
    },
}


def classify_crm_action(action_text: str, attempts: int = 4) -> dict[str, Any]:
    for attempt in range(attempts):
        try:
            response = client.chat.completions.create(
                model="deepseek-v4-flash",
                messages=[
                    {
                        "role": "system",
                        "content": (
                            "Classify the proposed CRM action with the supplied schema. "
                            "Labels describe content; actions follow business policy. "
                            "When context is insufficient, choose review."
                        ),
                    },
                    {"role": "user", "content": action_text},
                ],
                response_format={
                    "type": "json_schema",
                    "json_schema": MODERATION_SCHEMA,
                },
            )
            content = response.choices[0].message.content
            if content is None:
                raise ValueError("Model returned no structured moderation result")
            result = json.loads(content)
            if set(result) != {"category", "severity", "action", "reason"}:
                raise ValueError("Moderation result does not match the required fields")
            return result
        except RateLimitError as error:
            if attempt == attempts - 1:
                raise
            retry_after = error.response.headers.get("retry-after")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("Moderation attempts exhausted")


if __name__ == "__main__":
    proposed_action = "Call the buyer at the private number quoted in the transcript."
    decision = classify_crm_action(proposed_action)
    print(json.dumps(decision, indent=2))
```

Set `OPENAI_BASE_URL` to the selected provider's OpenAI-compatible v1 base before running the sample. There is no write call on purpose. The classifier returns a decision; a separate worker applies the versioned business rule and performs a CRM mutation only for an allowed or reviewed-and-approved action. That split makes retries safer and leaves an auditable point where policy can evolve without rewriting the CRM UI or storage model.

## Which design was rejected, and when is it valid?

The rejected design is a single provider-native `safe` boolean written directly beside the CRM action. It looks efficient, but it collapses label, severity, and business outcome into one value. Reviewers cannot tell privacy exposure from harassment, product teams cannot revise one policy mapping, and a provider replacement leaks into stored records.

Still, it has a valid use case. A low-risk internal prototype with no automatic external action, no sensitive retention, and a human reviewing every result may prefer the smaller implementation. Keep it temporary, record the raw proposed action separately from the decision, and migrate before automation makes the boolean consequential.

For production, test the seven categories with representative sales-call excerpts, ambiguous quotations, multiple labels, and empty context. Watch reviewer disagreement rather than chasing a huge taxonomy. Expand only when a new distinction changes `allow`, `review`, `block`, or the destination queue.

## References

Further reading:

- [OpenAI Batch API guide](https://platform.openai.com/docs/guides/batch)
- [OpenAI Whisper repository](https://github.com/openai/whisper)
