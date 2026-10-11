# Spec: Software factory on Matt Pocock's skills

Decisions from the grilling sessions that rewrote `BUILD_PROMPT.md`. Written in
the `/to-spec` template. `BUILD_PROMPT.md` is the build instruction; this file
records what was decided and why.

## Glossary

| Term               | Meaning                                                                                                                                  |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------- |
| **Issue**          | A record in the issue tracker. It holds a request, a bug or a ticket. Never "issue ticket".                                              |
| **Request**        | An issue labelled `enhancement`: a feature or improvement. It is grilled into a spec and tickets.                                        |
| **Bug**            | An issue labelled `bug`. It is triaged and reproduced.                                                                                   |
| **Spec**           | The `/to-spec` document written into a request.                                                                                          |
| **Ticket**         | An AFK slice of a request's spec, created by `/to-tickets` as a sub-issue labelled `ready-for-agent`. Built inside a job; never a job.   |
| **Job**            | One factory run, with one `job.json` and one dashboard card. Types: feature, bug, triage, audit, retro.                                  |
| **Review request** | The pull request or merge request (or local file) that a feature or bug job ends in.                                                     |
| **Report**         | The `report.html` that a triage, audit or retro job writes.                                                                              |
| **State label**    | One of Matt's five triage states: `needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`.                         |

## Problem Statement

The original factory was autonomous from a one-line request to a merge into
`main`. A `spec-writer` agent guessed at anything ambiguous and recorded the
guesses as Assumptions, and an `approver` agent could merge without a human. The
owner wants more human control: every request should be agreed through a
grilling session before anything is built, and every change should be approved
by a human before it merges. The owner also wants the factory to follow Matt
Pocock's engineering skills (triage, grilling, specs and tickets, TDD, two-axis
code review, bug diagnosis, architecture audits, retrospectives) while keeping
those skills and their label vocabulary as intact as possible, and to work with
any issue tracker, not only GitHub.

## Solution

The factory becomes an issue-tracker-driven pipeline with a human at both ends:

1. **Triage (scheduled, nightly).** The factory runs the unattended parts of
   Matt's `/triage` over new issues: it applies the category, checks whether
   the issue is already implemented or was rejected before, reproduces bugs, and
   posts triage notes with a recommended state. It decides nothing.
2. **Intake (human and agent, interactive).** In `/factory status` the owner
   accepts, rejects or grills each recommendation. `/factory grill` turns a
   request into a spec and tickets (or a bug into an agent brief) and, on the
   owner's yes, marks it `ready-for-agent`.
3. **Build (unattended).** `/factory next` builds the oldest ready request with
   `implement-spec` (implementers using `tdd` on tickets in parallel), or
   `/factory next bug` fixes the oldest ready bug with `diagnosing-bugs`. It
   runs `code-review` once, fixes the findings, checks that the review really
   ran, and opens a review request written with the `pr` skill and labelled
   `ready-for-human`.
4. **Approval (human).** The owner runs `/factory approve` to merge and close
   the request and its tickets, or `/factory rework` to send it back.
5. **Maintenance (scheduled, weekly).** An architecture audit and a
   retrospective each leave an HTML report on the dashboard and an issue for
   the owner to close. An unclosed one is closed as `superseded` by the next run
   of the same type.

The build prompt ends by running `/setup-matt-pocock-skills`, which asks which
issue tracker to use.

## User Stories

1. As the owner, I want new issues triaged every night, so that each morning they have a category and a recommendation waiting for me.
2. As the owner, I want nightly triage to check whether a request is already implemented or was rejected before, so that I don't grill duplicates.
3. As the owner, I want nightly triage to reproduce bugs, so that I only spend time on bugs that are real.
4. As the owner, I want nightly triage never to mark anything ready, close anything or apply `wontfix`, so that every decision stays mine.
5. As the owner, I want nightly triage to recommend a grilling session when a request needs one, so that I know where my input is needed.
6. As the owner, I want `/factory status` to list issues by state with the nightly recommendations, so that I can work through them in one place.
7. As the owner, I want to accept, grill or reject each recommendation with one word, so that triage decisions are quick.
8. As the owner, I want every request grilled with me before it is built, so that the factory builds what I mean rather than what it guessed.
9. As the owner, I want the spec written into the original request, so that the request and its spec stay in one place.
10. As the owner, I want the spec broken into tickets with blocking edges during the interactive session, so that I approve the breakdown and the build can run unattended.
11. As the owner, I want to either apply `ready-for-agent` myself or tell the factory at the end of `/factory grill` that it can, so that nothing is built until I decide it is ready, without an extra step when I already have.
12. As the owner, I want the factory to use Matt's five state labels unchanged, so that his skills work without modification.
13. As the owner, I want `/factory next` to build the oldest ready request and never pick up a ticket, so that I can work through the queue without naming issues.
14. As the owner, I want `/factory next` to skip a request that already has an open review request, so that nothing is built twice.
15. As the owner, I want `/factory <ref>` to build a specific request or bug, so that I can choose the order when I need to.
16. As the owner, I want independent tickets built in parallel, so that larger requests finish sooner.
17. As the owner, I want implementers to work test-first at the seams the spec names, so that the code comes with tests worth keeping.
18. As the owner, I want `code-review` to run exactly once per build, so that review is thorough without repeated loops.
19. As the owner, I want the factory to verify that `code-review` actually ran before it opens a review request, so that a skipped review is caught.
20. As the owner, I want every review request written with the `pr` skill and labelled `ready-for-human`, so that each one shows a summary, evidence and a merge-danger call, and I can find the ones waiting on me.
21. As the owner, I want `/factory approve` to merge into `main` and close the request and its tickets, so that one command finishes the job.
22. As the owner, I want `/factory rework` to read my review comments and address them, so that I can send a change back without editing it myself.
23. As the owner, I want `/factory next bug` to take the oldest ready bug, so that bugs have their own queue.
24. As the owner, I want bugs fixed with the `diagnosing-bugs` loop, so that every fix starts from a reproduction and ends with a regression test.
25. As the owner, I want a blocked job to comment on its issue and move it back to `needs-triage`, so that I know a decision is missing.
26. As the owner, I want a weekly architecture audit with an HTML report on the dashboard, so that I see deepening opportunities without asking.
27. As the owner, I want `/factory codebase audit` to tell me how stale the latest report is and offer to rerun it, so that I don't act on outdated findings.
28. As the owner, I want to go from an audit report straight into grilling a candidate, so that an audit finding can become a request.
29. As the owner, I want a weekly retrospective with an HTML report on the dashboard, so that I see how to improve the repo and the factory.
30. As the owner, I want the retrospective to separate repo-environment changes from factory changes, so that I can act on each independently.
31. As the owner, I want an audit or retro I haven't closed to be closed as `superseded` when the next one runs, so that only the latest report needs my attention.
32. As the owner, I want the factory to work with GitHub, GitLab, local markdown or another tracker, so that the blueprint fits any repo.
33. As the owner, I want the build to finish by running `/setup-matt-pocock-skills`, so that I choose my tracker during setup.
34. As the owner, I want several factory sessions to run at once without taking the same issue, so that I can build in parallel.
35. As the owner, I want a dashboard with a swimlane per job type, so that I can see every job's stage at a glance.

## Implementation Decisions

### Starting point
- The new prompt describes the factory as it currently works (one git worktree per job, one `job.json` per job, ports per job) and builds on that. The original prompt's single `board.json` is dropped.

### Terminology
- The glossary above is used in every factory file and copied to the top of the factory skill. "Issue ticket" is not used: an issue is a request, a bug or a ticket.

### Labels and lifecycle
- Requests and bugs use Matt's five state labels and his `enhancement` / `bug` categories, unchanged.
- The factory adds only `factory:audit`, `factory:retro` and `superseded`. Labels considered and dropped: `ready-for-implementation`, `needs-human`, `factory:in-progress`, `factory:review`.
- A request goes: no state or `needs-triage` → (nightly triage recommends) → `/factory grill` → `ready-for-agent` → job → review request → `/factory approve` → closed. A bug goes the same way, with triage reproducing it and either a one-word accept or `/factory grill` producing an agent brief.
- Tickets carry `ready-for-agent`, as `/to-tickets` applies it.
- Review requests carry `ready-for-human`, Matt's meaning for a PR ready for a human to merge. On a request or bug, `ready-for-human` keeps Matt's meaning (a person must implement it) and the factory never builds it.
- An issue has at most one state label; the factory removes the old one when it sets a new one.

### Triage
- Nightly `/factory triage --scheduled` runs an adapted copy of `/triage` (scheduled mode: gather context, redundancy and prior-rejection checks, verify/reproduce) through a `triager` agent over issues with no state or `needs-triage`.
- It applies the category, may apply `needs-info` when the reporter must answer something, otherwise sets `needs-triage`, and posts triage notes with Matt's AI disclaimer and a recommended state. A recommendation for a grilling session reads "grill".
- It never applies `ready-for-agent`, `ready-for-human` or `wontfix`, and never closes an issue.
- It is a `triage` job with a summary report on the dashboard, no tracker issue, and each night's card replaces the previous one.
- `/factory status` lists issues by state with recommendations, plus running and blocked jobs and review requests waiting on the owner. `accept` applies the recommended state through the triage copy's apply mode; `grill` starts `/factory grill`; `wontfix` runs the apply mode's wontfix outcome.

### Skills
- The skills the owner runs interactively (`/grill-with-docs`, `/to-spec`, `/to-tickets`, `/triage`, `/setup-matt-pocock-skills`, plus `grilling` and `domain-modeling`) come from the installed `mattpocock-skills` plugin.
- The skills the factory runs unattended are adapted copies inside the factory: `triage`, `implement-spec`, `tdd`, `code-review`, `diagnosing-bugs`, `pr`, `codebase-design`, `improve-codebase-architecture`, `retro`. Reason: Matt marks several of these user-invoked only, and a skill cannot invoke a user-invoked skill. Copies also make unattended behaviour fixed and reviewable. Each copy records its source commit.
- Adaptations are limited to removing questions to the user and writing outputs to the job folder.

### Intake
- `/factory grill <ref>` runs grilling with domain modeling (glossary and ADRs updated as decisions are settled). For a request it writes the spec in the `/to-spec` template into the same issue under `## Spec`, then runs `/to-tickets` with the owner. A small request may have one ticket. For a bug it writes an agent brief in `/triage`'s format.
- It ends by asking "Mark <ref> ready now?" and applies `ready-for-agent` only on an explicit yes. Otherwise the issue stays at `needs-triage` and the owner applies the label later.

### Build
- `/factory next` takes the oldest issue labelled `ready-for-agent` and `enhancement` (`bug` for `next bug`) that is not a sub-issue. It skips any issue with an open review request or an existing job folder and moves to the next one.
- Sessions claim an issue by creating its job folder (an atomic `mkdir`), not with a label.
- Requests: `implement-spec` with `implementer` agents (sonnet, using `tdd`) in per-ticket worktrees and a `merger` agent (sonnet) onto the integration branch.
- Bugs: one `diagnoser` agent (opus) following `diagnosing-bugs`.
- Review: `code-review` runs **once**, with its Standards and Spec checks as two agents (opus). One implementer fixes every finding in a single pass. The owner rejected a second review pass.
- The security, UX and UI reviewers, the `spec-writer`, the `builder` and the `approver` are removed. `code-review` is the only automated review.
- Verification: before the review request opens, the factory checks that `code-review.md` exists with both sections, covers the integration branch up to the last ticket merge, and is followed by a fix commit or states there were no findings. A failed check blocks the job.
- Blocked jobs: comment on the issue, change its state from `ready-for-agent` to `needs-triage`, keep the worktrees, set the job stage to `blocked`. The owner re-grills.

### Approval
- Every review request is written with the `pr` skill, includes `Closes <ref>`, and is labelled `ready-for-human`.
- `/factory approve <review>` checks mergeability and CI, squash-merges into `main`, closes the request and its tickets, and cleans up worktrees. It does not submit a formal approval review, because a host does not allow approving your own pull request.
- `/factory rework <review>` reads the review comments, fixes them with one implementer, and replies on each thread. It does not run `code-review` again.

### Scheduled jobs
- Run locally from cron (launchd on macOS) with `claude -p`: triage nightly, audit on Monday morning, retro on Friday afternoon, at minutes off the hour. `/factory schedule` proposes the entries and installs them after confirmation.
- Audit and retro runs write `report.html` to their job folder and open a `factory:audit` or `factory:retro` issue. Closing that issue is how the owner closes the job.
- Superseding: a new audit or retro run closes any still-open issue of its type with `superseded` and a link to the new job. A retro with no factory jobs since the last one skips its run.
- Audit runs scan without grilling and record the date and `main` commit they scanned.
- Retro covers factory jobs and session transcripts since the last retro, and reports in two sections, Repo environment and Factory, most severe first. It never applies changes.

### Interactive audit
- `/factory codebase audit` reports the latest audit's age in days, commits to `main` since, and which candidates' files changed since. It offers a rerun.
- The owner picks a candidate and grills it (Matt's normal flow, with glossary and ADR updates). The factory then asks whether to turn it into a factory job. If yes, it creates a request with a spec and tickets and asks "Mark ready now?", as `/factory grill` does, and links it from the audit issue. The audit issue stays open until the owner closes it.

### Tracker and code host
- All tracker operations follow the tracker file written by `/setup-matt-pocock-skills`, and label strings come from its triage-labels file. The factory names no specific tracker.
- The code host is detected from the git remote, independent of the tracker: GitHub pull request, GitLab merge request, or a local review request file when there is no remote.
- The build prompt checks that the plugin is installed and stops if not. It ends by running `/setup-matt-pocock-skills`. If the Skill tool refuses it because it is user-invoked only, the agent reads its instructions and follows them in the same session. The three factory labels are then created in the chosen tracker.

### Jobs, worktrees and ports
- Feature and bug job ids come from the issue number and a short slug (`042-signup-rate-limit`), or the slug alone on a local tracker. Triage, audit and retro jobs are named by type and date.
- The integration branch and each ticket get their own worktree next to the repo.
- Port: `20000 + 10 × (issue number mod 4000) + ticket`.

### Models
- Orchestrator: opus. Implementers: sonnet. Merger: sonnet. Bug diagnoser: opus. Code-review checks: opus. Triager, auditor and retro: opus.

### Dashboard
- Local HTML file polling every `job.json`. One swimlane per job type (feature, bug, triage, audit, retro) with that type's stages. Feature cards show a chip per ticket and Standards/Spec chips. Blocked jobs and review requests waiting on the owner stand out. Cards link to their issue and review request. Triage, audit and retro cards open `report.html`.

## Testing Decisions

- A good test of this blueprint checks the factory's behaviour from the owner's side, not the wording of generated files.
- Build the factory from `BUILD_PROMPT.md` in a scratch repo with a test suite, once per tracker: GitHub, GitLab, local markdown. For each, check:
  - the build stops when the plugin is missing;
  - setup asks for the tracker and creates the three factory labels;
  - nightly triage applies categories, reproduces a seeded bug, posts recommendations, and never applies `ready-for-agent`, `ready-for-human` or `wontfix` or closes an issue;
  - `/factory status` lists the recommendations and `accept` applies them;
  - `/factory grill` writes the spec into the same issue and creates tickets with blocking edges and `ready-for-agent`;
  - `/factory grill` applies `ready-for-agent` to the request after an explicit yes, and leaves it at `needs-triage` after a no;
  - `/factory next` never picks a ticket, skips a request with an open review request, and ignores issues without `ready-for-agent`;
  - two sessions running `/factory next` at once take different issues;
  - a request with two independent tickets builds them in parallel;
  - `code-review.md` exists with both sections, and deleting it blocks the job;
  - the review request has the `pr` skill's sections, `Closes <ref>` and `ready-for-human`;
  - `/factory approve` refuses on red CI and merges on green, closing the request and its tickets;
  - a seeded bug goes through `diagnosing-bugs` and ships a regression test;
  - running the scheduled audit twice supersedes the first issue;
  - a retro with no new jobs skips its run.
- Prior art: the two example jobs in `factory/jobs/` show the expected shape of job folders.

## Out of Scope

- Automated security, UX and UI reviewers.
- An automated approver agent or any merge without `/factory approve`.
- A second `code-review` pass, and `code-review` during rework.
- Unattended triage decisions: marking ready, `wontfix`, closing.
- New state labels beyond Matt's five.
- Cloud-scheduled runs (Routines) and a dashboard that reads from the tracker. Scheduling and the dashboard are local.
- The factory applying `ready-for-agent` to a request or bug without the owner's explicit yes, or applying retro or audit suggestions itself.
- Updating `README.md` and regenerating `.claude/` and `factory/`. The owner rebuilds those from the new prompt.

## Further Notes

### Where the owner overrode the recommended answer
- **Intake:** the recommendation was a grill stage inside each factory run. The owner chose a separate, issue-tracker-based intake.
- **Triage:** the owner added a nightly unattended triage run and `/factory status` to act on its recommendations.
- **Labels:** the first design used `ready-for-implementation` and `needs-human`. The owner dropped both to keep Matt's five states intact, uses `ready-for-human` on review requests instead of a new label, and rejected a claim label in favour of skipping requests that already have a review request.
- **Ready label:** the recommendation was that only the owner applies it. The owner added the option of telling the factory to apply it at the end of `/factory grill`.
- **Reviewers:** the recommendation was to keep security, UX and UI alongside `code-review`. The owner chose `code-review` only.
- **Merging:** the recommendation was that the owner merges on the host. The owner chose `/factory approve` to merge.
- **Review passes:** the recommendation was up to two `code-review` passes. The owner chose one, with a check that it ran.
- **Audit entry:** the owner added `/factory codebase audit` (freshness check, optional rerun, then grilling).
- **Models:** the recommendation put implementers on opus and code-review on sonnet. The owner chose implementers on sonnet, code-review on opus, and the orchestrator on opus.
- **Trackers:** the owner required tracker independence through `/setup-matt-pocock-skills`, run at the end of the build.

### Open risks
- Copied skills drift from Matt's plugin. Re-copy them deliberately and update the recorded commits.
- Scheduled `claude -p` runs need permissions granted ahead of time. `/factory schedule` asks how.
- The port formula repeats after 4000 issues. Collisions only matter if two such jobs run at the same time.
- Claiming by job folder only works for sessions sharing one checkout. Sessions on different machines could take the same issue.
- The triager's model (opus) was set by analogy with the diagnoser and has not been confirmed by the owner.
