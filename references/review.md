# Adversarial review protocol

Adversarial review was the single highest-value practice across all experiment runs: run with the discipline below, it caught a real, ship-blocking defect **every cycle** — silent data loss on mid-edit deletion, an export leaking secrets at mode 0644, a spoofable `X-Forwarded-For` rate limiter (demonstrated with 40 bypassing requests), an unbounded public-endpoint DoS, a fail-open publish default, a parser surfacing deliberately-hidden passwords. None of these would have been caught by "looks good to me."

## Who reviews

Use the `ai-grouch-claude` (Oscar) skill if installed, run on a model **stronger than the implementer** (the lead's model, or a Sonnet-class sub-agent). Code reviewed by a model at or below the strength of the model that wrote it misses too much.

If the skill is not installed, run the review yourself under this persona: *a severe but fair security reviewer who assumes the code is broken until shown otherwise, hunts for concrete failure mechanisms — not style — and changes their mind only on evidence.* Focus areas: input validation, file I/O failure paths, data integrity under interruption, injection of every kind, trust boundaries (who can spoof what), secrets in output or history, error handling that fails open, resource exhaustion, TOCTOU windows, and whether the tests would actually fail if the behavior broke.

Self-reviewing carries a real blind spot: you are auditing code built from your own spec, and motivated reasoning will want it to pass. The evidence rule is your guard, and it cuts both ways — findings need a live reproduction, and a **clean verdict needs receipts too**: list the specific attacks you actually ran (the crafted inputs, the concurrent writers, the interrupted process) rather than declaring the code sound. A clean verdict with no attempted probes listed is not a review; it's an assumption. This applies with extra force to properties your own spec introduced — probe hardest exactly where you made the design calls.

## The evidence rule (absolute)

A finding counts only with a **live reproduction**: the exact request, byte sequence, crafted file, or command that demonstrates the failure — produced by actually running it, not by reasoning from memory. "Consider escaping this" is not a finding; "this exact curl bypasses the limiter; here is the assertion that proves the fix" is.

And the force-multiplier from past runs: **the lead reproduces the exploit personally before writing the fix spec.** This turns vague concerns into decision-locked fix instructions with a regression test attached, and it filters false positives before they cost an implementer cycle.

## Cadence

- **Every code-touching slice** gets a diff-scoped review before it lands. Prioritize depth when the change touches integrity, file I/O, validation, auth, or failure handling.
- **Every 5 cycles and before any release**, run a whole-codebase audit instead. Diff-scoped reviews systematically miss **absence-class defects** — things wrong everywhere equally, so no individual diff looks wrong. (In one run, five consecutive per-cycle reviews all missed that no file I/O anywhere specified an encoding; only a deliberate full audit found it.) The audit asks: what property should hold everywhere, and does it?
- Also verify the **docs**: claims in the README must match actual behavior. Overclaiming documentation is a review finding.

## Handling findings

1. Lead reproduces each finding live. Not reproducible → record as rejected with the attempt, move on.
2. Reproduced → write a decision-locked fix spec including the reproduction as a required regression test; delegate normally.
3. Independently verify the fix: re-run the reproduction (must now fail to exploit) and the full suite.
4. Log every finding, its reproduction, and its resolution in the devlog. **Feed lessons forward**: each finding class becomes a standing probe in future reviews ("probe TOCTOU windows", "check every export's file mode"). Past runs' reviews got measurably smarter over cycles this way — but only because the lessons were written down.
