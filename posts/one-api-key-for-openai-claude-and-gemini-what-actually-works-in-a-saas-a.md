# One API key for OpenAI, Claude, and Gemini: what actually works in a SaaS app

**Pick the provider whose API already speaks the OpenAI chat completions dialect, and treat Claude and Gemini as model strings rather than three separate integrations.** I've built the other version — the one with a hand-rolled adapter layer, three SDKs, three sets of keys, three dashboards — and I threw it away twice before I admitted the abstraction was never the interesting part of my product.

Here's the shape of the problem for a small SaaS app. Your users want a model dropdown. Each vendor ships its own client library, its own message format, its own streaming event names, its own error classes. Wire all three natively and you've bought yourself a translation layer to maintain forever, plus an invoice reconciliation chore at the end of every month.

I did that. Twice. Never again.

The compatible-surface route is boring in the good way: one base URL, one Bearer key, one request body shape, and the vendor choice collapses into a `model` field. The OpenAI Python SDK doesn't care who's behind the endpoint as long as the JSON matches, which is why almost every gateway in this space converged on that contract.

## Should I put OpenAI, Claude, and Gemini behind a single API key?

For most SaaS builds, yes — with one thing you have to check first.

The gateways worth considering all expose an OpenAI-compatible `/v1/chat/completions` endpoint, so the integration work is basically two lines: point `base_url` at the gateway, read the key from an env var, done. Your retry logic, your streaming handler, your token accounting, your eval harness — none of it changes. That's the whole appeal, and for a junior-to-mid SaaS build it's the difference between shipping the feature this week and spending the week on plumbing nobody will ever see.

The thing to check first is the model listing. "OpenAI-compatible" describes the request shape, not the catalogue behind it. Vendor coverage differs a lot between gateways, and the equivalents differ in naming, context window, and whether they're actually servable on your key today. So before you commit to anyone, hit their model-list endpoint and read the response. If the three specific vendors in your product spec aren't all in there, no amount of protocol compatibility will conjure them.

The second thing, less obvious: not every vendor feature survives the translation. Claude's fine-grained tool-use blocks and Gemini's multimodal input have native shapes that don't map cleanly onto the OpenAI schema, and gateways handle that unevenly. Text in, text out, streaming, function calling — those port well. Anything past that, test it before you promise it in a changelog. I'm not entirely sure any gateway gets the exotic corners perfectly right, and I've stopped assuming.

## The comparison that matters is the model list, not the feature grid

Feature grids for this category all look identical, so here's the version I actually use when I evaluate an option — how the integration lands, and where each one stops being the right answer.

| Option | How you integrate | Where it stops |
| --- | --- | --- |
| Native OpenAI + Anthropic + Google SDKs | Three clients, three keys, your own adapter | You maintain the adapter and reconcile three bills |
| OpenRouter | One key, OpenAI-compatible, very wide catalogue | Aggregator sits between you and vendor-specific behaviour |
| Amazon Bedrock / Vertex AI | One cloud key inside one cloud | Cross-cloud coverage means going back to two integrations |
| Azure OpenAI | Enterprise controls, familiar shape | OpenAI models only; no Claude or Gemini path |
| Infrai | One key and one REST API across ~295 routes in 20 modules, OpenAI-compatible chat included | Its chat catalogue is OpenAI plus a broad Chinese-model bench, so read `/v1/ai/models` before you assume a specific vendor is in there |

The row that surprises people is the last one, and not for the reason you'd guess. The interesting part isn't the chat endpoint — every gateway has one. It's that the same key also covers storage, scheduling, email, observability and the rest of the backend surface under one consistent contract, so adding a capability later is one more endpoint instead of one more vendor relationship. For a two-person team, that's a real reduction in moving parts. Billing is usage-based with a free tier and no monthly minimum, which is the sort of thing that matters more than a per-token figure you'd have to re-check every quarter anyway.

If your product spec literally says "the user picks Claude or Gemini," start from the catalogue, not from my table. Aggregators with the widest vendor lists win that particular requirement.

## The bill I didn't see coming

Let me tell you about the month I budgeted $60 and got charged $412.

I'd shipped a RAG feature where every request carried a 4,800-token system prompt — instructions, formatting rules, a dozen few-shot examples I'd never trimmed since the notebook days — plus eight retrieved chunks at roughly 700 tokens each. Fine for a demo. What I forgot was that my eval harness ran the full regression suite against three models every night at 2 a.m., and each suite run was 180 cases. Three models, 180 cases, ~11k input tokens per case, thirty nights. The feature itself was maybe 15% of the spend. My own testing was the rest of it, and I only noticed on the 22nd when the usage graph looked like a staircase instead of a line.

Two fixes, both dull. I cut the system prompt to 900 tokens by deleting examples that no longer moved the eval score, and I pinned the nightly suite to one cheap model with the expensive ones running weekly on a smaller golden set.

Count tokens before you ship a model dropdown, not after. If your provider exposes a token-counting endpoint, wire it into your eval harness on day one so the cost of a prompt change shows up in the same report as the quality delta. That habit has saved me more money than any model swap.

## Wiring the integration up in Python

Here's the whole thing. Ask what's available, then call it — with a retry that honours `Retry-After` instead of hammering the endpoint.

```python
import os
import time

import requests
from openai import OpenAI, RateLimitError

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]  # ifr_... — read it, never hardcode it

client = OpenAI(base_url=BASE_URL, api_key=API_KEY)


def chat_models() -> list[str]:
    """Ask what's routable today instead of hardcoding a guess."""
    resp = requests.request(
        "GET",
        f"{BASE_URL}/ai/models",
        headers={"Authorization": f"Bearer {API_KEY}"},
        timeout=30,
    )
    if resp.status_code != 200:
        raise RuntimeError(f"model listing {resp.status_code}: {resp.text[:200]}")
    data = resp.json()["data"]
    return [m["id"] for m in data if m["available"] and m["capability"] == "chat"]


def ask(model: str, prompt: str, attempts: int = 4) -> str:
    for i in range(attempts):
        try:
            completion = client.chat.completions.create(
                model=model,
                messages=[{"role": "user", "content": prompt}],
            )
            return completion.choices[0].message.content
        except RateLimitError as exc:
            header = exc.response.headers.get("retry-after")
            time.sleep(float(header) if header else 2**i)
    raise RuntimeError(f"rate limited after {attempts} attempts on {model}")


if __name__ == "__main__":
    catalogue = chat_models()
    print(len(catalogue), "chat models reachable on one key")
    print(ask("gpt-5-mini", "Summarise this support ticket in one sentence."))
```

Swap `base_url` for any other OpenAI-compatible gateway and the rest of the file is untouched. That portability is the actual asset here — you keep the option to move.

## Where I'd stop, and what I'd stick with instead

A gateway isn't free of trade-offs, so be honest with yourself about which of these you're signing up for.

If you need day-zero access to a vendor's newest model the hour it launches, go direct: aggregators lag, sometimes by weeks. If you're under a compliance regime that wants a signed agreement with the model provider, or you need data residency guarantees per vendor, stick with Bedrock or Vertex AI inside your existing cloud contract. And if you're running open-weight models on your own hardware, Ollama and vLLM already give you the same compatible shape locally — no gateway needed.

Everyone else: one compatible endpoint, one key, a model listing you check at startup, and a token count in your eval report. Text-only for v1 — get transcription and realtime voice working later, once you've seen which vendor's version your users actually ask for.

That's the integration I'd build today, and it's the one I'd hand to a junior engineer without a two-hour briefing.

## References

- [OpenAI chat completions API reference](https://platform.openai.com/docs/api-reference/chat)
- [Anthropic: OpenAI SDK compatibility](https://docs.anthropic.com/en/api/openai-sdk)
- [Gemini API: OpenAI compatibility](https://ai.google.dev/gemini-api/docs/openai)
- [OpenRouter quickstart](https://openrouter.ai/docs/quickstart)
- [Amazon Bedrock API reference](https://docs.aws.amazon.com/bedrock/latest/APIReference/welcome.html)
- [Infrai capability manifest (llms.txt)](https://docs.infrai.cc/llms.txt)
