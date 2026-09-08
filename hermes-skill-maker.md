---
name: hermes-create-skill
description: "Bootstrap a new Hermes skill from an API doc URL or a functional brief, following the project's router + specialized-skills architecture. Handles category creation, shared api_client scaffolding, SKILL.md drafting with the right frontmatter, Python CLI generation, propagation to the container, and smoke tests. Use when the user asks to add a new capability to Hermes (a new API integration, a new domain, a new tool)."
---

# Build a new Hermes skill — router + specialized skills

This skill guides you to **create a new Hermes skill** in this project
(`config/skills/`) that mirrors the architecture we've established across
the `sports/` and `nsn-ops/` categories.

**Never build a single monolithic skill.** Always follow the router +
specialized-skills pattern. Every new capability is a category, or a
specialized skill inside an existing category.

## Language rule — ENGLISH ONLY

**Every artefact you produce for a Hermes skill MUST be in English.**
Non-negotiable. This applies to:

- `SKILL.md` — frontmatter (`description`, `tags`) AND body (headings,
  When to use / NOT to use bullets, CLI help text, Examples, Gotchas).
  Only exception: user phrasings quoted as trigger examples MAY appear in
  the user's language when the users prompt in a non-English language
  (e.g., `"combien de matchs aujourd'hui"`) — but wrap them in quotes as
  QUOTED user input, keep every surrounding sentence in English.
- Python code — all comments, docstrings, variable names, function names,
  argparse `help=` strings, `--flag` names, `choices=`.
- User-facing script output — table headers, error messages,
  `(no rows)` / `(no matches)` fallbacks, JSON keys.
- `README.md` and `DESCRIPTION.md`.

Rationale: the LLM matches the SKILL.md description in English against
its own English-first tokenizer regardless of the user's UI language;
mixed-language content degrades routing and increases context cost. Our
Hermes runtime is English; user replies get translated at the LLM layer.

**Do NOT copy the French messages or comments from the earlier `sports/`
and `nsn-ops/` skills** (they're the historical accretion, kept as-is
for now). New skills you produce start fresh in English.

## Input types

The user will invoke you with one of two starting points:

- **API doc URL / OpenAPI spec / SDK reference**: fetch it, map endpoints
  to sub-domains, propose the architecture.
- **Functional brief** ("I want Hermes to query Zendesk tickets"): infer
  the sub-domains, find the API doc online, ask for auth details.

Your first action after loading this skill:
1. Determine which input type the user has given.
2. Ask ONE round of clarifying questions if key info is missing (auth
   method, base URL, main use cases, whether it fits in an existing
   category).
3. Present a **concise architecture plan** and wait for go/no-go before
   coding.

## Architecture recap — what you MUST produce

Every category under `config/skills/<category>/` has the SAME shape:

```
config/skills/<category>/
├── README.md                          ← dev doc (skip on first pass if minor)
├── DESCRIPTION.md                     ← Hermes-facing category summary
│
├── <category>-overview/               ← ROUTER + shared code
│   ├── SKILL.md                         → decision table + endpoint refs
│   └── scripts/
│       ├── api_client.py                → HttpClient + <Service> classes (stdlib only)
│       └── config.py                    → URLs + tokens (env vars + defaults)
│
├── <category>-<domain-A>/             ← specialized skill (1 SKILL.md + 1 script)
├── <category>-<domain-B>/
├── <category>-<domain-C>/
└── ...
```

Reference implementations:
- `config/skills/sports/` — 14 skills covering football/betting APIs
- `config/skills/nsn-ops/` — 4 skills covering internal GitLab/Sentry/ES/Metabase

**Read the corresponding `README.md` and `sports-overview/SKILL.md` (or
`nsn-ops-overview/SKILL.md`) before writing anything new** — they codify
the conventions.

## Granularity — how to split into specialized skills

Rule of thumb: **one specialized skill per sub-domain of user questions**.

Examples of good splits:
- Football: `sports-fixtures` (schedule), `sports-live` (scores), `sports-odds`
  (betting lines), `sports-standings` (league table), `sports-player`
  (stats/top scorers), `sports-team` (form/H2H), `sports-transfers`,
  `sports-coach`, `sports-injuries`, `sports-referee`, `sports-search`,
  `sports-insights` (value bets/arbitrage), `sports-meta` (provider metadata).
- NSN Ops: `nsn-gitlab` (MRs/issues/projects), `nsn-sentry` (errors),
  `nsn-elasticsearch` (logs), `nsn-metabase` (BI + ad-hoc SQL).

Bad splits (do NOT do):
- One skill per HTTP endpoint (too granular)
- One skill for the whole API (defeats routing)
- Skills organized by output type instead of user intent

**Target: 4-10 specialized skills per category.** Fewer → merge; more →
split into two categories.

## Description separation — non-negotiable

The LLM chooses which skill to load by **semantic matching on the
`description` frontmatter alone**. If two skills in the same category
share keywords, Hermes routes randomly between them → wrong data half
the time. Descriptions MUST be **mutually exclusive** within a category.

### The 3 layers

| Layer                   | Description shape                            | Example |
|-------------------------|----------------------------------------------|---------|
| **Router**              | INTENTIONALLY BROAD — catches every keyword the domain uses, ends with "Load this FIRST" | "Router + shared HTTP clients for all NSN internal ops questions (GitLab merge requests / issues / projects, Sentry errors, Elasticsearch logs, Metabase dashboards + questions). Load this FIRST when the user asks anything about our GitLab, Sentry, logs, or Metabase." |
| **Specialized skill**   | NARROW — only its own sub-domain, explicit "NOT" for adjacent scopes | "Read-only Sentry (sentry.n10.xyz) — list org projects, list unresolved errors by project, get issue details + latest event stack trace. Use for 'erreurs Sentry', 'stack trace de X'. NOT for raw logs (see /nsn-elasticsearch)." |
| **`When NOT to use`** in body | Points explicitly to sibling skills for adjacent intents — reinforces exclusivity in-context | "- Raw logs (no exception) → `/nsn-elasticsearch`" |

Router description overlaps siblings on purpose (it's the front door).
**Sibling descriptions must NOT overlap each other.**

### Rules for specialized skill descriptions

1. **Own ONE sub-domain per skill.** Never write "everything about X" —
   that competes with the router and steals its routing role.
2. **List 3-5 concrete user phrasings** in the user's spoken language
   (French + English if the users mix). These become the LLM's primary
   matching signal.
3. **Name adjacent-but-different scopes with a "NOT" clause** in the
   description itself. Example: `"...NOT for player search (see
   /sports-player)."`
4. **Match verbs to the skill's action**: "list", "get", "search", "count",
   "compare" — never generic verbs like "handle", "work with", "manage".
5. **Include the data source** when disambiguating (e.g., "SportMonks
   only" vs "Odds-API multi-sport"). If two skills query different
   providers for overlapping domains, that IS the differentiator.

### Anti-patterns (would break routing)

- Two skills claiming the same keyword: `sports-player: "top scorers"`
  AND `sports-standings: "top scorers"` → LLM flips a coin.
- Specialized skill with router-like breadth: `nsn-gitlab: "all GitLab
  operations including issues, MRs, projects, pipelines, commits,
  users..."` → competes with `nsn-ops-overview`.
- Vague verbs: `sports-team: "handle team data"` — matches nothing
  concretely, LLM won't reliably pick it.
- Descriptions in one language when users prompt in another. Match the
  user's language for the phrasings.

### Validation — conflict matrix (do this before shipping)

Build a mental matrix: rows = every planned specialized skill, columns
= 5-10 realistic user phrasings. For each cell, ask "which skill is
the natural match for this phrasing?". Every cell must have exactly ONE
winner. If two skills tie on any phrasing, rewrite descriptions until
they don't.

Concrete process to run yourself before completion:

1. Extract ALL keywords / user phrasings from every specialized skill's
   description in the new category.
2. Build a Python dict `{skill: set_of_keywords}`.
3. For every skill pair `(A, B)`, compute `keywords[A] & keywords[B]`.
4. If any pair overlaps on a domain-carrying keyword (not stopwords
   like "the", "use", "for"), FAIL and rewrite.
5. Also cross-check against sibling categories: `sports-player top
   scorers` should NOT collide with a `nsn-ops` keyword.

You can literally run this as a Python one-liner during Step 7 of the
workflow to catch collisions before propagation.

## Router skill — the 3 levers

There is NO Hermes flag that marks a skill as "the router". The router
pattern relies on 3 content levers in `<category>-overview/SKILL.md`:

1. **Description parapluie** — the frontmatter `description` mentions
   ALL sub-domain keywords the LLM might match, plus the phrase
   `"Load this FIRST when the user asks anything about <domain>"`.
2. **Explicit "Load FIRST" instruction** — plain English, the LLM reads
   and follows it.
3. **Decision table** in the body — `| user intent | skill to load |`
   rows covering every specialized skill.

The specialized skills have **narrow** descriptions matching only their
sub-domain, so they get loaded when (and only when) the router points
to them.

**The quality of the frontmatter description is the #1 reliability
factor.** Treat it as product copy: list the exact user phrasings, use
the language the user speaks, state what's NOT in scope.

## Code conventions (Python stdlib only, no exceptions)

- **Only stdlib**: `urllib`, `json`, `ssl`, `time`, `argparse`, `sys`,
  `datetime`. Zero `pip install` — the `hermes-webui` image has no
  package manager available at runtime.
- **Shared HTTP client** in `<category>-overview/scripts/api_client.py`:
  base class `HttpClient` with `_get`, `_post`, 60s in-process cache,
  `ApiError(status, body)` exception on HTTP >= 400.
- **Per-service classes** on top of `HttpClient` (one class per external
  service or API namespace).
- **Sub-commands via argparse** in each specialized script.
- **Every CLI accepts `--format json|table`**. JSON default (parseable
  by the LLM), table on demand for human display.
- **Exit codes**: 0 = success, 2 = `ApiError`, 3-4 = business errors
  (invalid input, no data, etc.) with a JSON `{error, hint}` on stderr.
- **Read-only by default**. Never expose write operations unless the
  user explicitly demands them AND we agree together it's safe.

Scripts import from the router's shared code:
```python
import sys
sys.path.insert(0, "/home/hermeswebui/.hermes/skills/<category>/<category>-overview/scripts")
from api_client import <Service>, ApiError
from config import <SERVICE>_URL, <SERVICE>_TOKEN
```

## SKILL.md structure (mandatory sections)

```markdown
---
name: <category>-<domain>
description: "<one dense sentence — list concrete user phrasings, keywords, and what's NOT in scope>"
version: 1.0.0
platforms: [macos, linux]
metadata:
  hermes:
    tags: [<domain>, <keywords>, ...]
    category: <category>
prerequisites:
  commands: [python3]
---

# <Category> — <Domain> (<one-line qualifier>)

## When to use
- <user phrasing 1>
- <user phrasing 2>
- ...

## When NOT to use
- <adjacent domain 1> → `/<other-skill>`
- Write operations → **refuse politely**, do via UI
- ...

## CLI

```bash
python3 ~/.hermes/skills/<category>/<category>-<domain>/scripts/<name>.py <cmd> \
  [--flag <val>] [--format json|table]
```

## Examples

**"<realistic user question 1>"**
```bash
<exact command that answers it>
```

**"<realistic user question 2>"** (2-step if resolution needed)
```bash
<step 1 — resolve ID/slug>
<step 2 — actual query>
```

## Gotchas
- <environment-specific fact that defies a reasonable assumption>
- <auth / rate limit / TLS specific to THIS API>
- <field returned in different case than requested (e.g., SportMonks `include=currentSeason` → response `currentseason`)>
- <pagination quirk that a naive loop would miss>
```

Section rules (spec-aligned with agentskills.io):
- **`## When to use`** and **`## When NOT to use`** — 3-5 concrete user
  phrasings each. Adjacent-skill routing goes in NOT clauses.
- **`## CLI`** — every sub-command with all flags shown, defaults inline.
- **`## Examples`** — realistic user questions → the exact command that
  answers them. Two-step examples when resolution is needed.
- **`## Gotchas`** (rename of "Notes / pitfalls") — environment-specific
  facts that defy assumptions. **Never generic advice** ("handle errors
  appropriately") — only concrete corrections to mistakes the LLM will
  make without being told. Each gotcha = one rule + one clause of WHY.

## SKILL.md size discipline

**Hard cap: 500 lines / ~5,000 tokens per SKILL.md** (agentskills.io spec).
Our current specialized skills sit around 80-150 lines — well under. This
cap prevents future drift.

When you'd exceed the cap:
1. Move detailed reference material to `<skill>/references/*.md`
2. **In SKILL.md, tell the LLM EXACTLY WHEN to load each reference file** —
   never write a generic "see references/ for more". Example:
   ```markdown
   For Odds-API HTTP 4xx / 5xx responses see
   `references/oddsapi-errors.md` (loaded on demand).
   ```
3. Keep the SKILL.md body itself focused on the 80% common workflow.

`references/` files DO NOT count toward the 500-line cap because they're
loaded on demand by the LLM only when the trigger you documented fires.

## Content selection rules

### 1. Add what the LLM lacks, omit what it knows

Do NOT explain generic technology (HTTP, JSON, what a PDF is, how OAuth
works). Focus exclusively on the **project-specific quirks** the LLM
can't know without your skill:

```markdown
✗ "Sentry is an error tracking service. It groups errors by fingerprint.
   To get errors, call the API..."

✓ "Sentry issue IDs are numeric strings (both int and str accepted). The
   `events` endpoint paginates — take the newest for a fresh stack trace.
   Auth header is `Bearer <token>`, org slug hardcoded to `nsn`."
```

Rule of thumb: "Would the LLM get this wrong without this line?" If no,
cut it.

### 2. Match specificity to fragility

- **Prescriptive (rigid commands)** for fragile / destructive / consistency-
  critical operations. State the exact command; forbid modifications.
- **Explanatory (why + freedom)** for flexible approaches where multiple
  paths work. Tell the LLM the reason so it makes good context calls.

Most skills mix both. Calibrate each section independently.

### 3. Provide defaults, not menus

When several tools / libs / flags could work, **pick one default** and
mention alternatives briefly:

```markdown
✗ "You can use pypdf, pdfplumber, PyMuPDF, or pdf2image..."

✓ "Use pdfplumber (text extraction). For scanned PDFs → pdf2image + pytesseract."
```

For CLI defaults, state them once in the sub-command signature — do NOT
list every value the user COULD set.

### 4. Procedures over declarations

The skill teaches the LLM **how to approach a class of problems**, not
what to produce for one instance:

```markdown
✗ "Join `orders` to `customers` on `customer_id`, filter `region='EMEA'`,
   sum `amount`."

✓ "1. Read the schema (`information_schema.tables`) to find relevant tables.
    2. Join via `_id` FK convention. 3. Apply user filters as WHERE clauses.
    4. Aggregate numeric columns; return a markdown table."
```

Specific details (output templates, hard constraints) are still valuable
— but the **approach** must generalize.

### 5. Progressive disclosure ordering

Put the **80% common workflow at the top** of the SKILL.md body. Edge
cases, rare flags, advanced patterns → bottom. The LLM reads top-to-bottom
and skims deeper content only when needed.

Order:
1. `When to use` / `When NOT to use` (routing signal)
2. CLI (surface area — highest use)
3. Examples (most common questions FIRST, exotic ones LAST)
4. Building the query / agent workflow (if any)
5. Gotchas (only surface as the LLM encounters them)

## Instruction patterns (use only when relevant)

Not every skill needs every pattern. Pick what fits.

### Checklists for multi-step workflows

When step order matters and skipping is costly, give the LLM an explicit
markdown checklist to track progress:

```markdown
## Form processing workflow
- [ ] 1. Analyze the form (`scripts/analyze_form.py`)
- [ ] 2. Create field mapping (edit `fields.json`)
- [ ] 3. Validate mapping (`scripts/validate_fields.py`)
- [ ] 4. Fill the form (`scripts/fill_form.py`)
- [ ] 5. Verify output (`scripts/verify_output.py`)
```

### Templates for output format

For "produce output in shape X" tasks, provide a **concrete example** —
the LLM pattern-matches structures far better than it follows prose:

````markdown
## Report structure

Use this template (adapt sections for the specific analysis):

```markdown
# [Analysis Title]

## Executive summary
[One paragraph]

## Key findings
- Finding 1 with data
- Finding 2 with data
```
````

Short templates inline; long ones in `assets/` referenced on demand.

### Validation loops / Plan-validate-execute

For batch or destructive operations (once we add write skills), have the
LLM:
1. Do the work OR create an intermediate plan (JSON)
2. Run a validator (script that checks the plan against source of truth)
3. Fix any issues → re-validate
4. Only then execute

The validator's error message must be informative enough for the LLM to
self-correct (e.g., "Field 'signature_date' not found — available fields:
customer_name, order_total, signature_date_signed").

Not applicable to our read-only skills today — but note this exists when
scaffolding future write-capable ones.

## Interactive workflow you MUST follow

### Step 1 — Understand the ask

Ask the user in ONE round:
1. What's the API / domain?
2. Do you have the API doc URL, or should I search?
3. Auth method + a working credential (or say "I'll set env vars later")?
4. Which category should this live in? (list existing ones; ask if new)
5. Main use cases in the user's own words (3-5 phrasings)?

Skip questions the user already answered.

### Step 2 — Analyze the source

**If API doc URL**: use `WebFetch` to pull the endpoint list + auth
scheme. Group endpoints by sub-domain (data type / resource kind /
user intent). Note pagination, rate limits, quirks.

**If brief only**: search for the official API doc (WebSearch), then
follow the "API doc" path. If truly no public API, ask the user for
schema samples or a working SDK example.

### Step 3 — Propose the architecture

Present a compact plan:
- New category `<name>` or add to existing `<name>`?
- Router description (draft)
- N specialized skills (list them with 1-line descriptions each)
- List of shared API client methods needed
- Auth handling (env var name + config var declaration)
- Anything unusual: TLS quirks, pagination, filters that must be
  client-side, etc.

**Wait for user go/no-go.** Don't code the whole thing then ask.

### Step 4 — Scaffold the category

Only for a NEW category:
1. `mkdir -p config/skills/<category>/{<category>-overview/scripts, <skill-A>/scripts, <skill-B>/scripts, ...}`
2. Write `DESCRIPTION.md` (short, Hermes-facing)
3. Write `<category>-overview/scripts/config.py` (URLs + tokens with env
   var fallback, matching `NSN_OPS_*` or similar prefix convention)
4. Write `<category>-overview/scripts/api_client.py` (`HttpClient` base
   + per-service class(es))
5. Write `<category>-overview/SKILL.md` (router — see next section)

### Step 5 — Write the router SKILL.md

Follow the template — the 3 levers matter:
```yaml
---
name: <category>-overview
description: "Router + shared HTTP clients for all <domain> questions (<comma-separated sub-domain keywords>). Load this FIRST when the user asks anything <matching phrasing>. Read-only by default."
metadata:
  hermes:
    tags: [<domain>, router, <keywords>]
    category: <category>
    related_skills: [<list all specialized skills>]
    config:
      - key: <category>.<service>_token
        description: <what it is>
        prompt: <label shown in WebUI>
---
```

Body must include: decision table, shared client import snippet,
endpoint summary table, output convention, error handling contract,
read-only policy statement.

### Step 6 — Write each specialized skill

For each `<category>-<domain>/`:
1. Draft the frontmatter (description = product copy with user phrasings)
2. Fill mandatory sections (When to / NOT to / CLI / Examples / Notes)
3. Write `scripts/<name>.py` — import from router, argparse sub-commands,
   `--format json|table`, exit codes.
4. Keep the script FLAT (parsing + rendering only). All HTTP goes through
   the shared client. If you need a new method, add it to `api_client.py`.

### Step 7 — Compile + propagate + smoke test

```bash
# Compile
python3 -m py_compile config/skills/<category>/<category>-overview/scripts/*.py \
                       config/skills/<category>/<skill>/scripts/*.py

# Propagate to live volume (hot — no container restart needed)
docker exec hermes_webui mkdir -p /home/hermeswebui/.hermes/skills/<category>/<skill>/scripts
docker cp config/skills/<category>/DESCRIPTION.md hermes_webui:/home/hermeswebui/.hermes/skills/<category>/DESCRIPTION.md
for skill in <category>-overview <skill-A> <skill-B> ...; do
  docker cp config/skills/<category>/$skill/SKILL.md hermes_webui:/home/hermeswebui/.hermes/skills/<category>/$skill/SKILL.md
  docker cp config/skills/<category>/$skill/scripts/. hermes_webui:/home/hermeswebui/.hermes/skills/<category>/$skill/scripts/
done
docker exec hermes_webui chown -R 1000:1000 /home/hermeswebui/.hermes/skills/<category>

# Smoke test each specialized skill with a real call
docker exec hermes_webui python3 /home/hermeswebui/.hermes/skills/<category>/<skill>/scripts/<name>.py <cmd> --format table
```

Do the smoke tests **before** telling the user you're done. If any fail
(HTTP 401, wrong endpoint shape, empty response), fix and retest.

### Step 8 — Report back

Concise recap:
- Table listing every new/modified file with the one-line change
- List of smoke tests + their real output snippets (proves it works)
- Any TODO left for the user (env var to set, token to rotate, etc.)
- Reminder: **Hermes needs a new conversation** to see new SKILL.md
  files (manifest snapshot). Scripts are picked up live.

## Propagation & lifecycle (critical to remember)

- **Skills seed lifecycle**: `docker-compose.yml` mounts
  `./config/skills:/seed/skills:ro`; `scripts/hermes-init.sh` copies to
  `/hermes-home/skills/` at boot. Skills are ALWAYS re-written from
  seed (no idempotence) — the repo is the source of truth.
- **Hot-copy for iteration**: use `docker cp` for individual files
  during development. Faster than `docker compose restart`.
- **SKILL.md are snapshotted at conversation start**. A conversation
  that started before your changes will keep seeing the old SKILL.md.
- **Python scripts run live** — a code fix works immediately, even
  mid-conversation.
- **Token rotation via config vars**: prefer WebUI Settings → Skills
  config var (`<category>.<service>_token`) over hardcoded values. The
  `config.py` should default to `os.environ.get("<PREFIX>_TOKEN", "<fallback>")`.
- **Alternative: `required_environment_variables` frontmatter field**
  (Hermes-native). Declares API keys/tokens that Hermes stores in
  `~/.hermes/.env` (never shown to the model) and auto-injects into the
  `terminal` / `execute_code` sandboxes when the skill loads. Use this
  instead of `metadata.hermes.config` when the value is a **secret**
  (tokens, API keys) — `config.hermes` is meant for non-sensitive settings
  like paths and preferences. Example:
  ```yaml
  required_environment_variables:
    - name: NSN_OPS_GITLAB_TOKEN
      prompt: "GitLab read-only PAT"
      help: "https://gitlab.nsn/-/user_settings/personal_access_tokens"
  ```
  Our current skills use hardcoded fallbacks in `config.py` — this is
  convenient for dev but the `required_environment_variables` route is
  strictly cleaner for production secrets.

## Non-negotiable checks before saying "done"

- [ ] `python3 -m py_compile` passes on every script
- [ ] At least one smoke test per specialized skill returned real data
- [ ] Router `description` mentions every sub-domain keyword
- [ ] Router decision table covers every specialized skill
- [ ] Each specialized skill's description lists concrete user phrasings
- [ ] `When NOT to use` lists adjacent skills to route to
- [ ] **Description conflict matrix run** — no domain-carrying keyword
      appears in more than one specialized skill's description within
      the category (router excluded, it's meant to be broad)
- [ ] For every realistic user phrasing you can invent, exactly ONE
      specialized skill is the natural match (test 5-10 phrasings)
- [ ] No write operations exposed (unless user explicitly agreed)
- [ ] Auth tokens read from env var with fallback (`config.py` pattern)
- [ ] Propagated to `/home/hermeswebui/.hermes/skills/<category>/`
- [ ] Chowned to `1000:1000` (webui user)
- [ ] User told to start a NEW Hermes conversation to see the manifest
- [ ] Each SKILL.md is under 500 lines / ~5,000 tokens (agentskills.io
      cap). If it grew past, refactor into `references/` with explicit
      "load when X" pointers in the body
- [ ] Body ordering follows progressive disclosure: 80% common workflow
      first, edge cases last
- [ ] Every section content passes the "would the LLM get this wrong
      without this line?" test — no generic tech explanations
- [ ] For each option / library / flag: a clear default is picked, not a
      menu of equal choices
- [ ] "Gotchas" section (renamed from "Notes / pitfalls") contains only
      environment-specific facts, never generic advice
- [ ] **All produced content is in English** — SKILL.md, Python code,
      comments, docstrings, argparse help, table headers, error messages,
      README, DESCRIPTION. User phrasings in another language allowed
      only as quoted trigger examples in `When to use` bullets.

## Common pitfalls (learned the hard way)

- **Do NOT paginate by post-filter count.** If you filter client-side
  (e.g., `merged_after` emulation), the pagination loop's stop condition
  must be `len(raw_page) < 100` (raw), not `len(out) < limit` (post-filter),
  otherwise a full page of filtered-out rows falsely signals EOF.
- **Dedupe composite aggregations by canonical slug** when two `(k1, k2)`
  pairs represent the same user-facing entity (see `Elasticsearch.sites()`
  merging `<slug>` and `<slug>-static` for the same site).
- **`--limit` defaults matter**: a low default silently truncates
  windowed queries. Set defaults to match the underlying client method's
  own default (usually 1000 for globally-scoped queries).
- **API field naming**: some providers respond with different case than
  the include (e.g., SportMonks `include=currentSeason` → response
  `currentseason`). Always fetch a raw sample before parsing.
- **Description shape matters**: too generic on a specialized skill
  makes it compete with the router; too vague on the router means it
  never gets auto-loaded. Compare with `sports/` router description as
  a reference.
- **Self-signed certs / TLS opt-out**: scope `ssl.CERT_NONE` to the
  specific client (not module-wide). See `GitLab` client in `nsn-ops`.
- **Never introduce new dependencies**. If you're tempted, extract the
  minimum you need from the stdlib (`urllib` covers 90% of what
  `requests` does).

## Quick reference files to read first

Before you start coding, load into context:
1. `config/skills/sports/README.md` — 11-section dev doc explaining the
   architecture and how Hermes discovers skills
2. `config/skills/sports/sports-overview/SKILL.md` — canonical router
   example with the decision-table pattern
3. `config/skills/sports/sports-overview/scripts/api_client.py` — shared
   HTTP client with multiple service classes
4. `config/skills/nsn-ops/README.md` — same architecture applied to
   internal ops tools, shorter and more focused
5. `config/skills/nsn-ops/nsn-ops-overview/scripts/api_client.py` — the
   4-service pattern (GitLab / Sentry / Elasticsearch / Metabase) with
   TLS quirks, auth headers, per-page pagination

Reading these first ensures the new skill fits seamlessly into the
existing project without you having to re-derive conventions from
scratch.
