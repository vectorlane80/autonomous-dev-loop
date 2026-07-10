# autonomous-dev-loop

A [Claude Code skill](https://docs.claude.com/en/docs/claude-code) that runs a fully autonomous development loop: hand it a project idea, plan, or PRD — or nothing at all ("build whatever you want") — and it owns the project end to end across many work cycles: charter, roadmap, decision-locked specs, delegation to cheap implementer sub-agents, independent verification, adversarial security review with a live-reproduction evidence rule, and an honest decision every cycle about whether the project deserves another one.

Distilled from three independent long-running autonomous development experiments (Codex and Claude). Every rule in the skill traces to a concrete success or failure observed in those runs.

Works best alongside the [senior-engineering skills](https://github.com/vectorlane80/senior-engineering-skills) and an adversarial reviewer skill (Oscar / `ai-grouch-claude`) — the loop maps each one to the specific step where it proved out in the original runs, and degrades gracefully when they're absent.

## Install

Symlink (recommended — the installed skill tracks this repo):

```bash
git clone <this-repo> autonomous-dev-loop
ln -s "$(pwd)/autonomous-dev-loop" ~/.claude/skills/autonomous-dev-loop
```

Or copy the directory to `~/.claude/skills/autonomous-dev-loop`.

## Use

In any Claude Code session:

- **Directed:** "Here's a PRD at ./PRD.md — run the autonomous dev loop on it." You approve the distilled charter once, then it runs unattended. Add "consider the charter pre-approved" to skip even that.
- **Freestyle:** "Build whatever you want. Run your loop." No sign-off; the surprise is the point.

Human interaction during a run happens through a `REQUESTS.md` checkbox file in the project repo.

## Layout

- `SKILL.md` — the lead agent's operating manual (roles, intake, setup, the per-cycle loop, wrap-up)
- `references/templates.md` — charter/roadmap/devlog/requests templates and the implementer constraints file
- `references/delegation.md` — delegation contract, implementer lifecycle, failure recovery
- `references/review.md` — adversarial review protocol and the evidence rule
- `evals/evals.json` — the assertion-based eval set used to validate the skill (with-skill runs passed 18/18 assertions vs 6/18 for no-skill baselines)
