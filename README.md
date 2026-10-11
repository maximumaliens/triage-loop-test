# triage-loop-test

> **Experimental.** This first release publishes no evaluation results and makes no claims
> about time, tokens or money saved.

A Claude Code skill and two subagents that review one ticket brief. A **simplifier** proposes
cuts, a separate **checker** attacks the whole result, and your main session adjudicates
between them. The skill delivers one **checked version** of the brief and a record of what
changed and why. It is read-only: nothing is implemented, committed or changed in your tracker.

The goal is the *least justified total work* that still meets the ticket as filed. Shorter
prose doesn't count, and neither does work moved to another ticket or to the owner.

## Origin

This skill reviews briefs produced by the `/triage` skill in
[Matt Pocock's skills repository](https://github.com/mattpocock/skills)
([`skills/engineering/triage`](https://github.com/mattpocock/skills/tree/main/skills/engineering/triage)).
`/triage` moves issues through triage states. When an issue reaches `ready-for-agent`, it
writes an agent brief: the specification an unattended agent will work from. Sometimes that
brief asks for more work than the problem needs. This loop was written to test the brief
before anyone builds it.

It is an independent add-on, not affiliated with or endorsed by Matt Pocock. It works on any
ticket brief held in a git repository, but it was written for `/triage` output. See
[Credits and licence](#credits-and-licence) for provenance.

## Contents

```
skills/triage-loop-test/SKILL.md      the orchestrator's procedure (runs in your main session)
skills/triage-loop-test/briefing.md   the run briefing template: the single source for the run's terms
agents/loop-simplifier.md             the simplifier subagent
agents/loop-checker.md                the checker subagent
LICENSE                               MIT
```

## How the loop works

### Roles

- **Orchestrator:** your main Claude Code session, following `SKILL.md`. It sets up the run,
  sends work to the two agents, checks their key evidence itself, decides what is accepted,
  and writes the output.
- **`loop-simplifier`:** revises the brief toward less total work. It backs every change with
  a ledger row covering what breaks if the change is made, the evidence and the replacement
  work.
- **`loop-checker`:** reviews the whole revised brief, not just the changes, against the
  problem statement, the binding items and the repo's floors. It returns ACCEPT,
  ACCEPT-WITH-FIXES or REJECT.

The same pair is kept for the whole run and continued with `SendMessage`, so each agent keeps
the evidence it has already verified.

### Setup (the run's only pause)

The orchestrator first checks the checkout:
- it is a git repository with `HEAD` at the pinned commit;
- `git status` is completely clean, untracked files included;
- the candidate, the problem statement, and every standing binding source and floor you named
  are readable at the pinned commit.

It then scans for anything outside the pinned history (other branches, tags, stashes,
worktrees, and ignored data the brief cites) and adds it to the exclusion list. Next it maps
the binding items and fills in `briefing.md`. It posts a one-screen setup summary listing the
inputs, the standing binding sources and floors as you supplied them, the binding items
found, the open scope, the exclusions, the output file and the budget. Then it waits for your
confirmation. After that the loop runs without stopping.

The **evidence boundary** is committed content at the pinned commit and its history, plus any
inputs you name. Untracked files, ignored data, other refs, the network and live systems are
excluded. A claim that rests on excluded material is reported as unverifiable.

### Each cycle

1. The simplifier receives the current checked version (the original brief in cycle 1), the
   record so far, and the orchestrator's last adjudication. It returns a revised brief
   (section A), a ledger, a net work delta, semantic boundaries, proposals and evidence limits.
2. The checker reviews the simplifier's whole output. The text it reviewed becomes the new
   **checked version**.
3. The orchestrator verifies key evidence with read-only commands, then marks each gap and
   suggested cut **accepted**, **open** or **rejected** (with a reason).
4. The orchestrator posts a cycle record. It covers substantive changes, work restored or
   added, the verdict, whether there was a net reduction, `quiet cycles: n of 2` and
   `cycle n of <budget>`.

A **substantive reduction** removes or shrinks a criterion, test, file, code path, owner step,
agent run or maintenance burden. It counts net of any work restored, and only after the checker
has verified it. Rewording and moved work don't count. A **quiet cycle** has no substantive
accepted reduction.

### Preservation of owner rulings

Some of the brief is **binding**: the owner decisions you name, plus everything recorded in the
repo's standing binding sources. Binding text is kept word for word. If either agent thinks
binding text is wrong, it writes a **proposal** naming the text, the conflict, the proposed
change and the evidence. A proposal never enters the brief. It is listed for the owner to
decide. Inside a triaged ticket, only rulings, definitions, ADRs and other items of a standing
binding source are binding. The agent brief, its acceptance criteria and the rest of the
triage output stay open to challenge.

Additions need a concrete correctness reason with cited evidence. "Safer", "more thorough" and
"the original had it" are not reasons.

### Standing binding sources and floors

Both are inputs you supply. Neither has a default: if one is missing, the skill asks for it,
and "none" is a valid answer. Once supplied, they are treated as follows:

- **Standing binding sources** are where your repo records decisions that always bind, for
  example recorded owner rulings, an approved glossary such as `CONTEXT.md`, and ADRs.
  - Everything recorded in them at the pinned commit is binding for the run, whether or not
    the brief cites it.
  - Setup maps the items the brief cites or changes.
  - Both agents keep that text as written and may challenge it only by proposal. The checker
    flags any contradiction, and any challenge adopted when it should only have been proposed.
- **Floors** are the repo rules every version of the brief must meet, named by file and
  section. Examples: acceptance criteria written as testable properties with concrete
  artifacts, or a failing test first for each hazard a change introduces.
  - The briefing gives both agents the same list.
  - The checker judges each floor against the rule's own text and also checks that the
    brief's semantic boundaries are resolved.

Both must be committed at the pinned commit. Setup stops and names the input if one cannot be
read there.

### Stopping conditions

The run ends with exactly one of three outcomes, using these labels word for word:

- **Stopping condition reached:** two consecutive quiet cycles within the budget, with nothing
  accepted still waiting to be checked. The loop stopped finding cuts.
- **Budget exhausted:** the cycle limit (default 10) was reached first. The loop stopped on
  the limit, not because it ran out of cuts. Cuts suggested after the limit are not applied.
  They are listed as "cuts not taken".
- **Blocked:** a named blocker, such as a missing input, evidence out of reach, a binding
  conflict the owner must settle, or closing rounds that don't settle.

**Closing rounds:** a run never ends with an accepted fix that hasn't been checked. If the
final cycle leaves an accepted correctness fix, the simplifier applies only that fix and the
checker reviews the whole brief again. Closing rounds don't count against the budget and never
add cuts. If a third closing round still accepts a fix, the run stops as Blocked.

### Checked output

Only a checked version is ever delivered. If the last check was a REJECT, the most recent
checked version that wasn't rejected is delivered instead, and the summary says so. The output
file is the only file the skill writes. Its default path is
`docs/issues/<id>-test/<id>-simplified-brief.md` in the checkout:

```
# <ID> loop-test brief (proposal)
> Pinned <commit>. <N> of <budget> cycles, <M> closing rounds. Outcome: <label>.
> Checked in <cycle k | closing round m>. Nothing applied.

## Brief          the checked version, verbatim
## Run record     cuts and their replacements; work added for correctness; evidence limits
## Unresolved     cuts not taken; unchecked fixes (Blocked only); proposals for the owner;
                  open gaps; residual risks accepted
```

## Requirements

- Claude Code with custom subagents. The skill continues agents with `SendMessage`.
- Access to Opus models (both agents pin `model: opus`).
- The ticket brief and problem statement committed in a git repository.

## Installation

User-level install, available in every project:

```sh
git clone https://github.com/maximumaliens/triage-loop-test.git
mkdir -p ~/.claude/skills ~/.claude/agents
cp -r triage-loop-test/skills/triage-loop-test ~/.claude/skills/
cp triage-loop-test/agents/loop-simplifier.md triage-loop-test/agents/loop-checker.md ~/.claude/agents/
```

For a single project, copy the same files into that project's `.claude/skills/` and
`.claude/agents/` instead. Then start a new Claude Code session. `/agents` should list
`loop-simplifier` and `loop-checker`.

## Invocation

`SKILL.md` sets `disable-model-invocation: true`, so Claude never starts a run on its own. You
start it:

```
/triage-loop-test <arguments>
```

Give these inputs in the arguments or the conversation. The skill asks once, in one message,
for anything missing.

| Input | Meaning | Default |
|---|---|---|
| Checkout and pinned commit | Where the run reads from; `HEAD` must equal the pin | working directory |
| Candidate | The brief under test: a file, or a ticket entry at a commit | — |
| Problem statement | The ticket as filed (commit and ID); the requirement source | — |
| Binding | The owner decisions you name | — |
| Standing binding sources | Where the repo records decisions that always bind (owner rulings, glossary, ADRs), by path | none; asked for, "none" allowed |
| Floors | Repo rules every brief must meet, by file and section | none; asked for, "none" allowed |
| Open | What may be cut or changed | implementation, testing and process work |
| Exclusions | Anything else outside the evidence boundary | none |
| Output file | Where the result is written | `docs/issues/<id>-test/<id>-simplified-brief.md` |
| Budget | Cycle limit | 10 |

Example:

```
/triage-loop-test T42 Use the complete T42 entry in TRACKER.md at a1b2c3d. Preserve my
original prompt and approved rulings. Standing binding sources: recorded owner rulings in
TRACKER.md, CONTEXT.md and docs/adr/. Floors: the "Acceptance criteria" and "Testing"
sections of AGENTS.md. Default limits.
```

Setup needs a clean tree at the pinned commit, so run it in a separate clone rather than your
working checkout:

```sh
git clone <path-to-your-repo> ../myrepo-t42-test
cd ../myrepo-t42-test
git switch --detach a1b2c3d
```

## Model configuration

The main session and the subagents are configured in different places.

| Role | Configured in | Model | Effort |
|---|---|---|---|
| Orchestrator (main session) | your Claude Code session; `SKILL.md` sets neither | the session's model | the session's effort |
| `loop-simplifier` | `agents/loop-simplifier.md` frontmatter | `opus` | `high` |
| `loop-checker` | `agents/loop-checker.md` frontmatter | `opus` | `high` |
| Fallback, if an agent type is missing | `SKILL.md`, step 2 | `Plan` agent type with `model: opus` | not settable; the run says so once |

- **Main session:** the author's normal setting is **High** effort on Opus. This package does
  not set it, and installing the package does not change your settings. The orchestrator does
  the setup checks and adjudication, so its model and effort affect the result.
- **Subagents:** the two agents are configured in this package. Their `model: opus` and
  `effort: high` come from each agent's frontmatter, and don't change with the main session's
  model or effort. `opus` is an alias that Claude Code resolves to its current Opus model.
- Both agents get the `Read`, `Grep`, `Glob` and `Bash` tools, and no `Edit` or `Write`. They
  keep Bash for git, so their read-only use of Bash relies on instruction.

## Comparing the original brief with the reviewed brief

Compare the delivered brief with the **original**, not cycle by cycle. Work the loop added in
one cycle and removed in a later one is not a saving, even if a cycle record lists it as a cut.

| Question | Where to read it |
|---|---|
| Why did the run stop? | The outcome label in the output header, plus the last cycle record's quiet-cycle count. *Stopping condition reached* means the loop ran out of cuts. *Budget exhausted* means it may not have; read "cuts not taken". *Blocked* names its blocker. |
| Is the binding text intact? | Diff each binding passage in the delivered brief against the pinned commit. It should be identical. Any challenge appears under Unresolved as a proposal, not as an edit. |
| Which cuts were accepted? | Run record → *Cuts and their replacements*, each with its replacement work. The cycle records show when each was made. |
| Which cuts were rejected? | The cycle records posted in the conversation ("rejected", with the reason), and Unresolved → *Cuts not taken* and withdrawn proposals. **The output file has no section for cuts rejected during cycles.** Keep the conversation, or copy the cycle records, if you need them. |
| What work was removed? | Net against the original: criteria, tests, files, code paths, owner steps, agent runs, maintenance. Subtract replacement work, including owner steps, follow-up tickets and later sessions. Rewording and merged bullets are not reductions. |
| What did the original get wrong? | Run record → *Work added for correctness*. Each item should cite evidence at the pinned commit. Also check Unresolved → *Open gaps* and *Residual risks accepted*. |
| What couldn't be verified? | Run record → *Evidence limits*. A decision that leans on an unverifiable claim says so there. |

The delivered brief is a proposal. Whether to adopt it, in whole or in part, is the owner's
decision.

## Limitations

- The agents' read-only use of Bash is enforced by instruction, not by tool permissions.
- Setup needs a clean checkout at the pinned commit, and the evidence boundary excludes
  everything uncommitted. Figures that live only in a local database or on a live system can
  be reported but not verified.
- Rejected cuts are recorded only in the conversation's cycle records (see above).
- Every cycle runs both Opus agents, and the main session does the setup and adjudication.
  Cost has not been measured.

## Credits and licence

- **Licence:** the original material in this repository is released under the MIT License.
  See [LICENSE](LICENSE).
- **`/triage`:** the skill this loop reviews is by Matt Pocock, in
  <https://github.com/mattpocock/skills> (MIT License).
- **Writing guidance:** the skill and agent files were drafted with Matt Pocock's
  `writing-for-agents` skill, from the same repository, loaded as guidance. They follow its
  conventions for a user-invoked skill, "done when" completion criteria for each step, and
  reference material kept behind a pointer (`briefing.md`). Anthropic's `skill-creator` skill
  (Apache-2.0) was also loaded while they were drafted.
- **Upstream notices:** this repository contains no upstream files or passages, so no upstream
  licence text is reproduced. The credits above are attribution.
