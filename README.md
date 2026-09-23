# GroupMe Scheduled-Send Web App

## The Problem

GroupMe's API has no native way to schedule a message to send later — no `scheduled_at` field, no delayed-send endpoint. If you want a message to go out at a future time, you have to build that entirely yourself: store the message and send time somewhere, then fire it later.

## The Core Design Decision

GroupMe gives you two ways to send a message programmatically:

1. **Messages API** — impersonate the actual user
2. **Bots API** — post as a bot

Impersonating the user means storing their OAuth access token long-term so it's available whenever the scheduled send eventually fires — a real liability, especially since GroupMe's OAuth is implicit-grant only (no refresh token; if revoked, the user has to fully re-authenticate).

The Bots API sidesteps this: once a bot exists for a group, sending through it only requires the bot's ID — no user token needed at send-time. So the user's token is only ever needed briefly, during initial setup, and can be thrown away right after.

## How It Works, End to End

1. User logs into GroupMe via OAuth; the backend gets a temporary access token.
2. User picks a group they're in, and either selects an existing bot or creates a new one for that group.
3. User writes a message and picks a future send time — this gets stored in the database, tied to the bot's ID.
4. The OAuth token is discarded almost immediately after setup — not stored, not kept "just in case."
5. A background scheduler watches the database and, at the right time, fires the message through the bot — no user credentials involved at that point at all.

## The Security Design Underneath It

Rather than relying on a logout button (fragile — closed tabs, crashes, and power loss all leave a lingering token), the token lives only in a short-lived, server-side session with a hard TTL (15–30 min) that expires automatically. It also gets wiped proactively the moment setup finishes successfully. So there's nothing sensitive sitting around waiting to be cleaned up, regardless of how the user's session ends.

## Tech Stack

**Backend**
- Python 3.12
- FastAPI
- Uvicorn

**Database & Storage**
- PostgreSQL (bots, scheduled messages, and doubles as the APScheduler jobstore)
- SQLAlchemy (async, via `asyncpg`)
- Alembic (migrations)
- Redis (short-lived, TTL-based session store for OAuth tokens)

**Scheduling**
- APScheduler with `SQLAlchemyJobStore` (Postgres-backed, survives restarts)

**External API Integration**
- `httpx` (async HTTP client)
- GroupMe OAuth (implicit grant) + Bots API

**Frontend**
- Jinja2 server-rendered templates (possibly HTMX for added interactivity — undecided)

**Containerization**
- Docker + Docker Compose

**Hosting/Deployment**
- Still under evaluation — candidates include Railway, Fly.io, Oracle Cloud Free Tier, and a low-cost VPS (Hetzner/DigitalOcean)

## In Short

A small app that plugs a real gap in GroupMe's API, built around one central constraint: do the job with the least possible standing access to the user's account.