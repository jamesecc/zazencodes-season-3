Build a simple, visual "software factory" in this repo using Claude Code skills,
subagents and markdown files, built on Matt Pocock's engineering skills
(https://github.com/mattpocock/skills). Keep it small: a viewer should understand
the whole thing in a few minutes. Inspect the repo first so the agents know its
language, test command, and conventions.

The factory is human-in-the-loop at both ends. A human grills every request into
a spec and tickets before anything is built, and a human approves every change
before it merges. In between, the factory builds unattended. On a schedule it
also triages new issues, audits the codebase and runs a retrospective, and
leaves the results for a human to act on. Keep Matt's skills and his label
vocabulary as intact as possible.

## Glossary

Use these terms exactly, in every file the factory contains. Copy this section
to the top of `.claude/skills/factory/SKILL.md`.

| Term               | Meaning                                                                                                                                                             |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Issue**          | A record in the issue tracker. It holds a request, a bug or a ticket. Never "issue ticket".                                                                         |
| **Request**        | An issue labelled `enhancement`: a feature or improvement. It is grilled into a spec and tickets.                                                                   |
| **Bug**            | An issue labelled `bug`. It is triaged and reproduced.                                                                                                              |
| **Spec**           | The `/to-spec` document written into a request under `## Spec`.                                                                                                     |
| **Ticket**         | An AFK slice of a request's spec, created by `/to-tickets` as a sub-issue labelled `ready-for-agent`. It is built inside a job. It is never a job.                   |
| **Job**            | One factory run, with one `job.json` and one card on the dashboard. Types: `feature` (one request), `bug` (one bug), `triage`, `audit`, `retro`.                      |
| **Review request** | The pull request or merge request (or local file) that a feature or bug job ends in.                                                                                |
| **Report**         | The `report.html` that a triage, audit or retro job writes.                                                                                                         |
| **State label**    | One of Matt's five triage states: `needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`. An issue has at most one.                          |

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
  in Matt's roles and looks up the string, so it works with renamed labels.
- **Code host** is separate from the tracker and is detected from `git remote -v`:
  GitHub opens a pull request (`gh pr`), GitLab opens a merge request
  (`glab mr`), and no remote means a local review request, the PR body written
  to `factory/jobs/<job-id>/review-request.md`. Below, `<ref>` means an issue
  reference in the tracker's own form (`#42` on GitHub or GitLab, a path or slug
  on local markdown), and `<review>` a review request reference.

## Labels and the issue lifecycle

The factory adds only three labels of its own: `factory:audit`, `factory:retro`
and `superseded`. Everything else is Matt's: the categories `enhancement` and
`bug`, and the five state labels.

```
Request (enhancement)
  no state / needs-triage ──nightly triage──► needs-triage + recommendation   (or needs-info)
        │
        ▼ /factory grill <ref>: grilling, spec, /to-tickets, "Mark ready now?" yes
  ready-for-agent ──/factory next──► job builds ──► review request labelled ready-for-human
        ▲                                │                          │
        └──── job blocked: needs-triage ◄┘                          ▼ /factory approve
                                                                 closed (with its tickets)

Ticket (sub-issue of a request)
  ready-for-agent ──built inside the request's job──► closed when the request closes

Bug
  no state / needs-triage ──nightly triage: reproduce──► needs-triage + recommendation
        │
        ▼ human accepts in /factory status, or grills with /factory grill <ref>
  ready-for-agent ──/factory next bug──► job ──► review request ready-for-human ──► closed
```

- `ready-for-human` keeps Matt's meaning. On a review request it means "ready
  for a human to merge". On a request or bug it means a person must implement it,
  and the factory never builds it.
- The factory changes a state label by removing the old one in the same step.

## Skills

The human runs Matt's interactive skills straight from the plugin:
`/grill-with-docs`, `/to-spec`, `/to-tickets`, `/triage`,
`/setup-matt-pocock-skills`, and the model-invoked `grilling` and
`domain-modeling`.

The factory runs some skills unattended, so it ships adapted copies under
`.claude/skills/`. Matt marks several of them user-invoked only, and a skill
cannot invoke a user-invoked skill. Copy each from the plugin (or the repo
above), remove `disable-model-invocation`, make the changes listed here and no
others, and record the source commit in `.claude/skills/CREDITS.md`:

| Skill                           | Change for the factory                                                                                                                                                    |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `triage`                        | Scheduled mode runs steps 1 and 3 (gather context, redundancy and prior-rejection checks, verify) and posts triage notes with a recommended state. Never waits, grills, applies `ready-for-agent`, `ready-for-human` or `wontfix`, or closes. Apply mode runs step 5 for a decision the human made. |
| `implement-spec`                | Uses the `implementer` and `merger` agents. Writes `code-review.md` to the job folder. Asks the user nothing.                                                             |
| `tdd`                           | Seams are the ones the spec's Testing Decisions name; no confirming them with the user.                                                                                   |
| `code-review`                   | Uses the `code-reviewer-standards` and `code-reviewer-spec` agents and writes the report to `code-review.md`.                                                             |
| `diagnosing-bugs`               | Used by the `diagnoser` agent. Shows hypotheses in `diagnosis.md` instead of asking the user.                                                                             |
| `pr`                            | Unchanged, plus a `Closes <ref>` line and, for bugs, the hypothesis that turned out correct.                                                                              |
| `codebase-design`               | Unchanged.                                                                                                                                                                |
| `improve-codebase-architecture` | Writes `report.html` to the job folder, not the OS temp dir, and records the date and `main` SHA it scanned.                                                              |
| `retro`                         | Reads factory jobs and session transcripts since the last retro. Writes `report.html`. Never applies a change.                                                            |

The orchestrator is one more skill, `/factory` (`model: opus`). Whichever Claude
Code session runs it becomes the orchestrator for one job: it spawns every agent,
and it is the only writer of that job's `job.json`. Every `/factory` command
reads the state labels to decide what it may act on.

## The loops (edit these to change the factory)

```
SCHEDULED  (cron, claude -p, never asks anything)
─────────────────────────────────────────────────
  /factory triage --scheduled   nightly   triager (opus) over issues with no state or needs-triage
  /factory audit --scheduled    Mon       auditor (opus)  → report.html + factory:audit issue
  /factory retro --scheduled    Fri       retro (opus)    → report.html + factory:retro issue
    an audit or retro supersedes the previous open one of its type


INTAKE  (human + agent, interactive)
────────────────────────────────────
  /factory status
    issues by state label, each needs-triage one with its nightly recommendation;
    jobs that are running or blocked; one word per item:
      accept      → apply the recommended state (triage apply mode)
      grill       → /factory grill <ref>
      wontfix     → triage apply mode (close, .out-of-scope/ for requests)

  /factory grill <ref>
    request: grilling + domain-modeling → spec into <ref> → /to-tickets
    bug:     grilling + domain-modeling → agent brief into <ref>
    "Mark <ref> ready now?"  yes → ready-for-agent


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
                    verify code-review.md ──fail──► blocked
                                   │ ok
                                   ▼
                    pr skill → review request "Closes <ref>", ready-for-human
                                   │
                ┌──────────────────┴──────────────────┐
                ▼                                     ▼
       /factory approve <review>             /factory rework <review>
       squash-merge into main,               reads review comments,
       close <ref> and its tickets           one implementer fixes, push


ON DEMAND  (human)
──────────────────
  /factory codebase audit
    latest audit: age, commits since → rerun now, or keep it
    grill a candidate → optional new request, as /factory grill does
```

## Target file system

```
.claude/
  skills/
    factory/SKILL.md               # the orchestrator's instructions, glossary first
    triage/ implement-spec/ tdd/ code-review/ diagnosing-bugs/ pr/
    codebase-design/ improve-codebase-architecture/ retro/
    CREDITS.md                     # source commit of each adapted copy
  agents/
    triager.md                     opus    scheduled triage of one issue
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
    042/                           # a feature job, named by issue number
      job.json
      code-review.md
      fixes.md
      review-request.md            # local code host only
      blocked.md                   # only when the job is blocked
      blocked-1.md                 # an earlier block, kept when the job is re-run
    043/                           # a bug job
      job.json
      diagnosis.md
      code-review.md
    triage-2026-10-12/
      job.json
      report.html
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

- **Job ids**: a feature or bug job is named by its issue number alone,
  zero-padded (`042`), or by the issue's slug on a local-markdown tracker. Two
  sessions taking the same issue therefore always try to create the same
  folder. Triage, audit and retro jobs are `<type>-<date>`.
- **Branches and worktrees**: the integration branch is
  `factory/<job-id>-<slug>`, with a 2–4 word kebab-case slug of the title, in
  `../<repo>-factory/<job-id>/`. Each ticket gets `factory/<job-id>-t<NN>` in
  `../<repo>-factory/<job-id>-t<NN>/`. The main checkout is the user's and may be
  dirty; only `/factory approve` on a local code host touches it.
- **One place**: job state lives in the gitignored `factory/jobs/` of one
  checkout, so the factory runs from one machine and checkout at a time. See
  SPEC.md, Future improvements, for running it from several places.
- **Ports**: the dev server port is `20000 + 10 × (issue number mod 4000) + ticket`.

### job.json

```json
{ "id": "042", "type": "feature", "ref": "#42",
  "title": "Rate limit the signup form",
  "branch": "factory/042-signup-rate-limit",
  "worktree": "/Users/you/code/my-app-factory/042",
  "created_at": "2026-10-12T09:14:03Z", "attempt": 1,
  "stage": "tickets",
  "tickets": [ { "ref": "#43", "state": "merged" }, { "ref": "#44", "state": "building" } ],
  "review": { "standards": "pending", "spec": "pending" },
  "review_request": null }
```

- `type` is `feature`, `bug`, `triage`, `audit` or `retro`.
- `stage` per type:
  - feature: `tickets`, `code-review`, `review-request`, `merged`, `blocked`
  - bug: `diagnosing`, `code-review`, `review-request`, `merged`, `blocked`
  - triage: `running`, `report-ready`, `replaced`
  - audit, retro: `running`, `report-ready`, `closed`, `superseded`
- Ticket `state` is `pending`, `building` or `merged`. Review values are
  `pending`, `done` or `findings`.
- `created_at` is when the job was created (ISO-8601, UTC). `attempt` counts
  runs of the job, starting at 1.
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
- **Picking the next job**: `/factory next` takes the oldest issue that is
  labelled `ready-for-agent` and `enhancement` (`bug` for `next bug`) and is not
  a sub-issue, so tickets are never picked. It skips any issue that has an open
  review request (one whose body says `Closes <ref>`) or a job that is running
  or at `review-request`, and moves on to the next.
- **Claiming**: a session claims an issue with `mkdir factory/jobs/<job-id>`
  (no `-p`). If the folder already exists, read its `job.json`: if the stage is
  `blocked`, re-run that job (below); otherwise another session has the issue,
  so pick the next one.
- **Re-running a blocked job**: when a blocked job's issue is `ready-for-agent`
  again, rename `blocked.md` to `blocked-<k>.md` (the next unused `k`), add one to
  `attempt`, and build again in the same worktrees. Keep the tickets already
  merged into the integration branch only if their text has not changed since
  they merged; reset the rest to `pending`. A bug job starts the diagnosis
  again.
- **code-review runs once per build.** One implementer fixes every finding in a
  single pass and writes `fixes.md`. Before the review request opens, the
  orchestrator checks that `code-review.md` exists, has `## Standards` and
  `## Spec` sections, names a diff range ending at the integration branch's last
  ticket merge, and is followed by at least one fix commit or states there were
  no findings. If any check fails, the job is `blocked`.
- **Blocked**: if an implementer or the diagnoser cannot finish (the spec
  contradicts itself, a decision is missing, the environment fails), or the
  code-review check fails, write `blocked.md`, comment on `<ref>` with the
  reason, change its state from `ready-for-agent` to `needs-triage`, keep the
  worktrees, and set the stage to `blocked`. The human re-grills with
  `/factory grill <ref>`, which reads the old spec and the block comment.
- Every review request uses the `pr` skill and is labelled `ready-for-human`.
  The factory never merges or pushes to `main` except through `/factory approve`.
- **Superseding**: before a scheduled audit or retro writes its report, it finds
  any open issue of its own type, comments with a link to the new job, applies
  `superseded`, closes it, and sets that job's stage to `superseded`. A retro with
  no factory jobs since the last retro skips its run and supersedes nothing.
- Scheduled runs never ask questions. Interactive steps (grilling, deciding a
  triage recommendation, picking an audit candidate) happen only in commands a
  human runs.

## Commands

```
/factory status                   issues by state label with triage recommendations, and jobs; act on each
/factory grill <ref>              grill a request into a spec and tickets, or a bug into an agent brief
/factory next                     build the oldest ready request
/factory next bug                 fix the oldest ready bug
/factory <ref>                    build or fix <ref>, by its enhancement/bug label
/factory approve <review>         merge a review request into main and close its issue and tickets
/factory rework <review>          send a review request back with the reviewer's comments
/factory codebase audit           read the latest audit, optionally rerun it, then grill a candidate
/factory triage --scheduled       triage new issues (for cron)
/factory audit --scheduled        run the architecture audit (for cron)
/factory retro --scheduled        run the retrospective (for cron)
/factory schedule                 show and, once confirmed, install the cron entries
```

### /factory status

List open requests and bugs grouped by state label, oldest first. For each
`needs-triage` issue, show the latest triage recommendation and its one-line
reason. Then list jobs that are running or `blocked`, and review requests
labelled `ready-for-human`. The human answers per item: `accept` applies the
recommended state with the factory's `triage` skill in apply mode, `grill`
starts `/factory grill <ref>`, and `wontfix` runs apply mode's wontfix outcome
(close, and write `.out-of-scope/` for a rejected request). A bug whose
recommendation is `ready-for-agent` needs only `accept`.

### /factory grill \<ref>

Read `<ref>`, its comments and any triage notes. Run the `grilling` and
`domain-modeling` skills from the plugin with the user, updating `GLOSSARY.md`
and ADRs as terms and decisions are settled. When the user confirms there is a
shared understanding:

- For a request, write the spec in Matt's `to-spec` template into `<ref>` under
  a `## Spec` heading, below the original request. Then follow `/to-tickets` with
  the user and publish the tickets under `<ref>` with their blocking edges, each
  labelled `ready-for-agent`. A small request may have one ticket.
- For a bug, write an agent brief in the format of the `triage` skill's
  AGENT-BRIEF.md into `<ref>`.

Finally ask "Mark <ref> ready now?" Apply `ready-for-agent` only on an explicit
yes. Otherwise leave it at `needs-triage`. Never apply it on your own judgement.

### /factory next, /factory \<ref> (request)

Pick and claim, create the job and the integration worktree from `main`, run the
fresh checkout setup from AGENTS.md, then follow the factory's `implement-spec`
skill: implementers on the ready tickets in parallel (each in its own worktree
and port, with `tdd`), a merger after each, more implementers as tickets
unblock, then `code-review` once and one fix pass. Verify `code-review.md`, then
open the review request with the `pr` skill, label it `ready-for-human`, and set
`review-request`.

### /factory next bug, /factory \<ref> (bug)

Pick and claim, create the job and its worktree, and spawn the `diagnoser` with
the agent brief. It follows `diagnosing-bugs` through all six phases, writes
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
a new request, write the spec into it, run `/to-tickets` and ask "Mark ready
now?" exactly as `/factory grill` does, and link it from the audit issue. The
audit issue stays open until the human closes it.

### /factory triage --scheduled

Create the job and mark the previous triage job `replaced`. For each open issue
with no state label or `needs-triage` that has had no triage notes since its
last change, spawn a `triager` with the factory's `triage` skill in scheduled
mode. It applies the category, may apply `needs-info` when the reporter must
answer something, otherwise sets `needs-triage`, and posts triage notes that
start with Matt's AI disclaimer and end with a recommended state and a one-line
reason. A recommendation of a grilling session reads "grill". Write
`report.html`: one row per issue with what was done and what is recommended.
Open no tracker issue for it.

### /factory audit --scheduled, /factory retro --scheduled

Create the job, supersede the previous open one, and run the factory's
`improve-codebase-architecture` or `retro` skill without asking anything. Write
`report.html` and open a `factory:audit` or `factory:retro` issue linking to it.
The retro report has two sections, most severe first in each: **Repo
environment** (`AGENTS.md`, `CODING_STANDARDS.md`, automated checks, tooling) and
**Factory** (its skills, agents, labels and loop).

### /factory schedule

Propose crontab (or launchd on macOS) entries, at minutes off the hour, that run
from the repo root:

- `claude -p "/factory triage --scheduled"` every night,
- `claude -p "/factory audit --scheduled"` on Monday morning,
- `claude -p "/factory retro --scheduled"` on Friday afternoon.

Ask the user how the runs should get their permissions (an allowlist in
`.claude/settings.json` is recommended) and install the entries only once they
confirm.

## Dashboard

One self-contained HTML file, served with `python3 -m http.server` from factory/.
It polls every `jobs/*/job.json`. One swimlane per job type, with cards moving
across that type's stages. Feature cards show a chip per ticket that fills in as
tickets merge, and a Standards and a Spec chip for code-review. `blocked` jobs
and review requests waiting on the human stand out. Each card links to its issue
and review request. Clicking a card shows its markdown files; triage, audit and
retro cards show `report.html` in a panel. Clean and minimal.

## After building

1. Add `/factory/jobs/` to `.gitignore`.
2. Run `/setup-matt-pocock-skills` so the user picks their issue tracker, triage
   labels and domain doc layout. It is user-invoked only, so if the Skill tool
   refuses it, read its `SKILL.md` from the installed plugin and follow it step
   by step in this session, asking the user each question it asks.
3. Create the three factory labels (`factory:audit`, `factory:retro`,
   `superseded`) in the chosen tracker, following `docs/agents/issue-tracker.md`.
4. Tell the user which code host the factory detected for review requests, and
   that `/factory schedule` sets up the triage, audit and retro runs.
