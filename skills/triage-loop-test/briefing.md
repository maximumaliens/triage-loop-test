# Run briefing template

Fill every `<…>` and send the result as the first message to both `loop-simplifier` and
`loop-checker`. Both agents get the same text. This file is the single source for the run's
terms (binding, open, proposal, substantive, checked version, semantic boundary and evidence
boundary); the agent definitions defer to it.

---

# Run briefing: <ID> loop test

You are one of two agents in a read-only simplify/check loop on one ticket's brief. The checkout
is `<checkout>`, pinned at `<pinned commit>` on branch `<branch>`. Nothing will be implemented.
Your role and return format come from your agent definition.

## Goal

Find the least justified total work that still meets the problem statement.
- Progress means less implementation, testing, maintenance or owner work, once it has been
  checked. The owner is the person who rules on the ticket and decides proposals.
- Shorter prose, fewer bullets, moving work into another ticket or handing it to the owner are
  not progress. Count any replacement work.
- There is no reduction quota and no predetermined answer.

## Inputs

- **Problem statement:** <how to read it, e.g. `git show <commit>:<tracker file>`, lines or
  heading>. It is the requirement source.
- **Candidate:** <how to read it>. This is the brief under test in cycle 1. From cycle 2 on, the
  orchestrator sends the current checked version.

## Binding and open

- **Binding:** these stand.
  - <the owner decisions the user named, e.g. "the owner's <ID> rulings of <date>">;
  - everything recorded at `<pinned commit>` in <the standing binding sources with their
    paths, e.g. recorded owner rulings, an approved glossary such as `CONTEXT.md`, and ADRs;
    or "no standing binding sources">.
  - Binding items found so far: <list from setup, with paths>.
- **Floors:** every brief must meet <the floors, each named with its file and section; or "no
  floors beyond the problem statement">.
- **Open:** <scope, e.g. "implementation, testing and process work">. Every other choice in the
  candidate is open too, whoever wrote it, unless it is binding.
- **Proposal:** a challenge to binding text is written as a proposal, never as an edit to the
  candidate. A proposal names:
  - the binding text;
  - the conflict;
  - the proposed change;
  - its evidence.

  The owner decides proposals.

## Terms

- **Substantive reduction:** it removes or shrinks a criterion, test, file, code path, owner
  step, agent run or maintenance burden.
  - It is counted net of any work restored, and only once the checker has verified it.
  - Prose, reformatting and moved work are not substantive.
- **Checked version:** the exact candidate text the checker last reviewed. Only a checked
  version is ever delivered.
- **Semantic boundary:** a point where the change must decide what something means (what a
  failed measurement counts as, who owns a new exception) and the brief has not decided it.

## Evidence boundary

- **Permitted:**
  - committed content at `<pinned commit>` and its reachable history: `git show <pinned>:<path>`,
    `git grep -n <pattern> <pinned> -- <paths>`, `git log <pinned>`;
  - <other named inputs>.
- **Excluded:** <setup's exclusion list>, plus:
  - anything untracked or ignored;
  - commits outside the pinned history, whatever their date: branches, tags or remote refs
    that `git log --all --not <pinned>` reaches;
  - reflog, stash and worktrees;
  - other checkouts;
  - the network and live hosts.
- **Reported claims:** a claim that rests on excluded material is reported but unverifiable.
  Say so wherever a decision leans on one.

## Working rules

- Read files and run read-only git commands only. Use `git grep`, not `grep -r`.
- Cite evidence as `path:line` at `<pinned commit>`, or as a commit hash.
- Keep each report under about 1,800 words, excluding the candidate text.
