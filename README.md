# autonomous-dev-loop

A [Claude Code skill](https://docs.claude.com/en/docs/claude-code) that runs a **fully autonomous development loop**: hand it a project idea, plan, or PRD — or nothing at all ("build whatever you want") — and it owns the project end to end, across many work cycles, with the human out of the loop.

Distilled from three independent long-running experiments in AI-owned repositories (one Codex, two Claude). Every rule in the skill traces to a concrete success or failure observed in those runs — nothing here is theoretical.

## The core idea

> Cheap models can build good software only when a disciplined lead refuses to trust unverified success reports and gates every change on tests plus adversarial review. Quality comes from structure, not effort.

The skill turns the session's model into that lead. Three roles, strictly split:

| Role | Model | Does |
|---|---|---|
| **Lead** | the session's model | Picks the work, makes every architecture and security decision, writes decision-locked specs, independently verifies everything, runs reviews, owns git and docs. Writes zero product code. |
| **Implementer** | cheapest feasible (Haiku-class) sub-agent | Writes 100% of product code and tests, strictly from the lead's specs |
| **Reviewer** | stronger than the implementer | Adversarial security review where every finding requires a **live reproduction** — the exact input, request, or byte sequence that breaks it |

## Each cycle

1. **Verify prior state** — environment hygiene, clean tree, green suite; recover from any dead sub-agent first
2. **Decide if the cycle should happen** — continue / pivot / pause / wrap up, against explicit stop criteria; a stuck item forces the decision after 3 cycles
3. **Pick one shippable slice** and write a spec with every design decision already made
4. **Delegate** to the implementer under a strict contract (file ownership, do-NOT list, required tests, honest-failure escape hatch)
5. **Independently verify** — re-run the suite, read the diff, smoke-test; a sub-agent's "all green" is treated as false until reproduced
6. **Adversarial review** — every code-touching slice; whole-codebase audit every 5 cycles to catch defects diff review structurally cannot see
7. **Land** — one slice, one commit; update the roadmap and devlog; heartbeat to the human

State lives in files (`CHARTER.md`, `ROADMAP.md`, `docs/DEVLOG.md`, `REQUESTS.md`), so any cold session can resume the loop with zero conversational history.

## Does it work?

In assertion-based evals (two projects, built with and without the skill on the same model), with-skill runs passed **18/18** assertions vs **6/18** for baselines. Both configurations produced working, tested software — the difference was everything else: with-skill reviews live-reproduced **7 real defects** (a crash hiding intact data, a forged-timestamp injection, an env-var edge case writing files to the wrong directory…); baselines caught zero, wrote no charter or stop criteria, and one never even ran `git init`. The eval set ships in [`evals/`](evals/evals.json).

## Install

Symlink, so the installed skill tracks this repo (recommended):

```bash
git clone https://github.com/vectorlane80/autonomous-dev-loop.git
ln -s "$(pwd)/autonomous-dev-loop" ~/.claude/skills/autonomous-dev-loop
```

Or just copy the directory to `~/.claude/skills/autonomous-dev-loop`.

Works best alongside the [senior-engineering skills](https://github.com/vectorlane80/senior-engineering-skills) and an adversarial reviewer skill (Oscar / `ai-grouch-claude`) — the loop maps each one to the specific step where it proved out in the original runs, and degrades gracefully when they're absent.

## Use

In any Claude Code session:

- **Directed mode:** *"Here's a PRD at ./PRD.md — run the autonomous dev loop on it."* The lead distills the PRD into a charter (surfacing every ambiguity and how it resolved it), you approve once, and it runs unattended from there. Add *"consider the charter pre-approved"* to skip even that.
- **Freestyle mode:** *"Build whatever you want. Run your loop."* No sign-off — the surprise is the point. Freestyle gets stricter default stop criteria, because the loop is excellent at continuing and must be forced to ask whether continuing is worthwhile.

During a run, the only human channel is a `REQUESTS.md` checkbox file in the project repo; the loop never blocks on approval for normal work.

## Layout

| File | Contents |
|---|---|
| [`SKILL.md`](SKILL.md) | The lead's operating manual: roles, intake, setup, the per-cycle loop, wrap-up, companion-skill map |
| [`references/templates.md`](references/templates.md) | Charter (with stop-criteria defaults per mode), roadmap/devlog/requests formats, the implementer constraints file |
| [`references/delegation.md`](references/delegation.md) | The delegation prompt contract, implementer lifecycle and cost ceiling, sub-agent failure recovery |
| [`references/review.md`](references/review.md) | Adversarial review protocol: the evidence rule, audit cadence, fallback reviewer persona |
| [`evals/evals.json`](evals/evals.json) | Assertion-based eval set used to validate the skill |
