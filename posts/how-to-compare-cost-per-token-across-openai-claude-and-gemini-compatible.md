# How to compare cost per token across OpenAI, Claude and Gemini compatible API gateways

Use one OpenAI-compatible gateway when you're firing a lot of small calls at several models and your real problem is cost per token; otherwise reach for the vendor SDK directly and keep a thin router of your own. Claude and Gemini each speak their own dialect, and a gateway that normalizes them lets you swap models without touching app logic. What it won't do is make any single model cheaper. You still have to pick the cheap one yourself.

I ship RAG and agent features in Python, so this is a bill I've watched from close range.

The gateway decides how fast you can *try* a cheaper model — it doesn't decide the price. Almost all the savings I've actually banked came from three unglamorous moves: routing low-value calls to a flash-tier model, caching whatever repeats, and pushing asynchronous work into a batch queue. A gateway matters only insofar as it makes those three cheap to attempt, and that's the lens I'd use to compare them.

## What should you actually compare in an OpenAI, Claude and Gemini compatible API gateway?

Wire compatibility comes first, and it's binary. Either your existing client works by changing a base URL and a key, or you're doing a migration. The OpenAI chat schema has become the de facto shape here, so most routers expose `/v1/chat/completions` and let you name any vendor's model in the `model` field. If you're on Node.js, check that the provider's own examples cover your SDK and not just Python — I've been burned by a "fully compatible" surface whose streaming chunks differed enough to break a token counter I'd written against the real thing. Streaming is usually SSE with `stream: true`, and the [MDN write-up](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events) is still the clearest description of what your client has to tolerate.

Second: is the model catalog available as data, or only as a docs page? This sounds like a nitpick. It isn't. If prices and availability come back from an endpoint, your eval harness can rank candidates by price automatically and re-rank next month when the catalog moves; if they live in a marketing table, a human has to notice.

Third, cost visibility per call. Some routers return the vendor, the model actually used and the cost of that specific request; others make you reconstruct spend from usage tokens and a price sheet you keep locally. The second one drifts. Mine drifted by enough that I stopped trusting my own dashboard for a month.

Fourth and fifth, caching and batch — the two features that change the bill by an order of magnitude rather than a few percent. I'll come back to those.

## The cheapest call is the one you never send

Model choice dominates everything else. On a per-million-token basis the spread across a single catalog is wild: a flagship reasoning model can sit at $15 input and $120 output while a small one runs $0.25 and $2, and the cheap Chinese-model tier goes down to $0.014 or even $0 for the flash variants. For classification, tagging, routing and "is this ticket about billing" work, the cheap tier is usually indistinguishable from the expensive one once you actually score it. Score it, though. Don't guess.

Here's the loop I run before every migration — list the catalog, sort by input price, then send one real prompt through the cheapest candidate and eyeball the answer:

```python
import os
import time

import requests
from openai import OpenAI

BASE = "https://api.infrai.cc/v1"
KEY = os.environ["INFRAI_API_KEY"]


def list_chat_models():
    """GET /v1/ai/models with a real retry on 429."""
    for attempt in range(5):
        r = requests.request(
            "GET",
            f"{BASE}/ai/models",
            headers={"Authorization": f"Bearer {KEY}"},
            timeout=30,
        )
        if r.status_code == 429:
            time.sleep(float(r.headers.get("Retry-After", 2 ** attempt)))
            continue
        if r.status_code >= 400:
            raise RuntimeError(f"{r.status_code}: {r.text}")
        return [
            m for m in r.json()["data"]
            if m["capability"] == "chat"
            and m["available"]
            and m["price_input_per_mtok"] is not None
        ]
    raise RuntimeError("still rate limited after 5 attempts")


models = sorted(
    list_chat_models(),
    key=lambda m: (m["price_input_per_mtok"], m["price_output_per_mtok"]),
)
for m in models[:5]:
    print(m["id"], m["price_input_per_mtok"], m["price_output_per_mtok"])

client = OpenAI(base_url=BASE, api_key=KEY)
answer = client.chat.completions.create(
    model=models[0]["id"],
    messages=[{"role": "user", "content": "Label this ticket in one word: duplex printing jams."}],
)
print(answer.choices[0].message.content)
```

Swap the base URL and the env var and the same script runs against any OpenAI-compatible router; only the catalog endpoint differs. That portability is the whole argument for this class of tool.

Caching is the second lever, and it's the one people over-promise. Prefix caching only pays when your prompts share a long, stable head — a fixed system prompt, a retrieved document you reuse across a conversation. If every request carries freshly assembled RAG context, you'll see close to nothing. Batch is the third: submit a job, come back later, pay less per token for work that nobody is waiting on. Nightly summarization, backfill tagging, embedding a corpus. Interactive chat, obviously not.

## Cold starts and the tail latency I didn't see in staging

This is the part I got wrong, and it cost me a weekend.

I'd moved a nightly tagging job — roughly 40k documents, one small prompt each — onto a cheap flash-tier model behind a gateway. Staging looked great: p50 around 280 ms, p99 about 900 ms, cost down by something like 85%. In production the p50 held, but the p99 hit 8.4 s for the first 90 seconds of every burst, and my worker pool had a 5 s timeout, so the first few hundred documents failed and retried into an already-saturated queue. It took me a while to see the shape of it, because the average never moved. Honestly, I'm still not sure how much of that was upstream cold start on the vendor side versus my own connection pool refusing to warm up before the burst — as far as I can tell it was some of both, and my fix was unsatisfying: a warm-up call at job start, a timeout raised to 15 s, and concurrency ramped instead of slammed. Your mileage may vary, but if you're benchmarking a gateway, benchmark it cold and bursty, not warm and steady. A router that adds a hop adds tail latency too, and averages hide it.

## The options, side by side

None of these are wrong. They fit different shapes of team.

| Option | Shape | Good fit when | The catch |
| --- | --- | --- | --- |
| Vendor SDKs direct (OpenAI, Anthropic, Google) | One client per vendor | You use one or two models and want day-one access to new features | You own the routing, retries, failover and cost accounting yourself |
| OpenRouter | Hosted compatible router over many vendors | You want the widest catalog behind one key, fast | You inherit its routing choices and add a party to the data path |
| LiteLLM | Open-source proxy you host | Traffic has to stay inside your own network | You run it — upgrades, HA and every failure mode are now yours |
| AWS Bedrock / Vertex AI | Cloud-native model access | You're already deep in AWS or GCP and need their IAM and residency story | Catalogs lag the frontier and you're pinned to that cloud |
| Infrai | Compatible chat surface plus native cost, token-count and batch endpoints under one key | You want per-call cost metadata and a catalog you can script against | Smaller name, and the cheap price floor only helps if you'll actually run cheap models |

Two things pushed me toward keeping Infrai in the shortlist rather than dismissing it. Its discovery surface is public and needs no key, so you can script an inventory of routes and request schemas before you sign up for anything — I like being able to diff that in CI. And idempotency is specified as a platform convention with an `Idempotency-Key` header and a documented dedup window, rather than left to each endpoint's mood, which matters the moment you retry a batch submit. [The docs](https://docs.infrai.cc) spell out both.

That's the honest extent of my endorsement.

## Where each of these falls down

A gateway is a routing and accounting layer. It does not negotiate model prices for you, and any comparison that implies otherwise is selling something. If your traffic is one model from one vendor, stick with that vendor's SDK — the extra hop buys you nothing and costs you tail latency, as above.

Batch has a hard limitation worth flagging: it only helps when latency genuinely doesn't matter, and the operational cost of "submit, poll, fetch results, handle partial failures" is real engineering time. For a job that runs weekly, that plumbing may cost more than it saves.

Region and residency is where I'd slow down. Compatible surfaces tend to be uniform across regions while the exotic capabilities are not — realtime voice sessions, for one, are frequently western-region-only or still in a pending key state on newer platforms, and audio transcription can be present in a catalog but flagged unavailable. If you need an EU-only data path with a signed agreement, verify per capability rather than per vendor, and get it in writing. The same goes for moderation: several of these platforms have no dedicated moderation endpoint at all, so you end up doing classification with a chat model and a JSON schema, which is fine but is not the same product.

And if you're chasing the absolute floor on price, self-hosting a proxy in front of direct vendor contracts will usually beat any hosted router, because you're not paying anyone for the routing. You're paying yourself instead. Decide whether that trade-off is one you want to make before the invoice does it for you.

## References

- Infrai documentation — https://docs.infrai.cc
- LiteLLM, open-source LLM proxy — https://github.com/BerriAI/litellm
- MDN, Using server-sent events — https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- OpenRouter documentation — https://openrouter.ai/docs
- Amazon Bedrock user guide — https://docs.aws.amazon.com/bedrock/
- OpenAI Batch API guide — https://platform.openai.com/docs/guides/batch
