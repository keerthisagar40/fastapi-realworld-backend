# ![RealWorld Example App](.github/assets/logo.png)

> ### Python / FastAPI codebase containing real world examples (CRUD, auth, middlewares advanced patterns, etc.) that adheres to the [RealWorld](https://github.com/gothinkster/realworld) spec and API.

### [Demo](https://demo.realworld.io/)&nbsp;&nbsp;&nbsp;&nbsp;[RealWorld](https://github.com/gothinkster/realworld)

---

## SDET Assignment — Test Coverage Summary

### Why this implementation

This FastAPI + PostgreSQL backend implements the full RealWorld / Conduit spec. I chose it because the stack (async Python, SQLAlchemy, pytest-anyio) is one I'm comfortable owning end-to-end, the app has clean layering (routes → services → repositories) that makes failure modes easy to reason about, and it had no existing test coverage — meaning every test adds signal rather than duplicating what's already there.

---

### What was tested

#### API tests (`tests/api/routes/`)

| File | Coverage |
|---|---|
| `test_registaration.py` | Register happy path; duplicate email / username → 400 |
| `test_login.py` | Login happy path; wrong email, wrong password → 400 |
| `test_user.py` | Get current user; update email, username, bio, password |
| `test_profile.py` | Get profile; follow / unfollow; can't follow self; can't follow twice; can't unfollow if not following; 404 for unknown user |
| `test_article.py` | Create (with/without tags, duplicate tags, same title); read; delete own; can't delete/update foreign article; favorite / unfavorite; can't double-favorite |
| `test_article_filters.py` | Filter by tag, author, favorited; pagination (limit/offset) |
| `test_article_integrity.py` | Deleting an article removes its comments and favorites |
| `test_comments.py` | Create comment; response shape; list with count; unauthenticated list; delete own; deleted comment disappears from list; can't delete another user's comment → 403; comment on non-existent article → 404; GET comments on non-existent article → 404 |
| `test_auth_permissions.py` | Create article / favorite / comment without auth → 403; invalid JWT → 403 |
| `test_validation.py` | Empty title/body/description, empty comment body, missing required fields → 422 |
| `test_tags.py` | Global tag list returns tags present in articles |
| `test_health_check.py` | Health endpoint returns 200 |
| `test_contract.py` | Response shape / field / type / timestamp-format assertions derived from the official RealWorld Postman collection; covers auth, articles (list, create, get, update, favorite, unfavorite, feed), comments, profiles (get, follow, unfollow), tags (17 tests) |

#### E2E flows (`tests/api/routes/test_e2e_flows.py`)

Each test registers users through the API (not fixtures) and drives the full sequence via HTTP calls, matching how a real client behaves.

| Flow | Description |
|---|---|
| `test_full_article_lifecycle` | Register → create article → comment → verify comment in list → delete article → article 404 → comments 404 |
| `test_follow_and_feed_flow` | Alice and Bob register → Bob publishes → Alice follows Bob → Alice's feed contains Bob's article → Alice unfollows → feed empty |
| `test_favorite_count_flow` | Register → create article → favorite → `favoritesCount` increments → unfavorite → count back to 0 |
| `test_cross_user_comment_permissions` | Alice publishes → Bob comments → Alice can't delete Bob's comment (403) → Bob deletes his own comment (204) |

#### Service-layer tests (`tests/services/`)

Unit tests that mock repositories to verify that `ArticleService`, `CommentService`, and `UserService` raise the correct domain exceptions when a repository call fails — decoupled from the HTTP layer.

---

### CI

A GitHub Actions workflow (`.github/workflows/tests.yaml`) runs the full test suite on every push and pull request to `master`. It spins up a PostgreSQL 16 service container, installs dependencies with pip caching, and executes `pytest tests/ -v` with the same environment variables used locally.

---

### What was deliberately left out

- **Frontend / browser tests** — the assignment scope was backend; Playwright E2E covering the React UI would be the natural next step.
- **Performance / load testing** — out of scope for a 3-hour exercise, but worth adding for the feed and article-list endpoints which hit the database with joins.
- **Full CRUD cycle for users via service layer** — covered at the API level; service-layer unit tests were kept focused on failure paths.
- **Rate limiter fix** — the bug is documented and the test workaround is in place, but the production fix (Redis-backed counter) was not implemented as it is out of scope for the assignment.

---

### How I used AI agents

**Tool used:** Claude Code (claude-sonnet-4-6 via the CLI).

**Where it helped:**
- Rapidly surveying the existing test surface and identifying gaps (no comment CRUD tests, no E2E flows) by reading across a dozen files at once.
- Generating the structural scaffold for `test_comments.py` and `test_e2e_flows.py` in line with the existing fixture conventions.
- Catching that `ArticleData` uses `favoritesCount` as the JSON alias (not `favorites_count`) before writing the assertion — saving a run-fail-fix cycle.

**Where it produced something I had to correct:**
- The first draft of `test_user_cannot_delete_another_users_comment` called `create_app()` inside the test body to mint a second HTTP client, which would bypass test isolation (a fresh app, no shared DB session). I rejected this and rewrote it to accept the session-scoped `application` fixture as a parameter, keeping both clients pointed at the same test database.

**One decision where I overrode the agent's suggestion:**
- The agent initially proposed stubbing the second user's token inside a fixture. I overrode this in favor of accepting `auth_token_service` directly in the test — it's simpler, makes the dependency explicit, and avoids hiding DI wiring inside a fixture that would be confusing to read later.

---

### Bugs / observations found

- `test_user_can_create_article_with_existing_title` passes — the implementation allows duplicate titles (they generate unique slugs by appending a suffix). This is intentional per the RealWorld spec but worth a note in docs.
- No enforcement on comment body max length — a 10 000-character comment is accepted with 200 OK. Whether this is a bug depends on product requirements.
- No enforcement on username max length — a 300-character username is accepted with 200 OK. No `max_length` validator on the registration schema's `username` field.
- **Stale slug accessible after title update (genuine defect).** `get_by_slug` in the article repository uses Python `or` instead of SQLAlchemy's `|` operator:
  ```python
  # Bug: Python `or`, not SQL OR
  query = select(Article).where(
      Article.slug == slug or Article.slug.contains(slug_unique_part)
  )
  ```
  In SQLAlchemy 2.x, `bool(BinaryExpression)` evaluates to `False`, so Python's short-circuit `or` always discards the left-hand `slug == slug` equality and the WHERE clause reduces to `WHERE slug LIKE '%<unique_suffix>%'`. Since title updates preserve the unique suffix while changing the slug prefix, the pre-update slug still matches the new row via the LIKE, so the article is reachable under both the old and new slug simultaneously. The fix is to replace `or` with `|` (SQLAlchemy's bitwise-or, which generates a SQL `OR`) or drop the `contains` fallback entirely and use an exact match. Confirmed by inspecting the generated SQL and by the probe test `test_stale_slug_accessible_after_title_update`.
- **Rate limiter is broken at scale (genuine defect).** `RateLimitingMiddleware` stores request counts in a plain Python dict on the middleware instance (`self.request_counts`). This has two real consequences:
  1. **Multi-process deployments:** each worker process holds its own counter, so the effective limit across `N` workers is `N × 100 req/min` per IP — the rate limit is silently ineffective under any real load.
  2. **Test suite:** the `application` fixture is session-scoped (one FastAPI instance for the entire run), so the counter accumulates across all 82 tests. After ~100 requests from the shared `testserver` IP, every subsequent test gets 429 instead of its expected status code — 20 pre-existing tests fail when the suite runs in full. They all pass in isolation, which is how the bug manifests and was discovered.
  The fix is to back the counter with Redis (or another shared store) and use a TTL-based key per IP, making the limit both process-safe and resettable.
  **Test suite workaround:** to prevent the rate limiter from causing false failures in CI, `tests/conftest.py` raises the class-level ceiling to 100,000 for the test session via a session-scoped autouse fixture. This does not fix the production bug — it only prevents it from interfering with unrelated tests. The bug is still present in the middleware code and would affect any real deployment.

---

### What I'd do with more time

1. **Feed pagination** — `limit` / `offset` on `/articles/feed` is not yet covered by tests.
2. **Unicode edge cases in slugs** — non-ASCII titles, emoji, right-to-left characters in slugs are not yet probed.


---

### Bonus — Testing non-deterministic AI features

A data analytics platform will almost certainly expose AI-powered features — summaries, classifications, anomaly explanations, natural-language query results. These cannot be tested with `assert output == expected` because the same input can produce legitimately different outputs on every call. The strategy below is how I'd approach it.

#### Why it's different

Deterministic code has one correct answer. A generative model has a distribution of acceptable answers. Testing shifts from *correctness* to *quality bounds* — you're asserting the output stayed within an acceptable region, not that it matched a string exactly.

#### Testing strategies

**1. Schema / structure validation**
Even if the content varies, the shape of the response should not. Assert every required field exists, has the right type, and falls within expected bounds (e.g. `word_count > 0`, `key_points` is a non-empty list). This is cheap, deterministic, and catches regressions immediately.

**2. Property-based assertions**
Invariants that must hold regardless of what the model says:
- A summary must be shorter than the original article
- A sentiment score must be between 0.0 and 1.0
- A response must not contain PII from the input (email, credit card patterns)
- A translation must preserve the original language's named entities

These are model-agnostic and run in CI like any other test.

**3. Semantic similarity**
Use word-overlap (Jaccard) for a cheap proxy or sentence-transformer embeddings for production-grade checks. Set a *floor* threshold — you're detecting drift to completely off-topic output, not enforcing exact paraphrasing. The threshold should be calibrated against a labelled validation set, not guessed.

**4. Golden-set regression**
A curated set of `(input, min_acceptable_score)` pairs, hand-labelled by domain experts. Run on every deployment. Track scores in an eval dashboard (LangSmith, Weights & Biases, RAGAS) rather than just asserting a hard pass/fail — a gradual score decline over weeks is as important to catch as a sudden drop.

**5. LLM-as-judge**
For complex outputs (multi-step reasoning, structured reports), use a separate, stronger model to evaluate the output against a rubric. More expensive than similarity metrics but handles nuanced quality criteria. Reserve for pre-release eval runs, not every CI push.

#### Practical patterns

- **Mock the LLM in unit and integration tests.** Test your own service logic — truncation, error handling, response parsing — not the third-party model. Real calls are slow, expensive, and non-deterministic.
- **Use real calls only in scheduled eval runs** (nightly or pre-deploy), isolated from the standard test suite.
- **Assert on the prompt, not just the output.** If your service should truncate a long article before sending it to the LLM, assert the prompt the mock received was truncated — not just that the response looks right.
- **Test graceful degradation.** When the LLM times out or returns a 429, the service should raise a clean domain error, not propagate a raw SDK exception to the caller.
- **Track latency and token cost as metrics.** AI calls have SLAs too. A prompt change that doubles token usage is a regression even if quality improves.

#### Implementation

`tests/ai/` contains a working demonstration of all these patterns against a hypothetical `ArticleSummarizer` service (see `tests/ai/summarizer.py` for the feature, `tests/ai/test_ai_summarizer.py` for the tests). No real LLM is called — everything is mocked with `unittest.mock`. The 13 tests cover schema validation, property assertions, semantic similarity, PII detection, token-limit enforcement, graceful failure, and golden-set regression.

---

## Description

This project is a Python-based API that uses PostgreSQL as its database.
It is built with FastAPI, a modern, fast (high-performance), web framework for building APIs with Python 3 based on standard Python type hints.

## Package layout
- `conduit/api`: HTTP layer (routes, schemas, middlewares)
- `conduit/services`: application services/use-cases
- `conduit/interfaces`: interfaces/abstractions (repositories, service contracts)
- `conduit/dtos`: DTOs used across layers
  - `conduit/dtos/domain`: business-level DTOs used by services and API schemas
  - `conduit/dtos/records`: persistence DTOs returned by repositories
- `conduit/infrastructure`: SQLAlchemy models, repositories, migrations
- `conduit/core`: config, logging, security, shared utilities

## Prerequisites
- Python 3.12
- FastAPI
- PostgreSQL
- Pytest
- Docker

## Installation

Create a virtual environment:

```sh
make ve
```

Install dependencies:

```sh
pip install -r requirements.txt
```

## Configuration

Replace `.env.example` with real `.env`, changing placeholders

```
SECRET_KEY=your_secret_key
POSTGRES_USER=your_postgres_user
POSTGRES_PASSWORD=your_postgres_password
POSTGRES_DB=your_postgres_db
POSTGRES_HOST=your_postgres_host
POSTGRES_PORT=your_postgres_port
JWT_SECRET_KEY=your_jwt_secret_key
```

## Run with Docker

You must have `docker` and `docker-compose` installed on your machine to start this application.

Setup PostgreSQL database with docker-compose:

```sh
make docker_build_postgres
```

Run the migrations:

```sh
make migrate
```

Run the application server:

```sh
make runserver
```

Also, you can run the fully Dockerized application with `docker-compose`:

```sh
make docker_build
```

And after that run migrations:

```sh
docker exec -it conduit-api alembic upgrade head
```

## Run tests

Tests for this project are defined in the `tests/` folder.

For running tests, create a separate `.env.test` file (same as `.env` but with a different database name):

```
POSTGRES_DB=conduit_test
```

Then run the tests:

```sh
make test
```

Or run the tests with coverage:

```sh
make test-cov
```

## Run Conduit Postman collection tests

For running tests for local application:

```sh
APIURL=http://127.0.0.1:8000/api ./postman/run-api-tests.sh
```

For running tests for fully Dockerized application:

```sh
APIURL=http://127.0.0.1:8080/api ./postman/run-api-tests.sh
```

## Web routes

All routes are available on `/` or `/redoc` paths with Swagger or ReDoc.
