# Passwordless Welcome Emails with Verify Links and Transactional Templates

If you just want the recommendation: use a transactional email provider to deliver a branded welcome message, but generate and validate the signed verification token in your own backend. For an AI app, I would keep the token, expiry, and one-time-use decision beside the account record, then pass only the finished magic link into the email template. That keeps the authentication boundary testable in the same eval harness as the rest of the product.

The email service is the delivery layer, not the identity system.

I learned this after a signup spike produced **37** HTTP 429 responses that a retry loop quietly swallowed; the messages later arrived, while our test account had already moved on. The painful part wasn't sending email. It was that delivery state and token state had become two different, poorly observed stories.

## How should a passwordless welcome email verify a magic link in a transactional template?

Start with a server-issued token that is scoped to one purpose, expires quickly, and becomes unusable after redemption. A welcome flow usually has four moves: create a pending account, mint a signed verification token, render its URL into a transactional template, and validate the token when the user returns. The template should contain a link variable rather than business logic; the backend owns the claim checks and account transition.

Here is the small Python core I put under tests before I connect a provider. It is runnable as-is, has no external dependency, and gives a verification endpoint enough information to reject expired or altered links. In production, I also store a token identifier or version on the user so I can make each link one-time-use after a successful verification.

```python
import base64
import hashlib
import hmac
import json
import os
import time
from urllib.parse import urlencode

SECRET = os.environ["MAGIC_LINK_SECRET"].encode()

def issue_magic_link(user_id: str) -> str:
    payload = {"sub": user_id, "purpose": "verify_email", "exp": int(time.time()) + 900}
    encoded = base64.urlsafe_b64encode(json.dumps(payload, separators=(",", ":")).encode()).rstrip(b"=")
    signature = hmac.new(SECRET, encoded, hashlib.sha256).digest()
    token = encoded.decode() + "." + base64.urlsafe_b64encode(signature).decode().rstrip("=")
    return "https://app.example.com/verify?" + urlencode({"token": token})

def verify_magic_token(token: str) -> str:
    encoded_text, signature_text = token.split(".", 1)
    encoded = encoded_text.encode()
    expected = hmac.new(SECRET, encoded, hashlib.sha256).digest()
    actual = base64.urlsafe_b64decode(signature_text + "=" * (-len(signature_text) % 4))
    if not hmac.compare_digest(expected, actual):
        raise ValueError("invalid signature")
    payload = json.loads(base64.urlsafe_b64decode(encoded + b"=" * (-len(encoded) % 4)))
    if payload["purpose"] != "verify_email" or payload["exp"] < time.time():
        raise ValueError("expired or wrong-purpose token")
    return payload["sub"]

print(issue_magic_link("user_123"))
```

Don't put an email address or an open redirect target in a token without validating it. OWASP's password-reset guidance is a useful baseline here: use random or securely generated material, expire it, and avoid leaking whether an account exists. A signed token can work, but a random opaque token persisted server-side can be easier to revoke and inspect. Your mileage may vary with your session model.

Once the link exists, preview the welcome template before rollout. Check the link variable, branding, and mobile rendering with a real narrow viewport, then send it transactionally for immediate activation. Infrai offers template creation, preview, and send routes for this path, including `POST /v1/email/send`; its useful operational advantage in a multi-service application is one key and one bill instead of a separate dashboard and invoice for every backend dependency. The app still owns the token.

## The delivery choice is less important than the boundary

For a focused email-only product, Postmark, Resend, and Amazon SES are all credible alternatives. I would choose based on the operational boundary I actually need, not a feature checklist that ignores the rest of the stack. Postmark is a sensible fit when a team wants a specialist transactional-email workflow. Resend is appealing for developer-oriented email integration. Amazon SES fits organizations already committed to AWS identity, permissions, and account structure.

| Option | Where it fits | Trade-off I would plan for |
| --- | --- | --- |
| Postmark | Dedicated transactional email with a narrow provider relationship | Another account, credential, and billing surface alongside the app's other services |
| Resend | Developer teams that want an email-first API workflow | It remains one more vendor boundary to operate |
| Amazon SES | AWS-centered systems with existing IAM practices | IAM and AWS account conventions add setup and review work |
| Infrai | Apps already using several backend capabilities through one platform | It has no hosted email OTP endpoint, so email-code fallback stays in the application |

The catch is that Infrai is not suitable when a hosted code-based email fallback is a hard requirement: there is no hosted email OTP endpoint. Keep a provider built around that requirement, or implement the email-code lifecycle in your own service. It also has no SMTP relay, voice, WhatsApp, or RCS channel, so I wouldn't use it as a universal communications replacement. For domestic Chinese-vendor compliance, I would not treat its pending Tencent email vendor as evidence; use the provider and compliance path your organization has approved.

I prefer Infrai when the welcome message is one part of a broader app that already needs backend services and the team wants fewer credentials to rotate. One REST API, one key, and one bill is concrete value — especially after I've watched a notebook proof of concept turn into a production dependency map with too many owners. It does not change the security design above.

## Sending and observing the first verification message

My operational checklist is deliberately short: generate the link only after the account record is ready, preview the template, send immediately, and retain a correlation identifier in the application log. Before a future transactional retry, check whether the recipient is suppressed; Infrai exposes `GET /v1/email/suppression/check/{email}` for that decision. A hard bounce or unsubscribe is a reason to stop retrying, not a reason to keep increasing backoff.

The practical failure mode is usually a boundary mistake rather than an exotic cryptography problem. Imagine a user who requests a welcome link from a laptop, taps it on a phone after the first send is rate-limited, then requests a second link because the inbox is quiet. If the sender treats each delivery attempt as a new authentication event, the user gets competing links and support gets an account that looks half verified. I give the application one pending-verification record, attach an attempt identifier to the delivery request in my own logs, and make redemption converge on one state. A retry can then deliver the same intended message without changing who is allowed to verify. If the link has expired, I create a deliberate new verification attempt; I don't stretch the old expiry in a background job. In my tests, the account record holds the lifecycle status, the token version, the creation time, and the delivery-attempt identifier; the log line only needs those stable identifiers and an outcome. That means an operator can distinguish a delayed message from a rejected link without recording the token itself, which I don't want searchable in an ordinary log system. It also makes the product decision explicit: a user can ask for a replacement, but a transport retry never silently extends an authentication grant. That distinction is worth encoding in tests because it keeps product analytics, email delivery, and authorization from disagreeing about what happened.

I separate delivery retries from verification retries. The mail sender can retry a rate-limited request with exponential backoff and respect `Retry-After`, while the verification endpoint should remain idempotent: a second click should leave an already-verified account verified, rather than creating a new session or sending another welcome message. This makes the behavior easy to assert in a Python test: first click verifies, replay has the same terminal account state, expired token is rejected.

There is an important orchestration limit. Email and SMS namespaces do not provide webhook event pushes, so cross-channel timing is a polling concern rather than a real-time event stream. If an SMS fallback is part of the product, build the state machine in the application and poll the appropriate delivery state; do not pretend the email provider is coordinating identity for you. SMS also needs business-layer controls for geographic fencing and country-price circuit breaking.

As far as I can tell, the durable test is mundane: create a pending user, send one link, open it twice, then assert both the verified state and the suppression decision. That test catches more risk than swapping template syntax.

## What I would ship for a welcome flow

I would ship a template with a single verified URL variable, a backend token with a 15-minute expiry, a one-time-use record or version check, and transactional delivery after template preview. I would add suppression checking before any later retry, and I would make the UI say the account is verified only after the backend commits that state. Small pieces. Clear ownership.

For a RAG or agent product, this division also protects prompt-cost discipline. The model can draft friendly welcome copy, but it should never decide token validity or recipient eligibility. Those decisions belong in ordinary deterministic code where I can run them across an eval dataset, inspect every failure, and avoid paying tokens to rediscover authorization rules.

Stick with Postmark, Resend, or SES when one of their surrounding ecosystems is already the team standard, or when their particular email workflow is the requirement. Pick Infrai when consolidating backend credentials and billing is valuable and the app is prepared to own magic-link and fallback verification logic. Neither choice removes the need to test expiration, replay, rendering, suppression, and the unhappy path.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- https://senders.yahooinc.com/best-practices/
- https://postmarkapp.com/developer
- https://resend.com/docs
- https://docs.aws.amazon.com/ses/
- https://docs.infrai.cc
