# One API key across OpenAI, Claude, and Gemini: what a router really costs

**Use a router like OpenRouter as the single API key across OpenAI, Claude, and Gemini while your startup app is small, and keep a direct provider client behind a config flag.** One router key gets you one credential, one invoice, and an OpenAI-compatible endpoint that the `openai` Python package already speaks, so the migration is about an afternoon of work. The pricing question underneath is duller than most comparison posts admit: for the big three you pay close to provider list price either way, and the real token cost savings come from sending cheap requests to cheap models, not from finding a clever middleman.

That's where the money is.

I ship RAG and agent features in Python, and I switched this layer twice last year — direct SDKs first, then a router, then a router with a second provider pinned for the paths that couldn't fail. Each move taught me something I'd have paid to learn earlier.

## Should a startup put OpenAI, Claude, and Gemini behind one API key?

Yes, if nobody on your team is paid to babysit vendor accounts.

Three direct integrations means three signups, three billing portals, three separate sets of rate limits to reason about, and three security questionnaires the first time an enterprise customer asks. A router collapses that into one contract. In my case the finance-side win landed harder than the engineering one — one prepaid balance instead of three company cards, plus a per-model spend breakdown I didn't have to build myself.

The engineering win is smaller than the pitch suggests. You still write a model-selection layer, you still own retries, and you still care which provider sits behind a given slug, because that's what sets your latency and which features you get. A router removes the account sprawl. It doesn't remove the routing logic, and I've seen teams act surprised by that.

## Four ways to buy one key, compared

| Approach | What you integrate | Covers OpenAI / Claude / Gemini | Main limitation |
| --- | --- | --- | --- |
| OpenRouter | one OpenAI-compatible base URL | all three, plus open-weight models | you inherit its uptime; per-model feature support varies |
| LiteLLM proxy (self-hosted) | a Python SDK, or a proxy you run | all three, through provider keys you still hold | you're on call for it; the vendor accounts stay in your name |
| Amazon Bedrock | AWS SDK and IAM | Claude yes, Gemini no, OpenAI open-weight only | tied to AWS regions and per-region quotas |
| Vertex AI | Google Cloud SDK | Gemini and Claude, no OpenAI | project and IAM setup is heavier than an API key |
| Direct SDKs | `openai`, `anthropic`, `google-genai` | all three, three keys | you write the routing and the accounting yourself |

Two of those look like the same product but aren't. OpenRouter is a hosted gateway that holds the provider relationships for you, so the credential count really does drop to one. LiteLLM is a library and a proxy you host, which unifies the call signature while leaving the OpenAI, Anthropic, and Google accounts in your own name — a good fit if procurement already approved those vendors and you just want one Python interface across them.

Bedrock and Vertex AI are the answer when your compliance story is already written in AWS or GCP terms. Neither one gives you all three families, so a startup that specifically wants to compare OpenAI against Claude against Gemini ends up with two clouds and no single key, which defeats the point.

On raw pricing: routers pass provider rates through and monetize elsewhere, typically on credit purchases rather than on tokens, so you shouldn't expect a discount versus going direct. Check the current numbers on each pricing page before you build a spreadsheet — every one of these moved at least twice in the past eighteen months.

## Token accounting is the part that decides your bill

The router won't make you cheaper. Measurement will.

Every OpenAI-compatible response carries a `usage` object, and if you fold it into a per-model ledger on the way out, you get cost attribution for free. This is the version I keep copying between projects:

```python
import os
from collections import defaultdict
from openai import OpenAI

client = OpenAI(
    base_url="https://openrouter.ai/api/v1",
    api_key=os.environ["OPENROUTER_API_KEY"],
)

# One slug per tier: swapping a vendor becomes a config change, not a refactor.
TIERS = {
    "cheap": "openai/gpt-4o-mini",
    "balanced": "anthropic/claude-3.5-sonnet",
    "long_context": "google/gemini-flash-1.5",
}

spend = defaultdict(lambda: {"prompt": 0, "completion": 0, "calls": 0})


def ask(tier: str, prompt: str) -> str:
    model = TIERS[tier]
    resp = client.chat.completions.create(
        model=model,
        messages=[{"role": "user", "content": prompt}],
        # Router-level fallback if the primary provider is unavailable.
        extra_body={"models": [TIERS["cheap"]]},
    )
    row = spend[model]
    row["prompt"] += resp.usage.prompt_tokens
    row["completion"] += resp.usage.completion_tokens
    row["calls"] += 1
    return resp.choices[0].message.content
```

Print that ledger at the end of every eval run and the routing decision stops being a debate:

```text
openai/gpt-4o-mini            412 calls   1,204,880 in    98,410 out
anthropic/claude-3.5-sonnet    47 calls     221,004 in    31,902 out
google/gemini-flash-1.5        11 calls   1,880,442 in     9,120 out
```

Now the war story I owe you, because this one cost me real hours. I had a retry wrapper around every call with four attempts and exponential backoff, written back when the whole thing lived in a notebook. It caught `RateLimitError` and retried. It logged nothing. My nightly eval over 800 examples crept from 9 minutes to 41 minutes across a week, and I blamed the vector store, then the embedding model, then my own batching — three evenings of poking at the wrong layer. The 429s were coming from a single provider whose per-minute quota I'd blown through by moving traffic onto it, and my retry loop was swallowing every one of them into a silent `sleep`. The fix took four lines: log the attempt, read the `retry-after` header instead of guessing, and emit a counter. I'm not sure why I ever wrote an exception handler with no logging in it — notebook habits, probably.

```python
import logging
import time

from openai import RateLimitError

log = logging.getLogger("llm")


def with_retries(fn, *args, attempts: int = 4, **kwargs):
    for i in range(attempts):
        try:
            return fn(*args, **kwargs)
        except RateLimitError as exc:
            wait = float(exc.response.headers.get("retry-after", 2 ** i))
            log.warning("429 on attempt %d/%d, sleeping %.1fs", i + 1, attempts, wait)
            time.sleep(wait)
    raise RuntimeError(f"rate limited after {attempts} attempts")
```

A router spreads that load across providers, which raises your effective ceiling. It also hides which provider is throttling you, so keep the model slug in your log line.

## When a router is the wrong call

The catch is feature lag, and it shows up in the places you'd least like to find it.

Strict structured outputs are the clearest example. OpenAI's `response_format` with a JSON schema and `strict: true` guarantees the shape of what comes back; Anthropic expects you to get there through tool definitions, and Gemini has its own response-schema field. A gateway passes your `response_format` down, but the guarantee only holds where the underlying model implements it, so the same call that validates against `openai/gpt-4o-mini` can hand you prose from a different slug. Test that path per model, not once.

Prompt caching and batch discounts are the other gap. Both are provider-specific mechanics that meaningfully change token cost at volume, and as far as I can tell, gateway support for them trails the providers by weeks and varies per model. If half your spend is a 30-thousand-token system prompt replayed all day, go direct and use the caching primitive — that's a bigger lever than any routing trick.

Streaming has a smaller wrinkle worth knowing. Chat completions stream over server-sent events, but the browser `EventSource` API only issues GET requests, so you can't point it at a POST endpoint; you either proxy through your own backend or read the stream with `fetch` and a `ReadableStream` reader. That's true of direct providers and routers alike.

Stick with direct SDKs when you're single-model and staying that way, when a compliance review names the sub-processors you're allowed to use, or when your spend is large enough that a provider will negotiate. Use Bedrock or Vertex AI when the data has to stay inside one cloud account. And if latency is your product, benchmark the extra network hop before you commit — it's usually small, and it's not always small.

## References

- OpenRouter quickstart and API reference — https://openrouter.ai/docs/quickstart
- LiteLLM documentation — https://docs.litellm.ai/docs/
- OpenAI structured outputs guide — https://platform.openai.com/docs/guides/structured-outputs
- OpenAI pricing — https://platform.openai.com/docs/pricing
- Anthropic pricing — https://www.anthropic.com/pricing
- Gemini API pricing — https://ai.google.dev/gemini-api/docs/pricing
- Amazon Bedrock supported models — https://docs.aws.amazon.com/bedrock/latest/userguide/models-supported.html
- MDN: Using server-sent events — https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
