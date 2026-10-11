---
name: triage-loop-test
description: Read-only simplify/check loop on one ticket brief, to find the least justified total work. It runs a pair of Opus agents for up to the cycle budget and delivers the last checked version.
disable-model-invocation: true
---

# Triage loop test

Arguments: $ARGUMENTS

You run a read-only experiment on one ticket's brief, as the **orchestrator**.
- A `loop-simplifier` proposes cuts.
- A separate `loop-checker` attacks the whole result.
- You adjudicate between them, keep the record, and deliver one **checked version**.

Nothing is implemented. The run's terms (binding, open, proposal, substantive, checked version,
semantic boundary and evidence boundary) are defined once, in [briefing.md](briefing.md). Read
it before step 1.

## Inputs

Take these from the arguments and the conversation. Ask once, in a single message, for anything
that is missing.

- the checkout (default: the working directory) and the **pinned commit**;
- the **candidate**: a file, or a ticket entry at a commit;
- the **problem statement**: the ticket as filed, given as a commit and an ID;
- what is **binding**: the owner decisions the user names. Everything recorded in the
  standing binding sources is binding anyway.
- the **standing binding sources**: where the repo records decisions that always bind, such as
  recorded owner rulings, an approved glossary (for example `CONTEXT.md`) and ADRs. The
  **owner** is the person who makes those rulings and decides proposals.
- the **floors**: the repo rules every brief must meet, each named with its file and section.

The standing binding sources and the floors have no default. Ask for them if they are missing;
"none" is a valid answer for either.
- what is **open**. The default is implementation, testing and process work.
- any extra exclusions;
- the output file. The default is `docs/issues/<id>-test/<id>-simplified-brief.md` in the
  checkout.
- the budget: a limit of 10 cycles, a provisional default. Use another number only if the user
  sets one.

## Step 1: Setup

1. **Read the checkout's `CLAUDE.md`** (or `AGENTS.md`), and every standing binding source and
   floor the user named. They all come from the checkout you are testing.
2. **Confirm the inputs.**
   - It is a git repository.
   - `HEAD` is the pinned commit.
   - `git status --short --untracked-files=all` is empty.
   - The candidate and the problem statement are both readable.
   - Each named standing binding source and floor is readable at the pinned commit
     (`git show <pinned>:<path>`).

   If any input is missing, report which one and stop.
3. **Scan for contamination.**
   - `git log --oneline --all --not <pinned>` lists commits outside the pinned history, which
     are excluded objects. Name the refs that hold them with
     `git for-each-ref --contains <commit>`. Old side tags count too.
   - Also check `git stash list` and `git worktree list`.
   - Check ignored data the candidate cites with `git check-ignore <path>`.

   Everything found here joins the exclusion list.
4. **Map the binding items.** List the recorded owner rulings in the candidate's entry and in
   the entries it cites, and the items from the standing binding sources (glossary
   definitions, ADRs and the like) that it cites or changes.
   - A candidate committed at the pinned commit is binding only where its text is a ruling, a
     definition, an ADR or another item of a standing binding source. Its agent brief,
     criteria, size verdict and gates stay open.
   - Note any implementation, testing or process statement written inside binding text. It can
     be challenged only by a proposal.
5. **Fix the checkout if needed.** If it is not ready, show the exact commands:
   - fetch only the needed branch, from a local checkout;
   - create a new branch at the pinned commit;
   - remove leftovers.

   Run them only after the user confirms. In Plan Mode, list them for the user to run instead.
6. **Fill in [briefing.md](briefing.md)** and post a one-screen setup summary:
   - the inputs and the evidence boundary;
   - the standing binding sources and the floors as supplied, each with its path, or "none";
   - the binding items found and the open scope;
   - the exclusions;
   - the output file and the budget.

**Done when** the user confirms the setup summary. This is the run's only pause.

## Step 2: Cycles

Spawn `loop-simplifier` and `loop-checker` once each, and send each the filled briefing as its
first message. Keep the same pair for every cycle, continuing each with SendMessage, so each keeps
the evidence it has already verified.

If either agent type is unavailable, use the `Plan` type with `model: opus`. Add the body of
the installed `loop-<role>.md` (in `~/.claude/agents/` or the checkout's `.claude/agents/`) to
its first message, and say once that effort could not be set.

Each cycle:

1. **Simplifier.** Send it three things:
   - the current checked version (in cycle 1, the candidate);
   - the cycle record so far;
   - your adjudication of the last check: fixes to apply, open items to weigh, and rejected
     items with their reasons.

   Keep its section A verbatim. That text becomes the candidate the checker reviews.
2. **Checker.** Send it the simplifier's whole output and the record. When its verdict
   returns, the reviewed section A is the new checked version, along with that verdict.
3. **Adjudicate.**
   - Before you accept a gap, a disputed cut or a restoration, verify its key evidence yourself
     with a read-only command and cite it. An agent's report is a claim, not an artifact.
   - Mark each gap and each suggested cut as **accepted** (it goes to the next cycle), **open**
     (for the simplifier to weigh) or **rejected** (with the reason).
   - A challenge to binding text joins the **proposals** list and never enters the candidate.
   - Decide whether the cycle made a substantive accepted reduction.
4. **Post the cycle record** in the conversation:
   - the substantive changes;
   - any work restored or added;
   - the checker's verdict;
   - whether there was a net accepted reduction;
   - the run of quiet cycles so far: "quiet cycles: n of 2" (a quiet cycle has no substantive
     accepted reduction);
   - the cycles used: "cycle n of <budget>".

A step is done when the agent's returned output is in hand. A status line is not output.

Notification turns and tool calls are not cycles. Changes accepted after a check are checked in
the next cycle. Once the limit is reached, they are checked in closing rounds.

**Closing rounds.** A run never ends with an accepted fix that hasn't been checked. If the
limit's final cycle ends with an accepted correctness fix still to apply, run closing rounds:

1. Send the simplifier only the accepted fixes, marked "closing round". It applies exactly those
   and changes nothing else.
2. Send the checker the result. It checks the whole candidate as usual, and the reviewed text
   becomes the new checked version.
3. Repeat until a closing check accepts no new fix.

Closing rounds don't count against the limit, add no cuts, and never change the outcome label.
Cuts suggested after the limit are not applied; they go under Unresolved as "cuts not taken".
Record each round as "closing round n, after cycle <budget>". If a third closing round still
accepts a fix, stop as Blocked: the fixes are not settling.

**Stop** for exactly one of three outcomes. Use its label word for word in the final record and
in the delivery.

- **Stopping condition reached.** Two consecutive quiet cycles within the limit, and the last
  check leaves nothing accepted that still needs checking. The loop ended because it stopped
  finding cuts. If this happens on the limit's final cycle, this label applies, not "Budget
  exhausted".
- **Budget exhausted.** The cycle limit was reached before the stopping condition was met. The
  loop ended on the limit, not because it ran out of cuts. Closing rounds checked every fix
  accepted at the limit. Report:
  - the quiet cycles reached (0 or 1 of 2);
  - the closing rounds run;
  - the cuts not taken.
- **Blocked.** A genuine blocker, such as:
  - a missing input;
  - evidence that cannot be reached;
  - a binding conflict the owner must settle before another cycle means anything;
  - closing rounds that do not settle after three.

  Name the blocker.

## Step 3: Delivery

**Pick the version.** Deliver the last checked version. If its verdict was REJECT, deliver the
most recent checked version that was not rejected, and say so.

**Write the output file.** It is the only file the skill creates. In Plan Mode, write to the
plan file instead and tell the user. Use this layout:

```
# <ID> loop-test brief (proposal)
> Pinned <commit>. <N> of <budget> cycles, <M> closing rounds. Outcome: <Stopping condition
> reached | Budget exhausted | Blocked>. Checked in <cycle k | closing round m>. Nothing applied.

## Brief
<the delivered checked version, verbatim: no edits after its check, not even wording>

## Run record
- Cuts and their replacements
- Work added for correctness
- Evidence limits

## Unresolved
- Cuts not taken: suggested after the limit, never applied (with the checker's reasoning)
- Blocked only: fixes still unchecked, with the checker's evidence
- Proposals for the owner against binding text (the text, the conflict, the change, the evidence)
- Open gaps in the delivered version
- Residual risks accepted
```

Then post a short summary in chat:
- the file path;
- the cycle count and the closing rounds;
- the outcome label, word for word;
- the number of unresolved items.

**Done when** the file holds the checked version verbatim, and every cut not taken, proposal and
open gap is listed under Unresolved. Except under Blocked, no accepted fix is left unchecked.

## Working rules

- Read files and run read-only git commands only. The writes are the output file and setup
  fixes the user has confirmed.
- Never run project code, commit, change tickets, invoke `/triage`, or use the network.
- The agents have no Edit or Write tool. They keep Bash for git, so their read-only Bash use
  relies on instruction.
