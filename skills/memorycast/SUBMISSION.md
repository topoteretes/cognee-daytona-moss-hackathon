# Team Submission

## Team

- Team name: MemoryCast
- Participants: Subhiksha Mukuntharaj
- Skill name: memorycast

## Skill

- Skill path: `skills/memorycast/SKILL.md`
- Final `SKILL.md` summary: v2 -- upgraded from 5 manual heuristics to 3 learned rules derived from 20 real forecasting runs stored in Cognee/Moss. Rules cite specific evidence (run counts, WAPE values) and tell the agent which model to try first for each series profile type.

## Runs

### Baseline Run

- Task or PR: Collections Volume (Finance) -- first run with v1 skill, no memory
- Score: 0.826
- Main failure mode: No learned rules -- ran full tournament on every series, no memory-guided shortcuts, avg WAPE 17.4%
- `SkillRunEntry.run_id`: fb19ae13
- Log path or link: `memory/skill_runs.json` (entry 1) at https://github.com/Subhi7/memorycast

### Improved Run

- Task or PR: SaaS Annual Recurring Revenue -- run 20, agent used learned rule to pick AutoARIMA directly
- Score: 0.987
- Improvement over baseline: +16.1pp (WAPE 1.3% vs 17.4%)
- `SkillRunEntry.run_id`: f4f67e5e
- Log path or link: `memory/skill_runs.json` (entry 20) at https://github.com/Subhi7/memorycast

## Feedback Loop

What feedback did Cognee record?

```text
error_type: high WAPE on high-vol/strong-seasonal series
error_message: AutoARIMA and SeasonalNaive both underperform on series with seasonality > 0.4
feedback: AutoETS won on 'Retail Demand (Seasonal)' (high-volatility, strong-seasonality). Profile: CV=0.19, seasonality=0.57. Tournament: AutoETS=2.9%, AutoARIMA=4.1%, SeasonalNaive=4.5%. 5 consecutive wins for AutoETS on this profile type.
success_score: 0.971
```

What changed in the skill because of that feedback?

```text
Before:
  "Prefer statistical models for strong-seasonality series"
  (no model specified, no evidence)

After:
  "High volatility + strong seasonality (volatility > 0.15 AND seasonality >= 0.4)
   -> Preferred model: AutoETS (avg success score: 0.97 across 7 runs)
   -> Evidence: Retail Demand (Seasonal) -- AutoETS won, WAPE 2.9%, score 0.971"
```

## PR Rescue Result

- Bug or regression found: Vague heuristics in SKILL.md v1 caused the agent to run all 4 models every time, wasting compute and sometimes picking a suboptimal model
- Fix proposed or applied: skill_updater.py reads all SkillRunEntry records and rewrites SKILL.md with specific learned rules + evidence citations. Agent now skips low-confidence models when memory consensus is 3/3.
- Tests run or specified: 20 cross-validation runs across 8 distinct series types. 3-window rolling CV per run.
- Remaining risk: Low-vol + strong-seasonality profile has only 3 runs -- rule confidence is lower for that bucket.

## Agent Team

```text
Profiler: profiles the series (volatility, seasonality, trend, ACF lag-1, regime shift, outlier rate)
Scout: queries Cognee/Moss for top-3 similar past SkillRunEntry records
Strategist: Claude agent (claude-haiku-4-5-20251001, 7 tools, up to 10 turns) decides which models to run
Runner: runs cross-validation tournament (SeasonalNaive / AutoARIMA / AutoETS / GradientBoosting)
Recorder: saves SkillRunEntry to Cognee (Moss cloud vectors) with success_score + feedback
Updater: skill_updater.py reads all entries and rewrites SKILL.md with learned rules
```

## Reproduction

```bash
git clone https://github.com/Subhi7/memorycast
cd memorycast
uv sync
cp .env.example .env  # fill in ANTHROPIC_API_KEY, OPENAI_API_KEY, MOSS_PROJECT_ID, MOSS_PROJECT_KEY
uv run python seed_memory.py          # seeds 20 lessons into Cognee/Moss
uv run python agents/skill_updater.py # generates SKILL.md v2, prints diff
uv run streamlit run app.py           # launches UI on port 8501
```

Environment assumptions:

```text
ANTHROPIC_API_KEY  -- Claude agent (claude-haiku-4-5-20251001)
OPENAI_API_KEY     -- Cognee embeddings (text-embedding-3-small)
MOSS_PROJECT_ID    -- Moss cloud vector store project ID
MOSS_PROJECT_KEY   -- Moss cloud vector store project key
```
