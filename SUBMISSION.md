# Team Submission

## Team

- Team name: ds
- Participants: Spandan Das
- Skill name: pr-rescue

## Skill

- Skill path: `skills/pr-rescue/SKILL.md`
- Final `SKILL.md` summary: A focused PR rescue skill built around one high-confidence finding with a minimal fix and a named test. Key additions over the starter: explicit empty-input safety checks for every storage write path, a strict 6-field output format that enforces evidence and test coverage, a growing "Rules Added From Prior Run Feedback" section seeded from observed failure patterns, and a self-improvement loop that fires when `success_score < 0.7` and adds exactly one concrete rule per run.

## Runs

### Baseline Run

- Task or PR: Security and correctness review of `https://github.com/topoteretes/cognee` (practice round — challenge PR pending)
- Score: weak / partial
- Main failure mode: The starter skill's process was under-specified. It directed the agent to "recall prior project memory" without grounding that in a specific tool call or symbol. The output format was informal — no enforcement of evidence or test coverage per finding. The agent produced findings but they lacked the depth and structure needed to confirm real impact or guide a fix.
- `SkillRunEntry.run_id`: baseline-pr-rescue
- Log path: `/tmp/daytona-demo-run6.log`

### Improved Run

- Task or PR: Same codebase — `https://github.com/topoteretes/cognee`
- Score: **1.0**
- Improvement over baseline: Skill loaded with specific memory-search grounding, strict output structure, and explicit storage-write checks. Agent produced 10 prioritized findings with exact file references, evidence, minimal fixes, and named tests for every issue.
- `SkillRunEntry.run_id`: `daytona:review:pr-rescue`
- Log path: `/tmp/daytona-demo-run-v2.log`

## Feedback Loop

What feedback did Cognee record?

```text
error_type:    weak_skill_output
error_message: Starter skill process was ambiguous — memory recall step had no
               grounding tool call. Output format was informal; evidence and test
               fields were not required per finding. No rules accumulated from
               prior runs to guide edge-case checks.
feedback:      -0.6
success_score: 0.2
```

What changed in the skill because of that feedback?

```text
Before (starter skill):

  Process step 2: "Recall prior project memory before making claims."
  → No tool specified, no symbol anchoring. Agent could skip this or do it
    inconsistently.

  Process step 4: "Check permission, tenant isolation, auth, destructive
    operations, and empty input behavior."
  → Empty input behavior lumped in with everything else; no storage-write
    specificity.

  Output format: informal bullet list — Severity, Location, Problem, Fix, Tests.
  → Evidence not required. Test not required to name a file and assertion.
    Easy for the agent to pad findings with no proof.

  Self-improvement: "If the run failed, update this skill with one concrete rule."
  → No score threshold. No constraint on rule format. Agent could add vague rules
    or skip the step.

  No accumulated rules section.

After (improved skill):

  Process step 2: "Call memory_search with the primary changed symbol before
    making any claim about its behavior."
  → Specific tool, specific trigger. Agent cannot skip it.

  Process step 5 (new): "For every storage write (graph, vector, relational,
    cache, file): verify the write is safe when the input collection is empty."
  → Isolated from the general checklist so the agent checks it explicitly for
    every write path, not just when it remembers to.

  Checklist additions: empty/None/missing input, destructive-op guard,
    token signature verification before trusting decoded claims.

  Output format: strict 6-field block — Severity / Location / Problem /
    Evidence / Fix / Test — all required, no padding allowed.

  Rules Added From Prior Run Feedback (new section): three concrete rules
    derived from observed failure patterns:
    1. Empty edge list safety on graph writes.
    2. Signature verification before trusting decoded token payload claims.
    3. Explicit user / tenant / dataset / role boundary naming on permission PRs.

  Self-improvement: fires when success_score < 0.7; adds exactly one rule;
    rule must name the specific condition and what the next agent should check.
```

## PR Rescue Result

From the improved run (`daytona-demo-run-v2.log`):

- **Bugs / regressions found**: 10 findings — 3 critical, 4 high, 3 medium
- **Top critical finding**: Hardcoded JWT secret defaults (`"super_secret"`) in `cognee/modules/users/get_user_manager.py:23-26` — any attacker can forge password-reset and email-verification tokens with zero database access.
- **Fix proposed**: Remove the defaults entirely; raise `ValueError` at startup if the env vars are absent or match the known-bad string.
- **Tests specified**: Every finding includes an exact test. Example: "Craft a signed JWT using `super_secret` against a fresh deploy; assert it is accepted pre-fix and rejected post-fix."
- **Remaining risk**: `ALLOW_HTTP_REQUESTS` is documented but never read (`save_data_item_to_storage.py:18-24`); shares a root cause with the SSRF finding and both are fixed by the same one-line settings addition.

Full findings from improved run:

| # | Severity | Location | Problem |
|---|----------|----------|---------|
| 1 | Critical | `modules/users/get_user_manager.py:23-26` | Hardcoded JWT secret `"super_secret"` — forgeable password-reset and verification tokens |
| 2 | Critical | `modules/users/api_key/hash_api_key.py:4` | API keys stored plaintext by default; when hashing is on, unsalted SHA-256 is used |
| 3 | Critical | `modules/notebooks/operations/run_in_local_sandbox.py:42` | Unrestricted `exec()` — full OS access for any notebook user |
| 4 | High | `tasks/ingestion/save_data_item_to_storage.py:61-63` | SSRF — HTTP URLs fetched without private-IP denylist; `ALLOW_HTTP_REQUESTS` never read |
| 5 | High | `modules/retrieval/cypher_search_retriever.py:55` | Cypher query injection — user input passed verbatim to graph engine; destructive writes possible |
| 6 | High | `infrastructure/files/storage/LocalFileStorage.py:88,131` | Path traversal — `os.path.join` with no boundary check |
| 7 | High | `api/v1/ui/ui.py:199-213` | Frontend ZIP extracted without SHA-256 integrity check — MITM / CDN compromise vector |
| 8 | Medium | `modules/users/get_user_manager.py:39,44` | Password-reset tokens written verbatim to logs |
| 9 | Medium | `api/v1/ui/ui.py:624`, `api/v1/ui/npm_utils.py:40` | Shell injection via unquoted `$XDG_CONFIG_HOME` in NVM path interpolation |
| 10 | Medium | `tasks/ingestion/save_data_item_to_storage.py:18-24` | `ALLOW_HTTP_REQUESTS=false` has no effect — env var declared but never consumed |

## Agent Team

Four-agent pipeline sharing persistent memory through Cognee's `shared_codebase_memory` dataset via Daytona volume snapshots and a live Moss vector index:

```text
Scout (arch):    Maps project structure, entry points, main modules, and
                 architecture. Stores findings in shared graph.
                 Result: 87 nodes, 173+ edges written to shared_codebase_memory.

API agent:       Recalls arch findings via cognee-search, then maps the full
                 API layer — routes, middleware, auth, request handling, DTOs.
                 Snapshot carried forward: 525 KB.

Tests agent:     Recalls arch + API findings, then maps the test suite —
                 229+ test files, 42 CI workflows, 5 concrete coverage gaps.
                 Snapshot carried forward: 749 KB.

Review agent:    Recalls all prior agent findings via cognee-search. Loads
                 pr-rescue skill from Cognee memory. Applies skill criteria to
                 produce 10 prioritized findings. Scores the run and writes
                 SkillRunEntry back to Cognee (entry_id: daytona:review:pr-rescue).
                 Final shared graph: 1018 KB snapshot, 221 nodes, 402 edges.
```

Each agent's session memory is private (`per-directory`, scoped by `COGNEE_SESSION_PREFIX`). Shared knowledge crosses agent boundaries only through explicit `cognee-remember` / `cognee-search` calls against `shared_codebase_memory` — agents cannot accidentally read each other's working state.

## Reproduction

```bash
# Clone repos
git clone https://github.com/topoteretes/cognee
cd cognee
git checkout graphskills-on-agentic

# Set env vars
export DAYTONA_API_KEY=...
export ANTHROPIC_API_KEY=...
export LLM_API_KEY=...
export MOSS_PROJECT_ID=...
export MOSS_PROJECT_KEY=...

# Run with improved skill
python distributed/deploy/daytona_onboarding_demo.py \
  --repo https://github.com/topoteretes/cognee \
  --skills-dir ../cognee-daytona-moss-hackathon/skills \
  --review-skill pr-rescue \
  --keep-volume
```

Environment assumptions:

```text
DAYTONA_API_KEY    — Daytona account with >= 8 GB memory quota available
ANTHROPIC_API_KEY  — For Claude Code CLI
LLM_API_KEY        — OpenAI key for Cognee graph extraction (gpt-4o-mini default)
MOSS_PROJECT_ID    — Moss project for shared vector index
MOSS_PROJECT_KEY   — Moss project key
                     Note: each full pipeline run consumes ~250 Moss control ops.
                     Ensure sufficient quota before running.
```
