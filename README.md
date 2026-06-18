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

### What was deliberately left out

- **Frontend / browser tests** — the assignment scope was backend; Playwright E2E covering the React UI would be the natural next step.
- **Performance / load testing** — out of scope for a 3-hour exercise, but worth adding for the feed and article-list endpoints which hit the database with joins.
- **Full CRUD cycle for users via service layer** — covered at the API level; service-layer unit tests were kept focused on failure paths.
- **Rate limiting** — the implementation has a `RateLimitExceededException` class but no active enforcement was observed; would be worth probing once the middleware is wired in.

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
- `RateLimitExceededException` is defined in `conduit/core/exceptions.py` but no middleware or route appears to raise it, suggesting the feature is partially implemented.

---

### What I'd do with more time

1. **CI** — add a GitHub Actions workflow that spins up a PostgreSQL service container, runs migrations, and executes the full test suite on every PR.
2. **Boundary / security probes** — test XSS payloads in article body (confirm they're stored and returned verbatim, not executed), very long usernames/passwords, Unicode edge cases in slugs.
3. **Contract tests** — verify the response shapes against the official RealWorld Postman collection to catch any field-name drift.
4. **Feed pagination** — `limit` / `offset` on `/articles/feed` is not yet covered by tests.


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
