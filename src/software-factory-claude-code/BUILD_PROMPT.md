Build a simple, visual "software factory" in this repo using Claude Code skills,
subagents and markdown files, built on Matt Pocock's engineering skills
(https://github.com/mattpocock/skills). Keep it small: a viewer should understand
the whole thing in a few minutes. Inspect the repo first so the agents know its
language, test command, and conventions.

The factory is human-in-the-loop at both ends. A human grills every request into
a spec and tickets before anything is built, and a human approves every change
before it merges. In between, the factory builds unattended. On a schedule it
also audits the codebase and runs a retrospective, and leaves both reports for a
human to read.

## Prerequisites

- The `mattpocock-skills` plugin is installed
  (`claude plugin install mattpocock-skills@claude-plugins-official`). If it
  isn't, tell the user and stop.
- The repo has an `AGENTS.md` or `CLAUDE.md` that says how to set up a fresh
  checkout, start the dev server on a given port, and run tests, build and lint.

## Issue tracker and code host

Nothing in the factory names GitHub. The factory works with any issue tracker
`/setup-matt-pocock-skills` supports: GitHub, GitLab, local markdown, or another
tracker the user describes.

- **Tracker operations** (create, read, comment, label, close, sub-issues,
  blocking edges) always follow `docs/agents/issue-tracker.md`, which
  `/setup-matt-pocock-skills` writes. A "label" means whatever that file says a
  label is: a GitHub label, a GitLab label, or a `Status:` line in a local file.
- **Label strings** come from `docs/agents/triage-labels.md`. The factory speaks
  in Matt's roles (`ready-for-agent`, `needs-info`, `ready-for-human`, `bug`,
  `enhancement`) and looks up the string. Recommend the user maps
  `ready-for-agent` to `ready-for-implementation`.
- **Code host** is separate from the tracker and is detected from `git remote -v`:
  GitHub opens a pull request (`gh pr`), GitLab opens a merge request
  (`glab mr`), and no remote means a local review request, the PR body written
  to `factory/jobs/<job-id>/review-request.md`. Below, "review request" means
  whichever of these applies, and `<ref>` means an issue reference in the
  tracker's own form (`#42` on GitHub or GitLab, a slug on local markdown).

## Skills

The human runs Matt's interactive skills straight from the plugin:
`/grill-with-docs`, `/to-spec`, `/to-tickets`, `/triage`,
`/setup-matt-pocock-skills`.

The factory runs the rest unattended, so it ships its own adapted copies under
`.claude/skills/`. Matt marks several of them user-invoked only, and a skill
cannot invoke a user-invoked skill. Copy each from the plugin (or the repo
above), remove `disable-model-invocation`, make the changes listed here and no
others, and record the source commit in `.claude/skills/CREDITS.md`:

| Skill                           | Change for the factory                                                                                               |
| ------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| `implement-spec`                | Uses the `implementer` and `merger` agents. Writes `code-review.md` to the job folder. Asks the user nothing.        |
| `tdd`                           | Seams are the ones the spec's Testing Decisions name; no confirming them with the user.                              |
| `code-review`                   | Uses the `code-reviewer-standards` and `code-reviewer-spec` agents and writes the report to `code-review.md`.        |
| `diagnosing-bugs`               | Used by the `diagnoser` agent. Shows hypotheses in `diagnosis.md` instead of asking the user.                        |
| `pr`                            | Unchanged, plus a `Closes <ref>` line and, for bugs, the hypothesis that turned out correct.                         |
| `codebase-design`               | Unchanged.                                                                                                           |
| `improve-codebase-architecture` | Writes `report.html` to the job folder, not the OS temp dir, and records the date and `main` SHA it scanned.         |
| `retro`                         | Reads factory jobs and session transcripts since the last retro. Writes `report.html`. Never applies a change.      |

The orchestrator is one more skill, `/factory` (`model: opus`). Whichever Claude
Code session runs it becomes the orchestrator for one job: it spawns every agent,
and it is the only writer of that job's `job.json`.

## The loops (edit these to change the factory)

```
INTAKE  (human + agent, interactive)
────────────────────────────────────
  enhancement <ref>                           bug <ref>
        │                                         │
        ▼                                         ▼
  /factory grill <ref>                       /triage  (plugin)
    grilling + domain-modeling                 reproduces the bug,
    spec (to-spec template) → into <ref>       writes an agent brief
    /to-tickets → tickets under <ref>             │
        │                                         │
        ▼                                         ▼
  human applies ready-for-agent             human applies ready-for-agent


BUILD  (unattended)
───────────────────
  /factory next  |  /factory <ref>            /factory next bug  |  /factory <ref>
        │                                         │
        ▼                                         ▼
  ┌──────────────────────────────┐          ┌──────────────────────────────┐
  │ implement-spec               │          │ diagnoser            opus    │
  │   ready tickets in parallel: │          │   diagnosing-bugs loop,      │
  │   implementer  sonnet  (tdd) │          │   regression test first      │
  │   merger       sonnet        │          └──────────────┬───────────────┘
  │   → integration branch       │                         │
  └──────────────┬───────────────┘                         │
                 └──────────────────┬──────────────────────┘
                                    ▼
                    ┌──────────────────────────────┐
                    │ code-review (once)           │
                    │   standards  opus  ┐ in      │
                    │   spec       opus  ┘ parallel│
                    │ one implementer fixes all    │
                    └──────────────┬───────────────┘
                                   ▼
                    verify code-review.md ──fail──► needs-human
                                   │ ok
                                   ▼
                    pr skill → review request "Closes <ref>"
                                   │
                ┌──────────────────┴──────────────────┐
                ▼                                     ▼
       /factory approve <review>             /factory rework <review>
       squash-merge into main,               reads review comments,
       close <ref> and its tickets           one implementer fixes, push


SCHEDULED  (cron, claude -p)                 ON DEMAND  (human)
────────────────────────────                 ──────────────────
  /factory audit --scheduled   Mon, opus       /factory codebase audit
  /factory retro --scheduled   Fri, opus         latest report: age, commits since
    report.html in the job folder                rerun now, or keep it
    + a factory:audit | factory:retro issue      grill a candidate
    the previous open one is closed              → optional new enhancement <ref>
    with the `superseded` label
```

## Target file system

```
.claude/
  skills/
    factory/SKILL.md               # the orchestrator's instructions
    implement-spec/ tdd/ code-review/ diagnosing-bugs/ pr/
    codebase-design/ improve-codebase-architecture/ retro/
    CREDITS.md                     # source commit of each adapted copy
  agents/
    implementer.md                 sonnet  builds one ticket with tdd, in its own worktree
    merger.md                      sonnet  merges a ticket branch into the integration branch
    diagnoser.md                   opus    runs diagnosing-bugs on one bug
    code-reviewer-standards.md     opus    Standards check of code-review
    code-reviewer-spec.md          opus    Spec check of code-review
    auditor.md                     opus    walks the codebase for improve-codebase-architecture
    retro.md                       opus    gathers evidence for retro
factory/
  dashboard.html                   # visual board, polls job.json files
  jobs/                            # gitignored runtime state, one folder per job
    042-signup-rate-limit/
      job.json
      code-review.md
      fixes.md
      review-request.md            # local code host only
      blocked.md                   # only when the job is blocked
    043-login-redirect-loop/       # a bug job
      job.json
      diagnosis.md
      code-review.md
    audit-2026-10-12/
      job.json
      report.html
    retro-2026-10-16/
      job.json
      report.html
docs/agents/                       # written by /setup-matt-pocock-skills
  issue-tracker.md
  triage-labels.md
  domain.md
```

## Jobs

- **Job ids**: features and bugs are `<NNN>-<slug>` from the tracker's issue
  number (zero-padded) and a 2–4 word kebab-case slug, or `<slug>` alone on a
  local-markdown tracker. Audit and retro jobs are `audit-<date>` and
  `retro-<date>`.
- **Worktrees**: the integration branch `factory/<job-id>` lives in
  `../<repo>-factory/<job-id>/`. Each ticket gets `factory/<job-id>-t<NN>` in
  `../<repo>-factory/<job-id>-t<NN>/`. The main checkout is the user's and may be
  dirty; only `/factory approve` on a local code host touches it.
- **Ports**: the dev server port is `20000 + 10 × (issue number mod 4000) + ticket`.
- **Factory labels**, created on first run if the tracker lacks them:
  `factory:in-progress`, `factory:review`, `factory:audit`, `factory:retro`,
  `superseded`.

### job.json

```json
{ "id": "042-signup-rate-limit", "type": "feature", "ref": "#42",
  "title": "Rate limit the signup form",
  "branch": "factory/042-signup-rate-limit",
  "worktree": "/Users/you/code/my-app-factory/042-signup-rate-limit",
  "stage": "tickets",
  "tickets": [ { "ref": "#43", "state": "merged" }, { "ref": "#44", "state": "building" } ],
  "review": { "standards": "pending", "spec": "pending" },
  "review_request": null }
```

- `type` is `feature`, `bug`, `audit` or `retro`.
- `stage` per type:
  - feature: `ready`, `tickets`, `code-review`, `review-request`, `merged`, `needs-human`
  - bug: `ready`, `diagnosing`, `code-review`, `review-request`, `merged`, `needs-human`
  - audit, retro: `report-ready`, `closed`, `superseded`
- Ticket `state` is `pending`, `building` or `merged`. Review values are
  `pending`, `done` or `findings`.
- Rewrite the whole file on every change. Paths are absolute.

## Example agent file

```
.claude/agents/<name>.md  (Claude Code subagent syntax)
---
name: example-agent
description: One line on when the orchestrator should use this agent.
tools: Read, Grep, Glob, Bash, Write
model: sonnet
skills: tdd, codebase-design
---
What this agent reads, what it does, and which file or branch it writes.
```

## Rules

- Agents get context from files and the tracker, not the conversation. Pass
  context pointers (spec, ticket, job folder, worktree, branch, port) instead of
  pasting content.
- Every agent reads `AGENTS.md` (or `CLAUDE.md`) and the domain docs that
  `docs/agents/domain.md` lists, and uses the `GLOSSARY.md` vocabulary.
- **Claiming**: before building, apply `factory:in-progress` to `<ref>`, then
  re-read it. If another session got there first, pick the next item.
- **code-review runs once per build.** One implementer fixes every finding in a
  single pass and writes `fixes.md`. Before the review request opens, the
  orchestrator checks that `code-review.md` exists, has `## Standards` and
  `## Spec` sections, names a diff range ending at the integration branch's last
  ticket merge, and is followed by at least one fix commit or states there were
  no findings. If any check fails, the job goes to `needs-human`.
- **Blocked**: if an implementer or the diagnoser cannot finish (the spec
  contradicts itself, a decision is missing, the environment fails), write
  `blocked.md`, comment on `<ref>` with the reason, swap `ready-for-agent` for
  `needs-info`, keep the worktrees, and set `needs-human`. The human re-runs
  `/factory grill <ref>`, which reads the old spec and the block comment.
- Every review request uses the `pr` skill. The factory never merges or pushes to
  `main` except through `/factory approve`.
- **Superseding**: before a scheduled audit or retro writes its report, it finds
  any open issue of its own type, comments with a link to the new job, applies
  `superseded`, closes it, and sets that job's stage to `superseded`. A retro with
  no factory jobs since the last retro skips its run and supersedes nothing.
- Scheduled runs never ask questions. Interactive steps (grilling, picking an
  audit candidate) happen only in commands a human runs.

## Commands

```
/factory grill <ref>              grill an enhancement into a spec and tickets on <ref>
/factory next                     build the oldest ready enhancement
/factory next bug                 fix the oldest ready bug
/factory <ref>                    build or fix <ref>, by its bug/enhancement label
/factory approve <review>         merge a review request into main and close its issue
/factory rework <review>          send a review request back with the reviewer's comments
/factory codebase audit           read the latest audit, optionally rerun it, then grill a candidate
/factory audit --scheduled        run the architecture audit (for cron)
/factory retro --scheduled        run the retrospective (for cron)
/factory schedule                 show and, once confirmed, install the cron entries
```

### /factory grill \<ref>

Read `<ref>` and its comments. Run the `grilling` and `domain-modeling` skills
from the plugin with the user, updating `GLOSSARY.md` and ADRs as terms and
decisions are settled. When the user confirms there is a shared understanding,
write the spec in Matt's `to-spec` template into `<ref>` under a `## Spec`
heading, below the original request. Then follow `/to-tickets` with the user and
publish the tickets under `<ref>` with their blocking edges. A small feature may
have one ticket. Stop there: the human applies `ready-for-agent`.

### /factory next, /factory \<ref> (enhancement)

Claim, create the job and the integration worktree from `main`, run the fresh
checkout setup from AGENTS.md, then follow the factory's `implement-spec` skill:
implementers on the ready tickets in parallel (each in its own worktree and
port, with `tdd`), a merger after each, more implementers as tickets unblock,
then `code-review` once and one fix pass. Verify `code-review.md`, then open the
review request with the `pr` skill, apply `factory:review`, and set
`review-request`.

### /factory next bug, /factory \<ref> (bug)

Claim, create the job and its worktree, and spawn the `diagnoser` with the
triage brief. It follows `diagnosing-bugs` through all six phases, writes
`diagnosis.md`, and commits the regression test and fix. Then `code-review`
once, one fix pass, verify, and the review request, as above.

### /factory approve \<review>

Check the review request is mergeable and the code host's checks pass (on a
local code host, run AGENTS.md's validation commands in the worktree). If not,
stop and say why. Otherwise squash-merge into `main`, delete the branches,
close `<ref>` and its tickets, remove the job's worktrees, and set `merged`.

### /factory rework \<review>

Read the human's review comments. Spawn one implementer on the integration
branch to address them, push, and reply on each comment thread saying what
changed. No new code-review.

### /factory codebase audit

Find the latest audit job. Tell the user how old it is in days, how many commits
`main` has gained since its SHA, and which of its candidates' files have changed
since. Offer to rerun it now. Then show the candidates, let the user pick one,
and run `improve-codebase-architecture`'s grilling loop with `grilling` and
`domain-modeling`. At the end ask "Turn this into a factory job?" If yes, create
a new enhancement issue, write the spec into it and run `/to-tickets` exactly as
`/factory grill` does, and link it from the audit issue. The audit issue stays
open until the human closes it.

### /factory audit --scheduled, /factory retro --scheduled

Create the job, supersede the previous open one, and run the factory's
`improve-codebase-architecture` or `retro` skill without asking anything. Write
`report.html` and open a `factory:audit` or `factory:retro` issue linking to it.
The retro report has two sections, most severe first in each: **Repo
environment** (`AGENTS.md`, `CODING_STANDARDS.md`, automated checks, tooling) and
**Factory** (its skills, agents, labels and loop).

### /factory schedule

Propose crontab (or launchd on macOS) entries that run
`claude -p "/factory audit --scheduled"` on Monday morning and
`claude -p "/factory retro --scheduled"` on Friday afternoon from the repo root,
at minutes off the hour. Ask the user how the run should get its permissions
(an allowlist in `.claude/settings.json` is recommended) and install the entries
only once they confirm.

## Dashboard

One self-contained HTML file, served with `python3 -m http.server` from factory/.
It polls every `jobs/*/job.json`. One swimlane per job type, with cards moving
across that type's stages. Feature cards show a chip per ticket that fills in as
tickets merge, and a Standards and a Spec chip for code-review. `needs-human`
jobs stand out. Each card links to its issue and review request. Clicking a card
shows its markdown files; audit and retro cards show `report.html` in a panel.
Clean and minimal.

## After building

1. Add `/factory/jobs/` to `.gitignore`.
2. Run `/setup-matt-pocock-skills` so the user picks their issue tracker, triage
   labels and domain doc layout. It is user-invoked only, so if the Skill tool
   refuses it, read its `SKILL.md` from the installed plugin and follow it step
   by step in this session, asking the user each question it asks.
3. Create the factory labels in the chosen tracker, following
   `docs/agents/issue-tracker.md`.
4. Tell the user which code host the factory detected for review requests, and
   that `/factory schedule` sets up the audit and retro runs.
