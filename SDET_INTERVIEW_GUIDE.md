# SDET Assignment — Decision Journal & Interview Guide

This document tells the full story of what was built, why each decision was made, where Claude went wrong and how it was corrected, and how to explain every crossroad to an interviewer.

---

## 1. Choosing the implementation

**The assignment gave a choice of RealWorld implementations.** Several options existed — Go, Node, Django, Rails, FastAPI.

**Decision: FastAPI + PostgreSQL**

**Reasoning:**
- The stack (async Python, SQLAlchemy 2.x, pytest-anyio) is familiar enough to own the debugging, not just the writing
- The app had zero existing tests — every test written adds new signal rather than duplicating existing coverage
- Clean layering: routes → services → repositories means failures can be isolated to a layer
- The async nature (asyncpg driver) makes it realistic and slightly tricky — a good demonstration surface

**How to explain to interviewer:**
> "I chose a stack I could fully own. The goal wasn't to pick the easiest option — it was to pick one where I could reason about failures at every layer, write meaningful tests, and actually debug when something went wrong. Zero existing test coverage meant every test I wrote was genuinely additive."

---

## 2. Understanding the test infrastructure before writing anything

**Before writing a single test, read all existing fixtures in `tests/conftest.py`.**

Key discoveries:
- `application` fixture is **session-scoped** — one FastAPI instance for the entire test run
- `create_tables` is **autouse + function-scoped** — tables created and dropped between every test (isolation)
- `session`, `test_user`, `test_article` are **function-scoped** — fresh data per test
- Tests run via Docker because there's no local Python 3.12

**Decision: Work within the existing fixture conventions, not around them**

**Reasoning:** Introducing new fixture patterns would create two competing conventions. The existing setup is clean — follow it.

**How to explain to interviewer:**
> "Before writing any test I read the full conftest to understand the fixture lifecycle. The key insight was that the app is session-scoped but tables are dropped between tests — so every test starts with a clean DB but shares the same app instance. That matters for rate limiting, as we later discovered."

---

## 3. Multi-user test — the first major Claude correction

**The task:** Test that User A cannot delete User B's comment.

**What Claude produced first:**
```python
async def test_user_cannot_delete_another_users_comment(...):
    app2 = create_app()  # NEW APP INSIDE THE TEST
    async with AsyncClient(app=app2, ...) as other_client:
        ...
```

**Why this was wrong:**
- `create_app()` inside a test creates a brand new FastAPI application
- This new app has its own DI container pointing to a different database connection
- The two clients are no longer looking at the same database state
- Tests that share state would silently fail or pass for wrong reasons

**The fix:**
```python
async def test_user_cannot_delete_another_users_comment(
    application: FastAPI,  # USE THE SESSION-SCOPED FIXTURE
    authorized_test_client: AsyncClient,
    auth_token_service: IAuthTokenService,
    ...
):
    # Mint a token for the second user using the shared DI container
    other_token = auth_token_service.generate_jwt_token(user=other_user)
    async with AsyncClient(app=application, ...) as other_client:
        ...
```

**How to explain to interviewer:**
> "Claude's first draft called `create_app()` inside the test body. I caught this immediately — it would create an isolated app with its own DB connection, breaking test isolation. The fix was to accept the session-scoped `application` fixture as a parameter, ensuring both clients share the same database state. This is a subtle but critical distinction in async test design."

---

## 4. auth_token_service vs fixture stubbing — second override

**Claude's suggestion:** Create a new fixture that stubs the second user's token.

**My decision:** Accept `auth_token_service` directly in the test instead.

**Reasoning:**
- A fixture wrapping token generation hides the dependency — future readers don't know where the token comes from
- `auth_token_service` is already a fixture in conftest — using it directly is explicit and traceable
- One less fixture to maintain

**How to explain to interviewer:**
> "The agent wanted to hide token minting inside a helper fixture. I preferred injecting `auth_token_service` directly into the test — it makes the dependency visible, keeps the test self-documenting, and avoids adding infrastructure that only one test needs."

---

## 5. Discovering the stale slug bug

**How it was found:** While writing update-article tests, noticed the slug in the response changed after a title update. Wrote a probe test to check if the old slug still worked — it did.

**Root cause investigation:** Read `conduit/infrastructure/repositories/article.py`:
```python
query = select(Article).where(
    Article.slug == slug or Article.slug.contains(slug_unique_part)
)
```

**The bug:** Python `or` is not SQL `OR`. In SQLAlchemy 2.x, `bool(BinaryExpression)` evaluates to `False`. So Python's short-circuit `or` always discards the left side. The effective SQL is always:
```sql
WHERE slug LIKE '%suffix%'
```

Since title updates preserve the unique suffix, the old slug still matches via the LIKE clause.

**The fix (not implemented — out of scope):**
```python
# Replace `or` with `|` for SQLAlchemy's SQL OR
query = select(Article).where(
    Article.slug == slug | Article.slug.contains(slug_unique_part)
)
```

**How to explain to interviewer:**
> "I found this by probing update behaviour — the slug changes on title update but the old slug still resolved. I traced it to a Python operator precedence bug: `or` in SQLAlchemy 2.x silently reduces to just the LIKE clause because `bool(BinaryExpression)` is False. A one-character fix (`or` → `|`) would resolve it. I documented it with a probe test instead of fixing it — fixing production bugs wasn't the scope of the assignment."

---

## 6. Discovering the rate limiter bug

**How it was found:** Tests passed individually but failed with 429 when the full suite ran.

**Root cause:** `RateLimitingMiddleware` stores request counts in `self.request_counts` — a plain Python dict on the **instance**. Because `application` is session-scoped (one instance for all tests), the counter accumulates across the entire run. After 100 requests from `testserver`, every subsequent test gets 429.

**Two real consequences:**
1. Multi-worker production: each worker has its own counter → effective limit is N × 100 per IP
2. Test suite: counter never resets → tests fail in bulk after threshold

**The workaround (not a fix):**
```python
@pytest.fixture(scope="session", autouse=True)
def _raise_rate_limit_for_tests():
    from conduit.api.middlewares import RateLimitingMiddleware
    RateLimitingMiddleware.rate_limit_requests = 100_000
    yield
    RateLimitingMiddleware.rate_limit_requests = 100
```

**The real fix (documented, not implemented):** Redis-backed counter with TTL-keyed entry per IP.

**How to explain to interviewer:**
> "The rate limiter bug only manifested when tests ran together, not in isolation — a classic shared-state problem. I identified it by observing that the 20 failing tests all returned 429, and they all passed when run alone. The root cause was an in-memory dict on the middleware instance. The workaround raises the ceiling for tests; the production fix is Redis. I documented both and kept them separate — the workaround is in conftest, the bug is in the README."

---

## 7. Contract tests — three things Claude got wrong

**After writing 15 contract tests, the user asked "are these proper?" — critical review found three issues.**

### Issue 1: `commentsCount` asserted as a contract field

**What Claude asserted:**
```python
assert "commentsCount" in body
assert isinstance(body["commentsCount"], int)
```

**Why this was wrong:** The official RealWorld spec returns `{"comments": [...]}` only. `commentsCount` is an implementation extension. Asserting it in a *contract* test claims it's part of the spec — it isn't.

**Fix:** Removed the assertion, added a comment explaining why.

### Issue 2: Feed test passed vacuously

**The code:**
```python
for article in body["articles"]:
    assert_article_shape(article)
```

**The bug:** `authorized_test_client`'s user follows nobody → feed is always empty → loop never executes → `assert_article_shape` is never called → the test proves nothing.

**Fix:** Register a second user in the test, have them publish an article, follow them, then check the feed. Added `assert len(body["articles"]) > 0` before the loop.

### Issue 3: Missing DELETE response shapes

The Postman collection validates responses for:
- `DELETE /articles/{slug}/favorite` → returns article
- `DELETE /profiles/{username}/follow` → returns profile

Neither was tested. Both added.

**How to explain to interviewer:**
> "When asked to review the contract tests critically, I found three problems: asserting a non-spec field as a contract requirement, a vacuous test that passed because its loop never ran, and missing coverage for DELETE endpoints that return bodies. The vacuous test is the most dangerous kind — it gives you green CI while proving nothing. The fix was to explicitly assert the collection is non-empty before iterating."

---

## 8. CI workflow — design decision

**Choice:** Use GitHub Actions postgres service container vs docker compose in CI.

**Decision: postgres service container + pip install directly**

**Reasoning:**
- No Docker build step in CI → faster
- `services:` block is the idiomatic GitHub Actions pattern
- Matches how other Python projects structure CI
- Only difference from local: `POSTGRES_HOST=localhost` instead of `postgres`

**The problem that came up:** CI failed because ~30 tests got 429 — more than the ~20 we saw locally. The test ordering in CI (alphabetical by file) hit the rate limit at a different point than local.

**Fix:** The session-scoped autouse fixture raising the rate limit ceiling (see section 6).

**How to explain to interviewer:**
> "I chose the postgres service container approach over running docker compose in CI because it's faster — no image build step. The only env var difference is the hostname. The rate limiter issue in CI was a surprise because local runs only showed ~20 failures, but CI's alphabetical test ordering hit the threshold at a different point and caused more. That's what prompted the conftest workaround."

---

## 9. AI testing bonus — design decisions

**The task:** Demonstrate how to test non-deterministic AI features.

**Key decisions:**

**Mock everything — no real LLM calls in tests**
- Real calls are slow, expensive, non-deterministic
- You're testing YOUR code's logic — truncation, error handling, response parsing — not OpenAI's model
- Real calls belong in scheduled eval runs (nightly/pre-deploy), isolated from CI

**Use Jaccard word overlap as the similarity metric**
- No extra dependencies (built from Python sets)
- Good enough to demonstrate the concept
- Production equivalent: sentence-transformer embeddings or cosine similarity on TF-IDF vectors

**Separate PII detection from PII prevention**
- The service doesn't sanitize — it passes through model output
- Testing that the service sanitizes something it doesn't sanitize would be wrong
- Instead: test that the detector catches leaky output (unit test of the guardrail), and separately test that a clean model output passes the check

**Assert on the prompt, not just the output**
- Token-limit enforcement test checks what the mock received, not what it returned
- This tests your truncation logic, which is the thing you control

**How to explain to interviewer:**
> "The core insight is that AI testing shifts from correctness to quality bounds. You can't assert equality — you assert invariants: shape is right, length is reasonable, no PII leaked, semantically related to input. Mock the LLM in CI; use real calls in eval runs on a schedule. And always test the prompt you sent, not just the response you got — that's where your own logic lives."

---

## 10. What was left out and why

| Skipped | Reason |
|---|---|
| Redis-backed rate limiter fix | Production code change; out of scope for the assignment |
| Feed pagination tests | Low value at this stage; coverage already comprehensive |
| Frontend / Playwright tests | Assignment scope was backend API only |
| Performance / load testing | Out of scope for a time-boxed exercise |
| Unicode slug edge cases | Nice-to-have; functional coverage was the priority |

**How to explain to interviewer:**
> "I was deliberate about scope. The assignment is time-boxed, so I prioritised breadth (every major feature tested) over depth (every edge case). The things left out are documented — I can tell you exactly what's missing and why, which is more useful than pretending the coverage is complete."

---

## 11. Summary of Claude corrections / hallucinations

| What Claude did | Why it was wrong | What was done instead |
|---|---|---|
| Called `create_app()` inside a test | New app = different DB connection, breaks isolation | Used session-scoped `application` fixture |
| Suggested token stub inside a fixture | Hides dependency, adds unmaintained infrastructure | Injected `auth_token_service` directly |
| Asserted `commentsCount` as a contract field | Not in the RealWorld spec — it's an implementation extension | Removed assertion, added explanatory comment |
| Feed contract test that never executed its loop | Empty feed → vacuous pass, proves nothing | Register + follow a second user to populate feed |
| Missing DELETE response shape tests | Unfavorite and unfollow both return bodies per Postman spec | Added both tests |
| First regex in docstring used unescaped `\d` | Python `SyntaxWarning` — invalid escape sequence | Escaped in docstring using `\\d` |

---

## 12. One-line answers for common interview questions

**"How did you decide what to test first?"**
> Read the conftest to understand the fixture model, then mapped it to the RealWorld spec endpoints. Tested each resource in isolation (unit-style) before cross-resource flows.

**"How do you test without a frontend?"**
> `httpx.AsyncClient` with `transport=ASGITransport(app=...)` — drives the real FastAPI app in-process, no network needed, full DB.

**"How do you ensure test isolation?"**
> Function-scoped `create_tables` fixture drops and recreates all tables between tests. Each test starts with a clean schema. The session-scoped app instance is shared but the data is not.

**"What's the difference between your E2E tests and your API tests?"**
> API tests use pre-seeded fixtures and test one endpoint at a time. E2E tests register real users through the API and chain multiple HTTP calls — simulating a real client session from start to finish.

**"You found bugs — did you fix them?"**
> No, deliberately. Fixing production bugs was out of scope. I documented them with probe tests and README entries so they're visible and reproducible. A failing test that documents a known bug is more honest than silence.

**"What would you do differently?"**
> Add a Redis-backed rate limiter from the start so the test workaround isn't needed. Also write the contract tests before the implementation tests — contract first forces you to read the spec carefully before making assumptions about the response shape.
