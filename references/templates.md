# State-file templates

These files are the loop's entire memory. A cold session with zero conversational history must be able to read them and run the next cycle correctly. Keep them current — updating them is part of landing a slice, not optional bookkeeping. After writing any of them, read the file back to confirm the write persisted.

## CHARTER.md

```markdown
# Charter: <project name>

**Mode:** directed | freestyle
**Started:** <date>

## Target user
Who actually uses this, concretely. "Developers" is not an answer; "a developer who wants X while doing Y" is.

## Core use case
The one workflow this must support well. One paragraph.

## Non-goals
What this project deliberately does not do. Be specific — non-goals are what keep cycles small.

## Milestones
Ordered, each sized so its slices fit single cycles.
- M0: walking skeleton — <smallest end-to-end usable slice>
- M1: ...

## Definition of done
The observable state at which the loop enters wrap-up mode. In directed mode, derive
this from the PRD. In freestyle mode, write it before cycle 1 and keep it modest.

## Run budget
The maximum cycles this run may consume before pausing for renewal. Use the
user's limits verbatim if they gave any at invocation; otherwise the defaults:
freestyle 10 cycles, directed 25 cycles or the milestone list, whichever comes
first. On exhaustion: land in-flight work, clean tree, cancel any scheduled
loop, request renewal in REQUESTS.md and chat. Continuation is opt-in.

## Stop criteria
Conditions that force wrap-up (or a pivot decision) even if the definition of done
is not met. Checked every cycle, deeply every 5 cycles.

Freestyle-mode defaults — include all of these unless there is a written reason not to:
- Stop after 10 cycles unless a compelling, concrete use case has emerged.
- Stop if no target user can be named specifically.
- Stop when the next slice is polish rather than new capability or learning.
- Stop if the project is technically valid but nobody — including the user — would run it.

Directed-mode defaults:
- Stop when the definition of done is met.
- Pause to REQUESTS.md if the PRD turns out to be infeasible or self-contradictory
  in a way charter distillation didn't surface.
- Any single roadmap item unresolved after 3 cycles forces a pivot/pause/re-scope
  decision (this guard applies in both modes).

## Ambiguity resolutions
Directed mode only — omit this section entirely in freestyle mode (there is no
source document to be ambiguous). Each ambiguity found in the source document
and how it was resolved. This is the section the user is approving at sign-off.
```

## ROADMAP.md

The single source of truth for "what's next." Any recurring or wake-up prompt must derive the next action from this file, never name a task itself.

```markdown
# Roadmap

## Next up
1. <one-cycle-sized slice> (M1)
2. ...

## Done
- [cycle 3] <slice> — <commit sha>
- ...

## Cut / deferred
- <item> — <why>
```

Slices are one-cycle-sized: one coherent, shippable increment with clear expected behavior. If a slice needs more than one commit to make sense, split it in the roadmap first.

## docs/DEVLOG.md

One entry per cycle, appended, newest last. This is where the loop's judgment lives.

```markdown
## Cycle N — <date>
- **Shipped:** <slice> (<commit sha>)
- **Verification:** <suite result as you reproduced it, smoke test performed>
- **Review:** <findings + how each was reproduced and fixed, or "clean">
- **Lesson:** <anything durable — a failure class to probe next time, a spec gap>
- **Continue?** <yes and why / pivot / wrap-up — the honest answer to step 2>
```

Every 5th cycle and at every milestone boundary, expand the **Continue?** line into the deep reassessment: who benefits now, what real workflow it supports, whether the next slice is product work or polish, whether a human would fund another cycle.

## REQUESTS.md

The only human channel. Checkbox protocol: you write unchecked items; the human checks them and may add notes. Check for responses at the start of each cycle.

```markdown
# Requests for the human

- [ ] <date, cycle N — blocking|non-blocking> <what is needed and why, exactly how to provide it>
```

Only block the loop on items marked blocking, and only mark blocking what you truly cannot proceed without. A blocking item **exits the loop entirely** — scheduled safety net canceled, one clear final message stating the need and how to resume. The loop never idles awake waiting on a human; resumption is the human checking the box and re-invoking the skill.

## CLAUDE.md (project root — the implementer's standing constraints)

This file is read by the implementer sub-agent on every task. It functions as an always-on lean-code and security policy, which past runs found more effective than repeating rules in each delegation prompt. Adapt the specifics to the chosen stack; keep the shape:

```markdown
# Project constraints

- <dependency policy, e.g. "stdlib only" or "no new dependencies without a spec saying so">
- All SQL parameterized. No string-built queries.
- Product code must not spawn subprocesses or make network calls unless the spec
  says so. (Running the test suite to verify your own work is expected — this
  constrains the code you write, not your tooling.)
- File creation that must not clobber uses atomic create-exclusive (e.g. O_EXCL),
  never check-then-open.
- All file I/O specifies an encoding explicitly.
- Tests run against temp directories, never the repo or home directory.
- Errors surface as clean messages, never raw tracebacks, and never destroy user data.
- Match the existing code style. No abstractions beyond what the spec requires,
  no defensive code for conditions the spec says cannot occur, no drive-by refactors.
- Never run git. Never touch files outside the ones the task names.
```
