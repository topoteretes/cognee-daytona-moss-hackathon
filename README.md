# Cognee Daytona Moss Hackathon

Hackathon on 25.04.2026 in SF.

## PR Rescue Arena

Build a self-improving agent skill that rescues broken pull requests.

The point is not only to solve one PR. The point is to show this loop working:

```text
starter skill
-> agent run on PR #1
-> score and feedback recorded in Cognee
-> skill improves
-> agent run on PR #2
-> better result
```

Your final artifact is a skill that can keep getting better from run feedback.

## What You Build

Each team creates or improves a PR rescue skill pack:

```text
skills/<your-skill>/SKILL.md
optional helper files
before and after run logs
skill improvement diff
short submission writeup
```

A starter skill is included at:

```text
skills/pr-rescue/SKILL.md
```

Use it directly or fork it into your own skill folder.

## Competition Flow

1. Clone this repo and the Cognee demo branch.
2. Run the baseline agent on the public practice PR.
3. Record the run result with `SkillRunEntry`.
4. Improve `SKILL.md` from that feedback.
5. Run the improved skill on the final PR.
6. Submit your skill, run logs, before/after scores, and skill diff.

## Quickstart

Set the required environment variables:

```bash
export DAYTONA_API_KEY=...
export ANTHROPIC_API_KEY=...
export LLM_API_KEY=...
export MOSS_PROJECT_ID=...
export MOSS_PROJECT_KEY=...
```

Run the Daytona demo from the Cognee repo:

```bash
git clone https://github.com/topoteretes/cognee
cd cognee
git checkout graphskills-on-agentic

python distributed/deploy/daytona_onboarding_demo.py \
  --repo <challenge-pr-repo-url> \
  --skills-dir ../cognee-daytona-moss-hackathon/skills \
  --review-skill pr-rescue
```

The demo creates Daytona sandboxes, loads the skill into Cognee memory, runs agents, records a `SkillRunEntry`, snapshots shared memory, and cleans up the sandboxes.

## Awards

### Best PR Rescue Skill

Best final skill for finding and fixing real PR failures.

Judges look for:

- correct bug or regression identification
- concrete file references
- small practical fix
- useful test plan
- reusable skill design

### Best Self-Improvement Loop

Best evidence that the skill improved from feedback.

Judges look for:

- baseline score before improvement
- `SkillRunEntry` feedback stored in Cognee
- meaningful `SKILL.md` diff
- improved second run
- clear explanation of what changed and why

### Best Agent Team

Best multi-agent workflow for PR rescue.

Example roles:

- scout agent maps the PR and affected code
- fixer agent proposes the patch
- critic agent scores the result
- editor agent improves `SKILL.md`
- verifier agent reruns tests or checks the fix

## Rules

- The final hidden PR must be solved by the agent using the improved skill.
- You may edit the skill between practice runs.
- You may not manually edit the skill during the final hidden round.
- Do not hard-code answers to the practice task.
- Keep the skill readable and reusable.
- Do not bypass repository permissions or ignore safety constraints.

## Submission

Copy `templates/SUBMISSION.md` into your team folder or PR description and fill it out.

Required evidence:

- skill folder
- before score
- after score
- feedback records or logs
- skill diff
- Daytona run output
- short explanation of the loop

Tagline:

```text
Do not just fix the PR. Teach the agent to fix the next one.
```
