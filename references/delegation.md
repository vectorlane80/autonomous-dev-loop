# Delegation: contract, lifecycle, recovery

## Why the contract is strict

The implementer is a cheap model. It executes precise specs excellently and fills gaps poorly — under-specification in past runs produced defensive slop (an unreachable custom sanitizer duplicating a library), wrong design calls, and guessing. Meanwhile its self-reports cannot be trusted: implementers repeatedly declared "all green" on red suites and labeled self-inflicted regressions "pre-existing and unrelated." The contract below exists to keep the implementer inside its competence zone and to make its output cheap to verify.

## Choosing an implementer

If more than one coding CLI or model tier is available, a few things hold up in practice. Measure them on your own setup rather than trusting any list — specifics change with every model release, but the shape has been stable:

- **Prefer a dedicated coding CLI at its default or lowest reasoning effort.** On decision-locked implementation — design already settled, spec precise — raising reasoning effort mostly buys latency. The work is transcription, not invention.
- **Choose the quota-exhaustion fallback in advance.** A long run will hit a limit mid-cycle. Deciding what to switch to while the run is stalled wastes the window; make it a routing rule, not a judgment call, and prefer a fallback drawing on a different account or billing pool.
- **Do not use the same tool to implement and to review.** The two reward different behaviour, and the strongest implementer is not reliably the strongest reviewer — in one comparison it was the weakest. Cross-tool review also catches blind spots a model checking its own work will not.
- **Exclude any model that over-reports success, however cheap.** A model that says "all tests pass" on a red suite is not a cheap implementer; it is a source of silent corruption, and no price makes that worth it.

Which implementer you pick matters far less than the verification rule below, which is what makes a wrong choice survivable.

## Splitting the lead: spec author vs loop driver

The lead's job splits cleanly in two: authoring the delegation spec, and driving the loop (dispatching, verifying, re-dispatching). Different models can hold each, and on a full project build the split scored the same as one strong model doing both — while costing less, because the expensive model is invoked once per cycle instead of staying resident through all the repetitive work.

Give the spec author the state files as context each cycle — the brief, `CHARTER.md`, `ROADMAP.md`, the previous cycle's outcome, the current file list, and the one slice you want specced — then dispatch what it returns verbatim. Because the state files carry the context, no message bus or agent-to-agent channel is needed: the repo *is* the handoff, and a plain one-shot invocation per cycle is enough. Feed the previous cycle's outcome forward; a stateless spec service with no history slices noticeably worse.

**The reason to prefer the split is that it checks the spec author.** In one run a spec asserted a specific pseudo-random generator output as ground truth. The value was wrong, and it was about to become load-bearing in a test. The driving model ran it before dispatch, caught it, and corrected the one line. A lead writing its own specs dispatches its own hallucinated constants unchallenged.

So verify any concrete constant a spec asserts — an RNG output, an arithmetic result, an API signature — before it reaches the implementer. Correct only that, log every such deviation, and never silently rewrite a design decision. A factual correction and a redesign are different acts.

## Dispatching the implementer

Prefer a synchronous sub-agent call — the report comes back as the tool result and the loop continues in the same turn, with nothing to track. If you dispatch in the background instead, that also works *when you are the top-level session*: the harness re-invokes you when the child finishes, so end the turn cleanly rather than polling.

The one topology that breaks is **nesting**: if you are yourself running as a sub-agent (someone delegated the whole loop to you), your children's completion notifications may route past you and their reports may be unroutable back to you. In that case dispatch synchronously only; if you can't, treat the working tree as the source of truth and check it for quiescence (stable `git status`/mtimes across a real interval) before reading results — remembering that a half-changed tree usually means the implementer is *still working*, not dead.

## The delegation prompt

Every delegation is self-contained — assume the implementer knows nothing beyond the repo and `CLAUDE.md`. Include every section:

```
## Task
<one slice, one sentence>

## Working directory
<absolute path>

## Design decisions (already made — implement exactly, do not redesign)
<data formats, algorithms, error behavior, security choices. This section is the
quality lever: everything you'd rather not have a cheap model decide goes here.
If you include example code or output, make it agree exactly with the prose —
when they conflict, the implementer follows the example.>

## Files you own
<exact files to create/modify. Everything else is read-only.>

## Do NOT
- Touch files outside the list above, or revert/reformat unrelated code.
- Run git, commit, or push.
- Add dependencies, abstractions, or defensive code the spec doesn't call for.
- <task-specific exclusions>

## Required tests
<the specific behaviors that need test coverage, including the failure cases>

## Verify before reporting
Run: <exact command(s)>
Expected: <exact expected output, e.g. "174 passed, 0 failed">

## Required skills
If installed, apply writing-lean-code (no abstractions, defensive code, or
cleanup beyond this spec) and verification-before-completion (run the
verification commands and report their real output) on this task.
[An implementer on another vendor's CLI cannot load these skills — for those,
inline the rules instead: no abstractions/defensive code/cleanup beyond this
spec; and climb the whole verification ladder — imports, provided tests pass,
then exercise what the spec describes that no provided test covers.]

## If stuck
After 3 failed attempts at any part, stop and report honestly what works, what
doesn't, and what you tried. A truthful partial report is a success; a false
"all green" is the worst possible outcome.

## Report format
- Files changed (list)
- Test command run and its verbatim final line
- Anything you noticed that the spec didn't cover
```

Good delegation tasks are concrete implementation patches: a single slice's code plus tests, focused test additions, doc updates tied to specific behavior. Never delegate product strategy, architecture, security decisions, or debugging — debugging in particular collapsed into guessing when delegated; the lead root-causes (scientific-debugging style, against a live reproduction) and then delegates the *fix* as a normal decision-locked spec.

## The lead re-runs the tests. Always.

The implementer's report is a claim, not evidence. Across a few hundred measured delegations, roughly one in twelve reported success on work that failed a suite it had not seen — and the rate varied enormously by model, from never to most of the time. The most confident report enumerated the exact spec clauses it had skipped and declared them "Verified"; another opened "Perfect! All tests pass."

So, every cycle, before accepting a slice:

1. **Run the verification command yourself.** Do not trust the implementer's exit code or its quoted output — an agent inside the workspace can edit the test file, add a `conftest.py` that rewrites outcomes, or simply print success.
2. **Keep the acceptance check out of the implementer's reach** where you can. What it can see, it will code to — which is why a passing suite it was handed proves less than it looks.
3. **Treat a confident checklist as a smell**, not reassurance. Calibrated reports ("tests pass; I could not exercise X") were reliably honest; triumphant ones were where the false completions lived.

## Implementer lifecycle

**When you are the top-level session, prefer one persistent implementer, resumed each cycle** over fresh spawns. A persistent implementer accumulates conventions and context; in one run it began catching gaps in the lead's own specs by mid-project. Fresh spawns re-derive context every cycle and force longer prompts. Note that resuming an agent is a *background* dispatch — that's fine at top level (the harness re-invokes you when it finishes), which is exactly why this pattern is top-level-only: **when running nested, use a fresh synchronous spawn every cycle instead**, and compensate for the lost context with `CLAUDE.md` plus a short running conventions doc you update as the project develops (the same artifact the cost-ceiling handoff below requires).

**But persistence has a cost ceiling.** Transcript footprint grows every cycle (~8x over one full run). When the implementer's responses slow or its cost per task has roughly quadrupled from early cycles, retire it: write a short handoff doc (conventions adopted, recurring pitfalls, where things live), spawn a fresh implementer, and give it the handoff doc plus `CLAUDE.md` as its opening context.

## Failure recovery

Sub-agents die mid-task — usage limits, session loss, crashes. This happened in every long run. When an implementer dies or returns nothing usable:

1. **Confirm it is actually dead** before treating it as dead: a tree that is still changing between two checks means the implementer is alive and mid-write — leave it alone. (An eval-run lead nearly reverted a live agent's half-written work this way.)
2. **Inspect before assuming.** `git status` and `git diff` against HEAD to learn what actually landed. Never assume the task fully happened or fully didn't — partial application is the common case.
3. Decide per file: keep what's correct and complete, revert what isn't. When in doubt, revert to HEAD — a clean re-dispatch beats archaeology.
4. Re-dispatch with a narrower prompt that states what already exists and what remains.
5. If delegation fails twice on the same slice, record that in the devlog and either re-slice the task smaller or implement that one slice yourself as the lead — honestly noted, not silently. Do not pretend the delegation rule was satisfied when no worker produced the work.
