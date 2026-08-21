# Marketplace Content Moderation with Batch Classification, Token Cost, and Review Queue Control

Short answer: the cheapest defensible way to moderate a large stream of marketplace reports is to make most decisions with deterministic rules, send only ambiguous content through batch classification, and reserve the review queue for uncertain or high-impact cases. Count actual input and output tokens per decision, not characters per upload, and keep provider-specific requests behind an adapter so a price change doesn't become a migration project.

The important trade-off is false-negative exposure versus reviewer load. A low model invoice can hide an expensive queue, while aggressive automation can hide harmful listings or punish legitimate sellers. Cost therefore belongs beside label quality, retry behavior, appeal risk, and provider portability. It isn't a leaderboard number.

## How should batch classification route large-volume user content into a review queue?

Start with a narrow decision contract. For a marketplace report, the classifier should return a policy label, a confidence band, a reason code, and the version of the policy and prompt that produced the result. It should not decide account punishment. Enforcement has more context, larger consequences, and different audit requirements. Keeping those boundaries separate also means a new classifier can be replayed against stored, redacted inputs without silently changing the enforcement system.

A useful routing sequence is:

1. Reject malformed or oversized submissions before any model call.
2. Apply exact rules for known signals such as an invalid category or a duplicate report.
3. Group eligible reports by policy version, language, and risk tier.
4. Batch-classify only the text fields required by that policy.
5. Auto-close low-risk, high-confidence results; escalate uncertainty, conflicting signals, and high-impact categories.
6. Sample a small portion of auto-closed results for quality control, with the sampling rate set by risk rather than convenience.

The queue record needs more than `label=spam`. Store an immutable decision ID, a content hash, policy version, model adapter name, request attempt, usage returned by the provider, result, and routing reason. Keep sensitive message text under its own retention and access rules. This is the same instinct that keeps OTP systems sane: the event that triggered work, the delivery attempt, and the final disposition are related, but they are not one blob with one retention policy.

Labels will drift.

That is why every human override should preserve both the original output and the corrected label. Replacing history destroys the evidence needed to tune thresholds, compare adapters, or explain why reviewer volume changed after a rollout.

## Build the estimate from a token ledger

A cost estimate should be a replayable equation, not an average copied from a dashboard. For each policy and language segment, measure accepted reports, rule-filtered reports, classifier inputs, classifier outputs, retries, and escalations. Then calculate model spend from the provider's current input and output rates, supplied as configuration rather than embedded in business logic.

For one period, let `N` be submitted reports, `f` the fraction removed by deterministic filters, `i` average input tokens, `o` average output tokens, and `r` the average number of billed classification attempts per eligible report. If input and output prices per million tokens are `P_in` and `P_out`, the model estimate is:

`N * (1 - f) * r * ((i * P_in + o * P_out) / 1_000_000)`

Add reviewer cost separately: escalated reports multiplied by measured handling time and the organization's loaded labor rate. Also keep storage, observability, and data-transfer costs on their own lines. Blending them into a single per-report figure makes a provider comparison look easy, but it prevents anyone from seeing whether a change reduced inference spend by increasing manual work.

Consider an illustrative planning batch of 100,000 reports. If rules remove 35,000, classification receives 65,000. Those numbers are an example, not a benchmark. Plug measured token usage and current contracted rates into the ledger; don't infer tokens from byte length once adapters are in production, because tokenization and usage accounting are provider-specific. I'm not sure which provider will be cheapest for an unseen language and policy mix — a matched replay with current quotes is what resolves that uncertainty.

Short outputs matter. A fixed schema with a compact reason code is easier to validate and usually avoids paying for prose that reviewers won't read. Still, a two-token label with no policy version or trace identifier is false economy: it weakens appeals, evaluation, and incident reconstruction.

## Keep provider portability in the contract

Portability starts above HTTP. Each adapter must accept the same normalized moderation item and return the same decision envelope, while retaining raw provider usage in restricted telemetry. The application owns IDs, policy versions, deadlines, and idempotency keys. The adapter owns authentication, request shape, provider tokenization, and response parsing.

The focused Python example below leaves network calls to concrete adapters. That is deliberate: inventing a universal endpoint would make the example less portable, not more.

```python
from dataclasses import dataclass
from decimal import Decimal
from typing import Protocol, Sequence


@dataclass(frozen=True)
class Report:
    report_id: str
    listing_text: str
    reason: str
    policy_version: str


@dataclass(frozen=True)
class Decision:
    report_id: str
    label: str
    confidence: Decimal
    reason_code: str
    input_tokens: int
    output_tokens: int


class ClassifierAdapter(Protocol):
    name: str

    def classify_batch(self, reports: Sequence[Report]) -> list[Decision]:
        ...


def route(decision: Decision, high_impact_labels: set[str]) -> str:
    if decision.label in high_impact_labels:
        return "human_review"
    if decision.confidence < Decimal("0.90"):
        return "human_review"
    return "auto_close"


def estimate_model_cost(
    decisions: Sequence[Decision],
    input_price_per_million: Decimal,
    output_price_per_million: Decimal,
) -> Decimal:
    input_tokens = sum(item.input_tokens for item in decisions)
    output_tokens = sum(item.output_tokens for item in decisions)
    million = Decimal(1_000_000)
    return (
        Decimal(input_tokens) * input_price_per_million / million
        + Decimal(output_tokens) * output_price_per_million / million
    )
```

The `0.90` threshold is illustrative, not a universal safety line. Calibrate it per label against reviewed data, because the cost of missing prohibited goods can be very different from the cost of sending a merely irrelevant report to a person. Keep thresholds outside the adapter so a provider switch doesn't change policy by accident.

HTTP behavior also belongs in the portability contract. RFC 9110 distinguishes idempotent methods and explains why clients can automatically retry an idempotent request after a communication failure. A classification submission built on a non-idempotent operation needs an application-level idempotency key and a durable attempt record; otherwise, a retry can create duplicate work and duplicate charges. Retry only transport failures and explicitly retryable responses, apply bounded backoff, and stop at the report deadline. Don't turn every rejection into another paid attempt.

## Test queue pressure against label quality

A provider evaluation needs the same frozen, redacted corpus, policy version, output schema, and acceptance checks for every adapter. Split results by label, language, text-length band, and seller-impact tier. Aggregate accuracy can conceal the exact edge case that fills the queue — for example, one language segment producing many low-confidence decisions even though the overall score barely moves.

Track schema-valid response rate, per-label precision and recall on reviewed examples, escalation rate, reviewer override rate, p50 and p95 completion time, attempts per accepted decision, input and output tokens, and cost per accepted decision. The prompt itself is versioned test material. Prompt-engineering techniques can change model behavior, so a prompt edit deserves the same replay and staged rollout as an adapter change.

Guardrails should fail closed into human review for missing fields, unknown labels, policy-version mismatches, or confidence values outside the agreed range. That doesn't mean endlessly retrying invalid output. One bounded repair attempt may be part of the contract; after that, route the item with a machine-readable failure reason and preserve the attempt metadata. Reviewers need a stable queue, not a burst of duplicates.

Spam and abuse systems teach an awkward lesson: an attacker can optimize against any visible threshold. Avoid publishing exact enforcement cutoffs, cap report creation per account and target, and monitor correlated bursts. At the same time, rate limits must not erase a legitimate safety report. Preserve a compliant intake path, disclose the applicable retention policy, and restrict raw content access. Cheap classification is irrelevant if the surrounding workflow creates a privacy or appeals liability.

## Choose the operating point, then roll it out

There is no single cheapest architecture independent of workload. Deterministic filters plus batch classification are not suitable when decisions must be immediate, context spans live interaction state, or every action legally requires a human. In those cases, use synchronous classification for the latency-bound slice or keep mandatory review, then optimize reviewer tooling instead. A self-hosted model can suit teams that need tighter data control and have inference operations expertise; a managed API can suit teams that value a smaller operational surface. Neither choice removes evaluation or audit work.

Roll out by shadowing the new adapter first: write decisions and token usage, but don't alter enforcement. Next, send a small risk-bounded segment to the new decision path, compare queue pressure and overrides, and retain a switch back to the prior adapter. Expand by policy category and language only after acceptance criteria hold. The compact migration artifact is a versioned corpus, a provider-neutral decision schema, current rate inputs, threshold configuration, and a ledger that reconciles accepted decisions with billed tokens.

Price is one input. The durable advantage comes from being able to verify what was classified, why it was escalated, how much work it created for people, and whether another provider produces the same policy outcome.

## References

- RFC 9110: HTTP Semantics: https://www.rfc-editor.org/rfc/rfc9110
- Prompt Engineering Guide: https://www.promptingguide.ai
