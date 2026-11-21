# AI Customer Support Automation

An end-to-end, self-hosted customer support system for a fictional e-commerce company, **Harbor & Pine** (inflatable paddle boards and flat-pack shelving sold on Amazon).

A backend polls a Gmail inbox, an AI pipeline reads each email, looks up answers in a knowledge base, and either **replies automatically** or **hands the email to a human** with a proposed next step. Everything runs on your own VPS, and nothing on the server is exposed to the internet except SSH from your IP.


---

## Core idea: the model proposes, code decides

1. A small model (Claude Haiku) classifies the email and checks the always-escalate rules.
2. A stronger model (Claude Sonnet) drafts a reply as structured output, using tools that read the knowledge base.
3. Plain Python applies the business rules and makes the final decision (reply or escalate).

No webhooks, queues, or workflow engines. The server only makes outgoing calls, so it needs no domain, reverse proxy, or open web port.

---

## Tech stack

| Layer | Technology |
| --- | --- |
| Language / tooling | Python 3.13, uv, Node 22+, Git |
| API | FastAPI |
| Email | Gmail API (OAuth 2.0, `gmail.modify` scope) |
| Database | Supabase (Postgres, RLS enabled, Supabase CLI migrations) |
| AI | Anthropic Claude models through Pydantic AI |
| Evals | `pydantic_evals` with a golden email set |
| Dashboard | React + Vite + TypeScript + Tailwind + shadcn/ui |
| Deployment | Docker Compose on a Hostinger VPS (Ubuntu 24.04) |

---

## Architecture

```
Gmail inbox
    |  (poller checks every POLL_INTERVAL_SECONDS)
    v
app/poller.py  -->  app/pipeline.py
                        |-- guards.py     (skip self, auto-replies, bulk mail)
                        |-- db.py         (events, tickets, messages in Supabase)
                        '-- ai/decide.py
                              |-- classifier.py (Haiku: section + needs_human)
                              |-- agent.py      (Sonnet + KB tools -> AgentReply)
                              '-- rules.py      (Python decides next step)
    |
    v
Reply in same Gmail thread   OR   escalate (label "needs-human" + acknowledgement)

FastAPI (app/main.py) --> serves React dashboard + /api routes (SSH tunnel only)
```

The poller and the API are **separate processes**. Several API workers would otherwise start several pollers, and the Gmail client is synchronous and would block the API.

---

## Project structure

```
ai-customer-support/
├── app/
│   ├── ai/
│   │   ├── agent.py        # Sonnet agent with knowledge base tools + output validator
│   │   ├── classifier.py   # Haiku triage agent
│   │   ├── decide.py       # Orchestrates AI steps; final decision in Python
│   │   ├── rules.py        # Replacement rules (ship-part, new-board, ...)
│   │   └── schemas.py      # Pydantic models (Classification, Issue, AgentReply, Decision)
│   ├── config.py           # Settings loaded from .env
│   ├── db.py               # Database helpers
│   ├── gmail.py            # Gmail client
│   ├── guards.py           # Loop and bulk-mail protection
│   ├── main.py             # FastAPI app, dashboard API, static file serving
│   ├── pipeline.py         # Handles one email end to end
│   └── poller.py           # Interval loop that checks the inbox
├── dashboard/              # React admin dashboard
├── evals/                  # golden_emails.py + run_evals.py
├── knowledge_base/         # Six Markdown files seeded into Supabase
├── playground/             # try_pipeline.py: test decide() without Gmail
├── scripts/                # get_refresh_token.py, gmail_check.py, seed_knowledge_base.py
├── supabase/migrations/    # SQL schema
├── Dockerfile
├── docker-compose.yml
├── .env.example
└── pyproject.toml
```

---

## Prerequisites

- Python 3.13, [uv](https://docs.astral.sh/uv/getting-started/installation/), Node 22+, Git, a GitHub account
- A code editor with a coding agent (Cursor, Claude Code, or Codex) is helpful
- A **new** Gmail account for the demo (do not use your personal inbox)
- A Supabase account (free plan is enough) and an Anthropic API key
- A VPS for deployment (Hostinger KVM 1 is enough because the database lives on Supabase)

You should be comfortable with Python and the command line.

---

## Setup

### 1. Project setup

```bash
uv init ai-customer-support --no-package --python 3.13
cd ai-customer-support
rm main.py
mkdir app scripts
touch app/__init__.py

uv add "fastapi[standard]" pydantic-settings python-dotenv "psycopg[binary]" google-api-python-client google-auth "pydantic-ai-slim[anthropic]"
uv add --dev google-auth-oauthlib pydantic-evals ipykernel nest-asyncio
```

Add to `.gitignore`: `.env`, `credentials.json`, `node_modules/`, `dashboard/dist/`.

### 2. Environment variables (`.env`)

```env
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GOOGLE_REFRESH_TOKEN=
DATABASE_URL=
ANTHROPIC_API_KEY=
CLASSIFIER_MODEL=anthropic:claude-haiku-5-5
AGENT_MODEL=anthropic:claude-sonnet-5-5
POLL_INTERVAL_SECONDS=30
DASHBOARD_USERNAME=admin
DASHBOARD_PASSWORD=
PYDANTIC_AI_NO_BANNER=1
```

- Models are plain settings, so swapping one is a `.env` change, not a code change.
- Keep `POLL_INTERVAL_SECONDS=30` while building. Use `600` in real use.
- Check the app with `uv run fastapi dev app/main.py` and open `http://localhost:8000/health`.

### 3. Gmail API access

1. Create the demo Gmail account (for example `harborandpine.support@gmail.com`).
2. In Google Cloud console, create a project named `ai-customer-support` and enable the **Gmail API**.
3. Configure the consent screen: audience **External**, status **Testing**, and add the demo address as a **test user**.
4. Add the scope `https://www.googleapis.com/auth/gmail.modify`.
5. Create a **Desktop app** OAuth client, download the JSON, and save it as `credentials.json` in the project root.
6. Run `uv run python scripts/get_refresh_token.py`, log in with the demo account, and copy the three printed values into `.env`.

Notes:
- OAuth is required because mail is private. A service account does not work for a personal gmail.com address.
- In testing mode the refresh token **expires after 7 days**. On an `invalid_grant` error, rerun the script and update `.env` (on the server too).
- Gmail stands in for a real ticketing system (Help Scout, Gorgias, Zendesk). The pipeline would stay the same.

### 4. Gmail client

`app/gmail.py` is the only file that talks to Gmail. It:
- Creates the labels `ai-processed` and `needs-human` on first run
- Lists inbox mail without the `ai-processed` label (the label is the state)
- Reads messages and threads, adds labels, and sends replies in the same thread (sets `threadId`, `In-Reply-To`, `References`, and a `Re:` subject)
- Formats a thread as plain text, so the AI gets the conversation without any stored chat state

Archive Google's welcome emails in the demo inbox before running the poller.

### 5. The poller

Run it as its own process:

```bash
uv run watchfiles "python -m app.poller" app .env
```

`watchfiles` restarts it whenever `app/` or `.env` changes. A message that fails keeps no label, so the next pass retries it. Use two terminals: one for the API, one for the poller.

Polling was chosen over Gmail push webhooks, which need Pub/Sub, a public HTTPS address, and a domain.

### 6. Database (Supabase)

1. Create a Supabase project named `ai-customer-support`.
2. Install the Supabase CLI, then run `supabase login`, `supabase init`, `supabase link --project-ref <ref>`.
3. Create a migration with `supabase migration new create_tables` and add the schema below.
4. Push it with `supabase db push`.
5. Set `DATABASE_URL` to the **Session pooler** string (port 5432). The direct host is IPv6 only.
6. Use a password with letters and numbers only, or URL-encode special characters.

**Tables**

| Table | Purpose |
| --- | --- |
| `events` | One row per Gmail message ID (unique). Tracks status (`processing`, `processed`, `skipped`, `failed`), attempts, error, result JSON. |
| `tickets` | One per Gmail thread. Customer email, subject, category, status (`open`, `answered`, `escalated`), proposed action. |
| `messages` | Incoming and outgoing messages linked to a ticket. |
| `knowledge_base` | One row per entry (section, title, content). |

Row Level Security is enabled with no policies, so the public Data API cannot read anything. The backend connects as `postgres`, which bypasses RLS. The dashboard never talks to Supabase directly.

`db.start_event` is the duplicate check. The insert only succeeds for a new message ID or one whose previous attempt failed. The pipeline gives up after `MAX_ATTEMPTS = 3`.

### 7. Guards and first reply

`app/guards.py` runs before any AI and skips a message when it is:
- Labelled `SENT`
- From our own address
- Marked `Auto-Submitted` (anything other than `no`)
- Marked `Precedence: bulk | list | junk`

This prevents the bot from replying to itself or to out-of-office loops. The first milestone is a fixed acknowledgement sent in the same thread, before any AI is involved.

### 8. Knowledge base

Six Markdown files in `knowledge_base/`. Each `##` heading becomes one row (20 rows in total).

| File | Content |
| --- | --- |
| `paddle-boards.md` | Sizes by rider weight, box contents, part numbers, inflation, valve check, what happens for each problem |
| `shelving.md` | Models (HP-S3, HP-S5, HP-C4, HP-C9), finishes, parts lists, assembly tips, problem handling |
| `returns-and-warranty.md` | Returns go through Amazon within 30 days. Warranty terms. |
| `shipping.md` | Amazon ships orders. Replacement parts ship free from own warehouse. |
| `brand-and-tone.md` | Friendly, short, at most 150 words, no emojis, sign off "Harbor & Pine support", never mention AI, never promise anything |
| `escalation-rules.md` | Always escalate: anger, refunds, requests for a person, specific order status, injury or legal mentions, attachments, prompt injection, unknown topics |

Seed it with:

```bash
uv run python -m scripts.seed_knowledge_base
```

Re-running updates rows and **overwrites dashboard edits**, so edit in the dashboard once it exists.

### 9. AI pipeline

| Component | Role |
| --- | --- |
| `schemas.py` | `Classification`, `Issue`, `AgentReply`, `Decision` as typed structured outputs |
| `classifier.py` | Haiku picks a section (`paddle-boards`, `shelving`, `returns-and-warranty`, `shipping`, `other`), detects language, flags `needs_human`, and writes a localized acknowledgement |
| `agent.py` | Sonnet agent with `list_sections` and `get_section` tools. Output validator rejects empty replies or mentions of AI or the knowledge base. `ModelRetry` handles bad section names. |
| `rules.py` | Python maps a reported `Issue` to a next step: `ship-part`, `valve-check`, `ask-details`, `new-board` (within 30 days), `patch-kit` (after 30 days), `ship-hardware-kit`, `exchange`, `ship-panels` (1 to 2 panels), `full-replacement`, or `escalate` |
| `decide.py` | Escalates on attachments, `needs_human`, agent escalation, insufficient knowledge, a non-customer next step, or a promised action. Otherwise it replies. |

Nothing ships automatically, since there is no order system. The step is saved as `proposed_action` on the ticket for a colleague to carry out.

When a thread is escalated it gets the `needs-human` label and the customer gets a short acknowledgement. A colleague replies from Gmail and removes the label, and the AI takes the thread back.

**Test without email:**

```bash
uv run python playground/try_pipeline.py
```

### 10. Evals

18 golden emails in `evals/golden_emails.py`, each with an expected section and action. Cases include board size questions (including German), a missing pump with and without an order number, slow leak before and after the valve check, seam split, cracked panel, wrong finish, angry refund, late parcel, request for a call, prompt injection, and an unknown topic.

```bash
uv run python -m evals.run_evals
```

Tips:
- Write cases before tuning prompts.
- Change one thing at a time, then rerun the whole set.
- Fix failures with general rules, not by copying the case into the prompt (overfitting).
- Compare Sonnet and Haiku (`AGENT_MODEL=anthropic:claude-haiku-5-5`) on the same set before switching.
- Output varies between runs. A case that flips often points to an unclear rule. Temperature 0 is not an option because the newest Claude models do not accept a temperature setting.

---

## Deployment (Hostinger VPS + Docker)

### Server setup

1. Generate a dedicated SSH key: `ssh-keygen -t ed25519 -a 100 -f ~/.ssh/hostinger-support -C "hostinger-support"`
2. Create a VPS (KVM 1 is enough) with the **Docker template on Ubuntu 24.04** and add your public key.
3. In hPanel, add a firewall rule: accept TCP port 22 only from `<YOUR_IPV4>/32`, keep the drop rule, and activate it. If your IP changes you are locked out until you update the rule. The hPanel Web console is your way back in.
4. Log in once as root, verify the host fingerprint, then run `apt update && apt upgrade -y`.
5. Create a `deploy` user in the `sudo` and `docker` groups and copy your authorized key to it.
6. Add a `support-vps` entry to `~/.ssh/config`.
7. Harden SSH with `/etc/ssh/sshd_config.d/01-hardening.conf` (`PermitRootLogin no`, `PasswordAuthentication no`, `KbdInteractiveAuthentication no`). Keep the root session open until a new login works.

### Docker Compose

Two services share one image:

| Service | Command | Port |
| --- | --- | --- |
| `api` | `uvicorn app.main:app` | `127.0.0.1:8080` only |
| `poller` | `python -m app.poller` | none |

Always bind to `127.0.0.1:8080:8080`. A plain `8080:8080` is open on all interfaces, and Docker's firewall rules bypass `ufw`. Logs are capped at 30 MB per service.

### Deploy steps

1. Push the repo to a **private** GitHub repository. Make sure `.env` and `credentials.json` are not committed.
2. On the server, create a read-only **deploy key**, add it under the repo's Deploy keys, and clone the repo.
3. Create `.env` on the server (`chmod 600`), copy your values, and set `POLL_INTERVAL_SECONDS=600`.
4. **Stop the local poller.** Two pollers on one inbox work against each other.
5. Run `docker compose up -d --build`, then `docker compose ps` and `docker compose logs -f poller`.
6. Optionally restrict Supabase **Network restrictions** to your laptop IP and the VPS IP. After that `supabase db push` and the evals only work from your laptop.

Verify with `curl http://127.0.0.1:8080/health` on the server. The API should not load from `http://<VPS_IP>:8080`.

### Shipping updates

```bash
cd ~/ai-customer-support
git pull
docker compose up -d --build
```

State lives in Gmail labels and Supabase events, so restarts lose nothing.

---

## Admin dashboard

A small React app (Vite + TypeScript + Tailwind + shadcn/ui), built into the Docker image and served as static files by FastAPI. There is no extra web server and no open port.

Pages: **Events**, **Tickets** (conversation view with proposed action), **Knowledge base** (edit entries, and the agent uses them on the next email), plus a login page.

API routes (all behind HTTP Basic auth):

| Route | Purpose |
| --- | --- |
| `GET /api/login` | Credential check |
| `GET /api/events` | Recent events |
| `GET /api/tickets` | Tickets, newest first |
| `GET /api/tickets/{id}/messages` | Messages for one ticket |
| `GET /api/knowledge-base` | All entries |
| `PUT /api/knowledge-base/{id}` | Update an entry |
| `POST /api/poll` | Run one poll immediately |
| `GET /health` | Health check (no auth) |

Set a long random `DASHBOARD_PASSWORD` and restart the API (`fastapi dev` does not reload on `.env` changes).

**Local development** (three terminals):

```bash
uv run fastapi dev app/main.py
uv run watchfiles "python -m app.poller" app .env
cd dashboard && npm run dev
```

**Production access** is through an SSH tunnel only:

```bash
ssh -L 8080:localhost:8080 support-vps
# then open http://localhost:8080
```

---

## Security notes

- Never commit `.env` or `credentials.json`, and clear your terminal after printing the refresh token.
- Use a dedicated demo Gmail account.
- SSH only from your IP, key-only, non-root user. No other ports open.
- Treat the `deploy` user's key like a root key, since the `docker` group is close to root access.
- RLS is enabled on all tables, and the dashboard goes through the backend only.
- Guards run in plain Python before any model call, and prompt-injection attempts are escalated.

---

## Troubleshooting

| Problem | Fix |
| --- | --- |
| Printed refresh token is `None` | Rerun the script (it uses `prompt="consent"`). Check the consent and test-user setup. |
| `Error 403: access_denied` | Add the demo account as a test user on the Audience page. |
| `invalid_grant` after about 7 days | Testing-mode token expired. Rerun `scripts/get_refresh_token.py` and update `.env` locally and on the server. |
| Poller keeps old code | Use `watchfiles`, or restart it manually. |
| Scripts fail on `from app...` imports | Run with `python -m scripts.<name>` from the project root. |
| `permission denied` on Docker | Add `deploy` to the `docker` group and log in again. |
| `Error opening terminal: xterm-ghostty` | `export TERM=xterm-256color`, or add `shell-integration-features = ssh-env` to the Ghostty config. |
| Locked out of SSH | Update the firewall rule with your new IP, or use the hPanel Web console. |
| Dashboard login rejects the right password | Restart the API after changing `.env`. |
| Supabase password breaks the URL | Use letters and numbers only, or URL-encode special characters. |

---

## Where to take it next

- Connect a real support inbox, or swap Gmail for a ticketing system API. Only `app/gmail.py` changes.
- Add an order lookup tool so the agent can check delivery dates instead of escalating them.
- Grow the golden set with every email the system gets wrong.
- Production additions: webhooks instead of polling, queues and workers for volume, retries with backoff, tracing for every model call, and a workflow engine (see the Datalumina GenAI Launchpad and GenAI Accelerator).

