# Code Review Agent

> Multi-agent AI that reviews GitHub PRs for security, performance, and correctness | in under 60 seconds.

> [!NOTE]
> README-only for now. Implementation coming soon.

[![Python](https://img.shields.io/badge/Python-3.11+-3776AB?style=flat&logo=python&logoColor=white)](https://python.org)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.110+-009688?style=flat&logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## What It Does

When a pull request opens on your repository, three specialised AI agents run **in parallel** and post inline review comments directly to the PR | before a human reviewer even opens the diff.

```
PR opened on GitHub
        |
        v
  Webhook -> FastAPI
        |
        v
   Fetch diff via GitHub API
        |
        v
   asyncio.gather()
        |
   .----+-------------------+--------------------.
   |                        |                    |
   v                        v                    v
Security Agent       Performance Agent     Correctness Agent
OWASP Top 10         N+1 queries           Logic inversions
Secrets              O(n2) loops           Null dereference
SQL Injection        Blocking I/O          Unhandled errors
Auth bypass          Memory leaks          Off-by-one
   |                        |                    |
   '----+-------------------+--------------------'
        |
        v
     Aggregator
  deduplicate | rank | format
        |
        v
   GitHub API: post PR review
   Inline comment per finding
   Summary: 3 CRITICAL | 2 HIGH | 4 LOW
        |
        v
   < 60 seconds total
```

---

## Features

- **Three parallel agents** | security, performance, and correctness run simultaneously, not sequentially
- **Inline comments** | each finding links directly to the line it found the issue on
- **Severity ranking** | CRITICAL / HIGH / MEDIUM / LOW with colour-coded summary
- **Under 60 seconds** | parallel async execution; a 200-line diff costs under $0.01 to review
- **GitHub App or webhook** | works with any public or private repository
- **Language-agnostic** | Python, TypeScript, Go, Java, any language the LLM can read

### Security Agent Checks
- OWASP Top 10 (injection, broken auth, XSS, IDOR, etc.)
- Hardcoded secrets, API keys, credentials
- SQL / command / path injection patterns
- Insecure dependency versions matched against known CVEs
- Auth bypass patterns and missing access controls

### Performance Agent Checks
- N+1 query patterns in ORM code
- O(n²) or worse loops in hot paths
- Unbounded memory growth (infinite appends, unclosed streams)
- Blocking I/O called inside async functions
- Missing database indexes inferred from query patterns

### Correctness Agent Checks
- Off-by-one errors and fence-post bugs
- Null / None dereference without guards
- Unhandled exception paths
- Logic inversions (`>` vs `>=`, `and` vs `or`)
- Missing edge cases (empty list, zero, negative input)
- Type mismatches in dynamically typed code

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Backend | FastAPI (Python 3.11+) |
| Agents | OpenAI `gpt-4o` (security/correctness) | `gpt-4o-mini` (performance) |
| GitHub integration | GitHub App + Webhooks + PyGithub |
| Database | PostgreSQL (review logs, repo configs) |
| Frontend | Next.js + Tailwind CSS |
| Deploy | Railway / Render (free tier) |

---

## Project Structure

```
code-review-agent/
├── api/
│   ├── main.py                  # FastAPI app, lifespan, middleware
│   ├── github_client.py         # GitHub API wrapper (fetch diff, post review)
│   ├── routes/
│   │   └── webhook.py           # POST /webhook/github | HMAC-verified
│   └── agents/
│       ├── security.py          # Security analysis agent
│       ├── performance.py       # Performance analysis agent
│       ├── correctness.py       # Correctness analysis agent
│       └── aggregator.py        # Merge, deduplicate, rank findings
├── frontend/
│   └── pages/
│       ├── index.tsx            # Landing page
│       └── demo.tsx             # Live demo: paste code, see review
├── db/
│   └── schema.sql
├── .env.example
├── docker-compose.yml
└── requirements.txt
```

---

## Setup

### 1. Clone and Install

```bash
git clone https://github.com/TDahiya/code-review-agent
cd code-review-agent
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
```

### 2. Create a GitHub App

1. Go to **GitHub Settings | Developer settings | GitHub Apps | New GitHub App**
2. Set webhook URL to `https://your-domain.com/webhook/github`
3. Permissions: **Pull requests** (read & write), **Contents** (read)
4. Subscribe to: `Pull request` events
5. Generate and download a private key

### 3. Configure Environment

```bash
cp .env.example .env
```

```env
GITHUB_APP_ID=your_app_id
GITHUB_PRIVATE_KEY_PATH=./github_private_key.pem
GITHUB_WEBHOOK_SECRET=your_webhook_secret
OPENAI_API_KEY=your_openai_key
DATABASE_URL=postgresql://user:pass@localhost:5432/reviewagent
```

### 4. Run

```bash
docker-compose up -d db
psql $DATABASE_URL < db/schema.sql
uvicorn api.main:app --reload --port 8000
```

---

## API Reference

### `POST /webhook/github`

Receives GitHub webhook events. Verified via HMAC-SHA256 in `X-Hub-Signature-256` header.

**Response:** `202 Accepted` | review runs asynchronously.

### `POST /review/demo`

Run a review against a code snippet directly (no GitHub required).

**Request:**
```json
{
  "code": "def get_user(id):\n    return db.execute(f'SELECT * FROM users WHERE id={id}')",
  "language": "python"
}
```

**Response:**
```json
{
  "findings": [
    {
      "agent": "security",
      "severity": "CRITICAL",
      "line": 2,
      "title": "SQL Injection",
      "description": "User-controlled input interpolated directly into SQL query.",
      "suggestion": "Use parameterised queries: db.execute('SELECT * FROM users WHERE id=?', (id,))"
    }
  ],
  "summary": { "critical": 1, "high": 0, "medium": 0, "low": 0 },
  "duration_ms": 1840
}
```

---

## Cost Estimate

| PR Size | Estimated Cost |
|---------|---------------|
| Small (< 100 lines) | ~$0.003 |
| Medium (100-500 lines) | ~$0.008 |
| Large (500+ lines) | ~$0.015 |

All three agents run in parallel | wall time is the slowest single agent, typically 8-15 seconds.

---

## Roadmap

- [ ] `.github/code-review-agent.yml` config per repo (disable specific checks)
- [ ] Auto-approve PRs that score zero findings
- [ ] Slack / Teams notification on CRITICAL findings
- [ ] Fine-tuned model on accepted/rejected code review data

---

## Author

**Tanishq Dahiya** | [linkedin.com/in/tdahiya2845](https://linkedin.com/in/tdahiya2845) | [github.com/TDahiya](https://github.com/TDahiya)

MSc Computing (AI), First Class Honours | Dublin City University

---

## License

MIT
