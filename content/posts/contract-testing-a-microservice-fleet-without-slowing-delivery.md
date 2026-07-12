+++
date = '2026-07-12T10:00:00+02:00'
draft = false
title = 'Contract Testing a Microservice Fleet Without Slowing Delivery'
+++
**tl;dr:** End-to-end tests across a microservice fleet are slow, flaky, and force teams to deploy in lockstep. Consumer-driven contract testing replaces the shared staging bottleneck with fast, isolated checks that each service runs on its own. This post lays out an architecture built around a Pact Broker and a `can-i-deploy` gate, and shows how to wire it into CI so it protects integrations without blocking a single pipeline.

---
## Table of contents
- [The Problem: Integration Confidence Doesn't Scale](#the-problem-integration-confidence-doesnt-scale)
- [Why Not More End-to-End Tests](#why-not-more-end-to-end-tests)
- [Consumer-Driven Contracts in One Minute](#consumer-driven-contracts-in-one-minute)
- [The Architecture](#the-architecture)
- [The Consumer Side](#the-consumer-side)
- [The Provider Side](#the-provider-side)
- [The Deploy Gate: can-i-deploy](#the-deploy-gate-can-i-deploy)
- [Why This Stays Fast](#why-this-stays-fast)
- [Versioning and Environments](#versioning-and-environments)
- [Rollout Without a Big Bang](#rollout-without-a-big-bang)
- [Pitfalls](#pitfalls)
- [Wrap](#wrap)

## The Problem: Integration Confidence Doesn't Scale

A single service is easy to test. A fleet of forty services that call each other over HTTP is not. The question every team keeps asking is simple: *if I change this endpoint, whose deploy do I break?*

The usual answer is a shared staging environment and a suite of end-to-end tests. It works for a while. Then it stops. Staging drifts from production, one team's broken deploy blocks everyone else's tests, the suite takes forty minutes, and half the failures are flakes. Delivery slows to the speed of the slowest, flakiest test in the shared environment. That is the exact tax you adopted microservices to avoid.

## Why Not More End-to-End Tests

End-to-end tests are valuable, but they are the wrong tool for *integration confidence at the point of change*. They are:

- **Slow.** They boot real services, real databases, real network hops.
- **Flaky.** Every dependency is a chance to fail for reasons unrelated to your change.
- **Coupled.** They need every participant deployed together, so they force lockstep releases.
- **Late.** They run after merge, in a shared environment, far from the developer who made the change.

You still want a thin layer of end-to-end tests for critical user journeys. You do not want them to be the thing standing between a one-line change and production.

## Consumer-Driven Contracts in One Minute

Contract testing splits the integration test in half so the two services never have to be up at the same time.

- The **consumer** (the caller) writes a test describing exactly the requests it makes and the responses it expects. Running that test against a local mock produces a **contract**: a JSON file of concrete request/response examples.
- The **provider** (the callee) replays that contract against its real code and proves it can satisfy every expectation.

Neither test needs the other service running. The consumer tests against a mock, the provider tests against a recorded contract. The two halves meet only as a file, brokered between them. [Pact](https://pact.io) is the de facto standard for this, and a **Pact Broker** is the service that stores contracts and tracks which versions are compatible.

## The Architecture

```
   ┌─────────────────┐        publishes pact         ┌──────────────────┐
   │  Consumer CI     │  ───────────────────────────> │                  │
   │  (checkout-svc)  │                                │                  │
   │                  │  <── can-i-deploy? ──────────  │   Pact Broker    │
   └─────────────────┘                                │   (PactFlow or   │
                                                       │    self-hosted)  │
   ┌─────────────────┐   webhook: verify this pact     │                  │
   │  Provider CI     │  <───────────────────────────  │   - contracts    │
   │  (payments-svc)  │                                │   - versions     │
   │                  │  ── publishes verification ──> │   - env tags     │
   └─────────────────┘                                └──────────────────┘
             │                                                  │
             │              can-i-deploy? <─────────────────────┘
             ▼
        deploy to prod
```

Three moving parts, and the Broker is the only shared piece of infrastructure:

1. **The Broker** stores every contract, every provider verification result, and a tag for which environment each service version is deployed to. It is the single source of truth for "is version X of A compatible with what's in production of B?"
2. **Consumer pipelines** publish contracts on every build and ask the Broker `can-i-deploy` before releasing.
3. **Provider pipelines** verify contracts, either on their own commits or triggered by a Broker **webhook** when a new contract appears.

The key property: there is no shared runtime environment on the critical path. The Broker exchanges files and verdicts, not live traffic.

## The Consumer Side

The consumer test declares the interaction and generates the pact. Here it is with `pact-python`:

```python
import atexit
import requests
from pact import Consumer, Provider

pact = Consumer("checkout-svc").has_pact_with(
    Provider("payments-svc"), pact_dir="./pacts"
)
pact.start_service()
atexit.register(pact.stop_service)


def test_authorize_payment():
    expected = {"status": "authorized", "reference": "PAY-123"}

    (
        pact
        .given("a customer with a valid card")
        .upon_receiving("an authorization request")
        .with_request(
            method="POST",
            path="/v1/authorizations",
            body={"amount": 4200, "currency": "EUR"},
        )
        .will_respond_with(200, body=expected)
    )

    with pact:
        resp = requests.post(
            f"{pact.uri}/v1/authorizations",
            json={"amount": 4200, "currency": "EUR"},
        )

    assert resp.json()["status"] == "authorized"
```

This runs in milliseconds against a local mock. No `payments-svc` in sight. On success it writes `pacts/checkout-svc-payments-svc.json`, which the pipeline publishes to the Broker, stamped with the consumer's git SHA:

```bash
pact-broker publish ./pacts \
  --consumer-app-version "$CI_COMMIT_SHA" \
  --branch "$CI_COMMIT_REF_NAME" \
  --broker-base-url "$PACT_BROKER_URL" \
  --broker-token "$PACT_BROKER_TOKEN"
```

## The Provider Side

The provider verifies the contract against its real application. The Broker tells it which contracts to fetch, so the provider does not need to know its consumers up front:

```python
from pact import Verifier

verifier = Verifier(
    provider="payments-svc",
    provider_base_url="http://localhost:8080",
)

success, _ = verifier.verify_with_broker(
    broker_url=PACT_BROKER_URL,
    broker_token=PACT_BROKER_TOKEN,
    publish_version=CI_COMMIT_SHA,
    publish_verification_results=True,
    provider_states_setup_url="http://localhost:8080/_pact/provider_states",
)
assert success
```

The one piece of real work on the provider is **provider states**: a test-only endpoint that puts the service into the precondition a contract assumes (`"a customer with a valid card"`), usually by seeding a test database. That is the entire integration seam, and it lives in the provider's own repo where it belongs.

## The Deploy Gate: can-i-deploy

This is where contract testing pays for itself. Before any service deploys, it asks the Broker one question: *given everything already running in production, is my version safe to release?*

```bash
pact-broker can-i-deploy \
  --pacticipant checkout-svc \
  --version "$CI_COMMIT_SHA" \
  --to-environment production \
  --broker-base-url "$PACT_BROKER_URL" \
  --broker-token "$PACT_BROKER_TOKEN"
```

The Broker checks its matrix of contracts and verification results and exits non-zero if this version has any unsatisfied contract against what is currently in `production`. Green means every consumer it talks to, and every provider that depends on it, is verified compatible. Red means the deploy would break someone, and it names who.

No environment was booted to answer that question. It is a database lookup over recorded facts.

## Why This Stays Fast

The whole point is that delivery does not slow down. It stays fast because:

- **No shared environment on the critical path.** Consumer and provider are tested in isolation, in parallel, in their own pipelines. One team's red build never blocks another team's tests.
- **Checks run where the change is.** A contract test fails in the developer's own MR in seconds, not in a nightly end-to-end run nobody reads.
- **Provider verification is decoupled via webhooks.** When a consumer publishes a new contract, a Broker webhook triggers only the affected provider to verify. Nothing else in the fleet rebuilds.
- **The deploy gate is a lookup, not a test run.** `can-i-deploy` reads recorded results. It adds seconds, not minutes.
- **Only what changed gets verified.** Providers verify the contracts that actually reference them, not the entire fleet's suite.

The slow, flaky, lockstep parts are exactly the parts this architecture removes.

## Versioning and Environments

Two conventions make the matrix trustworthy:

- **Version every pacticipant by git SHA.** The Broker reasons about immutable versions, so `--consumer-app-version` and `--publish-version` should be `$CI_COMMIT_SHA`, never a floating tag.
- **Record environments with `record-deployment`.** When a service actually reaches production, tell the Broker:

```bash
pact-broker record-deployment \
  --pacticipant payments-svc \
  --version "$CI_COMMIT_SHA" \
  --environment production
```

Now `can-i-deploy --to-environment production` compares against reality, not a guess. The Broker always knows the exact set of versions live in each environment, which is what makes the compatibility answer correct instead of optimistic.

## Rollout Without a Big Bang

You do not convert forty services in a sprint. Start where the pain is:

1. **Pick one noisy integration.** The pair of services whose shared staging failures cost the most. One consumer, one provider.
2. **Stand up the Broker.** PactFlow if you want it managed, or the open-source Pact Broker on your own cluster. It is a small stateful service with a Postgres backing store, which fits neatly into a Helm/ArgoCD workflow.
3. **Make `can-i-deploy` advisory first.** Run it in pipelines but do not block on it. Let teams see it catch real breakages for a couple of weeks and build trust.
4. **Flip it to blocking** on that first pair, then expand outward one integration at a time. Each new pair is additive and never touches the others.

Because every check is isolated, adoption is incremental by construction. There is no fleet-wide switch to throw.

## Pitfalls

- **Contracts are not schema tests.** Keep them to the fields and behaviors the consumer actually uses. Over-specifying turns every provider refactor into a false failure.
- **Provider states are code.** Treat the setup endpoint like production code, not a test afterthought, or verification becomes flaky for the wrong reasons.
- **Contract testing is not end-to-end testing.** It proves two services agree on a message. It does not prove a business flow works across five of them. Keep a thin end-to-end layer for the journeys that matter most.
- **Forgetting `record-deployment`.** If the Broker does not know what is actually in production, `can-i-deploy` is answering the wrong question.

## Wrap

The failure mode of a growing fleet is not too few tests, it is integration confidence that only exists inside one slow, shared environment. Consumer-driven contracts move that confidence into each service's own pipeline, and a `can-i-deploy` gate turns "will this break someone?" into a fast, boring lookup instead of a forty-minute gamble.

The architecture is small: a Broker in the middle, contracts flowing in from consumers, verifications flowing in from providers, and a deploy gate reading the matrix. That is enough to let every team ship on its own cadence and still sleep at night. Quality built in from the start, not bolted on before release.
