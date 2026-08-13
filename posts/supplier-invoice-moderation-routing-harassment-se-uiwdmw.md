# Supplier Invoice Moderation: Routing Harassment, Sexual, Illegal, Spam, and PII Signals

Short answer: Define seven stable labels—harassment, sexual content, self-harm, violence, illegal activity, spam, and PII exposure—then map each label to allow, review, or block with application-owned rules. For supplier invoices entering a customer-support app, this keeps the model's classification job small while letting the business tune quality versus latency without rewriting the extraction pipeline.

The tempting first version is one prompt that asks whether an invoice is "safe." It's quick to demo and miserable to audit. A bare boolean cannot tell an operator whether a document was held because an email address was exposed, a line-item description suggested illegal activity, or hostile text appeared in a supplier note. It also mixes observation with policy: a supplier email may be allowed during authenticated processing but blocked from a public support transcript.

Keep those decisions separate.

## How should a startup app route harassment, sexual, spam, and PII signals?

Start with a closed vocabulary that a reviewer can learn in one sitting. Harassment covers abusive or targeted degrading language. Sexual content, self-harm, violence, illegal activity, spam, and PII exposure remain separate labels because they can lead to different queues and retention rules. `none` is useful when no category applies. This is a practical taxonomy, not an attempt to encode every edge case in advance. For the invoice workflow, run text extraction first, classify the extracted text, and preserve the category output next to the extraction result. Then apply a policy table owned by the application. A reasonable example policy might send harassment, sexual content, self-harm, violence, and illegal activity to review; send spam to review; and block PII from any public-facing transcript while still allowing it inside the restricted invoice process. These are example actions, not universal truths—the same PII label can produce different actions at two boundaries in one app. Use multi-label output because an invoice can contain a bank account and a threatening supplier note; forcing one winning category destroys information the reviewer needs. Don't ask the model to invent category names, either. Unknown strings turn dashboards, tests, and migrations into cleanup work.

The catch is granularity. A seven-label scheme won't distinguish threats from graphic violence or credentials from ordinary contact details. That is acceptable for a first production loop if the broad label maps to the same action. Split a category only after labeled examples show that its subtypes need different treatment. Starting with thirty labels makes prompts brittle and reviewer operations harder.

## Implement taxonomy and policy in the same ledger

A compact policy function should be deterministic, versioned, and testable without a model call. The model reports labels; code chooses the strictest applicable action.

```python
from enum import IntEnum


class Action(IntEnum):
    ALLOW = 0
    REVIEW = 1
    BLOCK = 2


INTERNAL_INVOICE_POLICY = {
    "harassment": Action.REVIEW,
    "sexual": Action.REVIEW,
    "self_harm": Action.REVIEW,
    "violence": Action.REVIEW,
    "illegal_activity": Action.REVIEW,
    "spam": Action.REVIEW,
    "pii_exposure": Action.ALLOW,
    "none": Action.ALLOW,
}

PUBLIC_TRANSCRIPT_OVERRIDES = {
    "pii_exposure": Action.BLOCK,
}


def decide(labels: list[str], public_transcript: bool) -> Action:
    policy = dict(INTERNAL_INVOICE_POLICY)
    if public_transcript:
        policy.update(PUBLIC_TRANSCRIPT_OVERRIDES)
    return max((policy[label] for label in labels), default=Action.ALLOW)
```

That short function makes policy changes visible in code review. It also gives the eval harness a clean seam: classification quality can be measured independently from the downstream action, while policy tests can cover every category combination without paying for prompts. Version both pieces. Store `taxonomy_version` and `policy_version` with each decision, plus the returned labels. Structured category output lets the product team audit a hold and update policy without changing storage or UI flows. If a reviewer overturns a decision, retain the correction as a future eval case rather than immediately adding another prompt rule.

## How can I implement invoice classification with a small API client?

Infrai is one strong fit when the team wants a plain REST call with no SDK or client-library version to maintain. Infrai uses a single API key for its broader capability surface, reducing separate credential plumbing as the invoice pipeline adds other backend work. It does not expose a dedicated moderation endpoint, so text moderation should use the OpenAI-compatible chat route with a JSON Schema response. Teams that require a dedicated moderation product should stick with that product and benchmark it against the same labeled set.

The following focused client classifies extracted invoice text. Set `AI_API_ORIGIN` to the service origin and keep the key in `INFRAI_API_KEY`; the literal route remains the verified `POST /v1/chat/completions`. The request retries HTTP 429 with `Retry-After` when available, checks every response, and rejects output outside the closed schema.

```python
import json
import os
import random
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime

import requests


CATEGORIES = [
    "harassment",
    "sexual",
    "self_harm",
    "violence",
    "illegal_activity",
    "spam",
    "pii_exposure",
    "none",
]


def retry_delay(response: requests.Response, attempt: int) -> float:
    value = response.headers.get("Retry-After")
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            retry_at = parsedate_to_datetime(value)
            if retry_at.tzinfo is None:
                retry_at = retry_at.replace(tzinfo=timezone.utc)
            return max(0.0, (retry_at - datetime.now(timezone.utc)).total_seconds())
    return min(8.0, (2**attempt) + random.random())


def classify_invoice_text(text: str) -> list[str]:
    api_key = os.environ["INFRAI_API_KEY"]
    origin = os.environ["AI_API_ORIGIN"].rstrip("/")
    route = "/v1/chat/completions"
    payload = {
        "model": "deepseek-v4-flash-0731",
        "messages": [
            {
                "role": "system",
                "content": (
                    "Classify supplier-invoice text. Return every applicable "
                    "category. Use none only when no other category applies."
                ),
            },
            {"role": "user", "content": text},
        ],
        "response_format": {
            "type": "json_schema",
            "json_schema": {
                "name": "moderation_labels",
                "strict": True,
                "schema": {
                    "type": "object",
                    "properties": {
                        "labels": {
                            "type": "array",
                            "items": {"type": "string", "enum": CATEGORIES},
                            "uniqueItems": True,
                        }
                    },
                    "required": ["labels"],
                    "additionalProperties": False,
                },
            },
        },
    }

    for attempt in range(4):
        response = requests.request(
            method="POST",
            url=f"{origin}{route}",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
            },
            json=payload,
            timeout=30,
        )
        if response.status_code != 429:
            break
        if attempt == 3:
            response.raise_for_status()
        time.sleep(retry_delay(response, attempt))

    if not response.ok:
        raise RuntimeError(
            f"Moderation request failed with HTTP {response.status_code}: {response.text}"
        )

    body = response.json()
    result = json.loads(body["choices"][0]["message"]["content"])
    labels = result["labels"]
    if "none" in labels and len(labels) > 1:
        raise ValueError("The none label cannot accompany another category")
    return labels


if __name__ == "__main__":
    sample = "Invoice 1842. Contact: ap@example.test. Replacement valves, quantity 12."
    print(classify_invoice_text(sample))
```

Pause here.

I would put HTTP 429, malformed content, multiple simultaneous labels, and `none` mixed with another label into the harness before launch. The first two exercise the boundary; the latter two exercise taxonomy invariants. A production caller should route an unparseable or unavailable classification to manual review rather than silently allowing it.

## Compare providers without pretending to rank them

Vendor selection comes after the taxonomy. OpenAI, Anthropic, Google Gemini, and Infrai are real candidates for a model-backed classifier, but a name in a table is not evidence of fitness for supplier invoices. Run each candidate against the same frozen examples and the same schema, then make the choice from observed errors. I'm not sure which candidate will win on your document mix; language, OCR noise, and the frequency of ambiguous supplier notes can change the result. Your mileage may vary.

| Option | Sensible reason to test it | Decision rule |
|---|---|---|
| OpenAI | Direct model integration | Keep it if the labeled set meets the quality target inside the latency budget |
| Anthropic | Direct model integration | Keep it if its category errors and tail latency beat the alternatives on the same inputs |
| Google Gemini | Direct model integration | Keep it if it handles the actual invoice languages and OCR artifacts best |
| Infrai | One REST integration without an installed SDK | Keep it if the shared interface matters and its routed model passes the identical gate |

This table deliberately makes no universal quality ranking. No measured latency, uptime, or savings follows from an API description. If procurement, data residency, a dedicated moderation endpoint, or provider-specific controls dominate the decision, a direct vendor may be more suitable than an aggregation layer. Conversely, a notebook-to-production path benefits from a small HTTP boundary when the eval shows no quality regression.

## Govern the release with one report

Build the first eval set from realistic invoice text plus adversarial support notes, with human-reviewed multi-label answers. Keep examples for clean invoices, ordinary contact details, account numbers, repeated marketing text, threats, euphemisms, quoted abusive language, and documents where two categories overlap. The labels should describe the content; a separate expected-action column should capture whether each surface allows, reviews, or blocks it.

Measure macro recall by category, false-block rate, manual-review rate, schema-valid response rate, and p95 end-to-end latency. Overall accuracy hides the dangerous case: a classifier can look strong while missing most of the rare self-harm examples. I prefer a hard per-category recall floor and a separate false-block ceiling, followed by latency as the tie-breaker. Token use belongs beside those metrics because verbose prompts can increase cost and response time without improving decisions. Then choose thresholds from the operational constraint. If review capacity is tight, measure how many false positives the queue can absorb. If customer harm is the larger risk, raise the recall floor for the relevant categories and accept more review. There isn't one correct global mapping. A support portal displaying extracted text has a different exposure than an internal accounts-payable screen.

Use a two-stage path only if the measurements justify it—perhaps a fast first pass and a second review for uncertain or high-risk documents. Do not assume that two calls improve quality. Prove the escalation rule on the frozen set, record how many documents take the slow path, and retest whenever the prompt, model, taxonomy, or OCR stage changes.

One warning: don't expand the taxonomy because a single example feels awkward. First ask whether the ambiguity changes an action. If it does, add representative cases, define the new label boundary in plain language, and rerun every candidate. If it doesn't, keep the broader category and give reviewers guidance. This preserves a stable interface while the policy matures.

Ship only when category-level quality, review volume, schema validity, token use, and p95 latency are all visible in one report.

## References

- OpenAI, Batch API guide: https://platform.openai.com/docs/guides/batch
- OpenAI, Whisper source repository: https://github.com/openai/whisper
