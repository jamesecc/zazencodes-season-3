# Spec: Software factory on Matt Pocock's skills

Decisions from the grilling session that rewrote `BUILD_PROMPT.md`. Written in the
`/to-spec` template. `BUILD_PROMPT.md` is the build instruction; this file records
what was decided and why.

## Problem Statement

The original factory was autonomous from a one-line request to a merge into
`main`. A `spec-writer` agent guessed at anything ambiguous and recorded the
guesses as Assumptions, and an `approver` agent could merge without a human. The
owner wants more human control: every feature should be agreed through a
grilling session before anything is built, and every change should be approved
by a human before it merges. The owner also wants the factory to follow Matt
Pocock's engineering skills (grilling, specs and tickets, TDD, two-axis code
review, bug diagnosis, architecture audits, retrospectives), and to work with
any issue tracker, not only GitHub.

## Solution

The factory becomes an issue-tracker-driven pipeline with a human at both ends:

1. **Intake (human and agent, interactive).** The owner grills each feature
   request with an agent. The result is a spec, written into the request's
   issue, and a set of tickets with blocking edges. The owner marks the issue
   ready, either by applying the label or by telling the factory to apply it at
   the end of the session. Bugs are triaged and reproduced with `/triage`, then marked ready.
2. **Build (unattended).** The factory picks up a ready issue and builds it with
   `implement-spec` (implementers using `tdd` on tickets in parallel), or fixes
   a bug with `diagnosing-bugs`. It runs `code-review` once, fixes the findings,
   checks that the review really ran, and opens a review request written with
   the `pr` skill.
3. **Approval (human).** The owner runs `/factory approve` to merge, or
   `/factory rework` to send the change back with review comments.
4. **Maintenance (scheduled).** A weekly architecture audit and a weekly
   retrospective each leave an HTML report on the dashboard and an issue for
   the owner to close. An unclosed report is closed as `superseded` by the next
   run of the same type.

The build prompt ends by running `/setup-matt-pocock-skills`, which asks which
issue tracker to use.

## User Stories

1. As the owner, I want every feature request grilled with me before it is built, so that the factory builds what I mean rather than what it guessed.
2. As the owner, I want the spec written into the original feature issue, so that the request and its spec stay in one place.
3. As the owner, I want the spec broken into tickets with blocking edges during the interactive session, so that I approve the breakdown and the build can run unattended.
4. As the owner, I want to either apply the ready label myself or tell the factory at the end of the grilling and spec session that it can mark the issue ready, so that nothing is built until I decide it is ready, without an extra step when I already have.
5. As the owner, I want the ready label to read `ready-for-implementation`, so that it matches my wording.
6. As the owner, I want `/factory next` to build the oldest ready enhancement, so that I can work through the queue without naming issues.
7. As the owner, I want `/factory <ref>` to build a specific issue, so that I can choose the order when I need to.
8. As the owner, I want independent tickets built in parallel, so that larger features finish sooner.
9. As the owner, I want implementers to work test-first at the seams the spec names, so that the code comes with tests worth keeping.
10. As the owner, I want `code-review` to run exactly once per build, so that review is thorough without repeated loops.
11. As the owner, I want the factory to verify that `code-review` actually ran before it opens a review request, so that a skipped review is caught.
12. As the owner, I want every review request written with the `pr` skill, so that each one shows a summary visual, before/after evidence, and a merge-danger call.
13. As the owner, I want `/factory approve` to merge an approved change into `main` and close its issue and tickets, so that one command finishes the job.
14. As the owner, I want `/factory rework` to read my review comments and address them, so that I can send a change back without editing it myself.
15. As the owner, I want bugs triaged and reproduced before the factory takes them, so that it fixes the bug I reported.
16. As the owner, I want `/factory next bug` to take the oldest ready bug, so that bugs have their own queue.
17. As the owner, I want bugs fixed with the `diagnosing-bugs` loop, so that every fix starts from a reproduction and ends with a regression test.
18. As the owner, I want a blocked job to comment on its issue and move it to `needs-info`, so that I know what decision is missing.
19. As the owner, I want a weekly architecture audit with an HTML report on the dashboard, so that I see deepening opportunities without asking.
20. As the owner, I want `/factory codebase audit` to tell me how stale the latest report is and offer to rerun it, so that I don't act on outdated findings.
21. As the owner, I want to go from an audit report straight into grilling a candidate, so that an audit finding can become a factory job.
22. As the owner, I want a weekly retrospective with an HTML report on the dashboard, so that I see how to improve the repo and the factory.
23. As the owner, I want the retrospective to separate repo-environment changes from factory changes, so that I can act on each independently.
24. As the owner, I want an audit or retro I haven't closed to be closed as `superseded` when the next one runs, so that only the latest report needs my attention.
25. As the owner, I want the factory to work with GitHub, GitLab, local markdown or another tracker, so that the blueprint fits any repo.
26. As the owner, I want the build to finish by running `/setup-matt-pocock-skills`, so that I choose my tracker during setup.
27. As the owner, I want several factory sessions to run at once without taking the same issue, so that I can build in parallel.
28. As the owner, I want a dashboard with a swimlane per job type, so that I can see every job's stage at a glance.

## Implementation Decisions

### Starting point
- The new prompt describes the factory as it currently works (one git worktree per job, one `job.json` per job, ports per job) and builds on that. The original prompt's single `board.json` is dropped.

### Skills
- The skills the owner runs interactively (`/grill-with-docs`, `/to-spec`, `/to-tickets`, `/triage`, `/setup-matt-pocock-skills`, plus `grilling` and `domain-modeling`) come from the installed `mattpocock-skills` plugin.
- The skills the factory runs unattended are adapted copies inside the factory: `implement-spec`, `tdd`, `code-review`, `diagnosing-bugs`, `pr`, `codebase-design`, `improve-codebase-architecture`, `retro`. Reason: Matt marks several of these user-invoked only, and a skill cannot invoke a user-invoked skill. Copies also make unattended behaviour fixed and reviewable. Each copy records its source commit.
- Adaptations are limited to removing questions to the user and writing outputs to the job folder.

### Intake
- `/factory grill <ref>` runs grilling with domain modeling (glossary and ADRs updated as decisions are settled), writes the spec in the `/to-spec` template into the same issue under a `## Spec` heading, then runs `/to-tickets` with the owner. A small feature may have one ticket.
- At the end of `/factory grill`, the factory asks whether to mark the issue ready. It applies the label only when the owner explicitly says yes in that session. Otherwise it stops and the owner applies the label later. The factory never marks an issue ready on its own judgement.
- Bugs go through Matt's `/triage`, which reproduces the bug before it is marked ready.

### Labels
- The factory uses Matt's role names and reads the actual label strings from the triage-labels file. The ready role maps to `ready-for-implementation`.
- Factory labels: `factory:in-progress` (claiming), `factory:review` (review request open), `factory:audit`, `factory:retro`, `superseded`.

### Build
- Features: `implement-spec` with `implementer` agents (sonnet, using `tdd`) in per-ticket worktrees and a `merger` agent (sonnet) onto the integration branch.
- Bugs: one `diagnoser` agent (opus) following `diagnosing-bugs`.
- Review: `code-review` runs **once**, with its Standards and Spec checks as two agents (opus). One implementer fixes every finding in a single pass. The owner rejected a second review pass.
- The security, UX and UI reviewers, the `spec-writer`, the `builder` and the `approver` are removed. `code-review` is the only automated review.
- Verification: before the review request opens, the factory checks that `code-review.md` exists with both sections, covers the integration branch up to the last ticket merge, and is followed by a fix commit or states there were no findings. A failed check sends the job to `needs-human`.
- Blocked jobs: comment on the issue, swap ready for `needs-info`, keep the worktrees, set `needs-human`. The owner re-grills.

### Approval
- Every change ends in a review request written with the `pr` skill, including `Closes <ref>`.
- `/factory approve <review>` checks mergeability and CI, squash-merges into `main`, closes the issue and its tickets, and cleans up worktrees. It does not submit a formal approval review, because a host does not allow approving your own pull request.
- `/factory rework <review>` reads the review comments, fixes them with one implementer, and replies on each thread. It does not run `code-review` again.

### Scheduled jobs
- Run locally from cron (launchd on macOS) with `claude -p`: audit on Monday morning, retro on Friday afternoon, at minutes off the hour. `/factory schedule` proposes the entries and installs them after confirmation.
- Each run writes `report.html` to its job folder and opens a `factory:audit` or `factory:retro` issue. Closing that issue is how the owner closes the job.
- Superseding: a new run closes any still-open issue of its type with `superseded` and a link to the new job. A retro with no factory jobs since the last one skips its run.
- Audit runs scan without grilling and record the date and `main` commit they scanned.
- Retro covers factory jobs and session transcripts since the last retro, and reports in two sections, Repo environment and Factory, most severe first. It never applies changes.

### Interactive audit
- `/factory codebase audit` reports the latest audit's age in days, commits to `main` since, and which candidates' files changed since. It offers a rerun.
- The owner picks a candidate and grills it (Matt's normal flow, with glossary and ADR updates). The factory then asks whether to turn it into a factory job. If yes, it creates an enhancement issue with a spec and tickets, as `/factory grill` does, and links it from the audit issue. The audit issue stays open until the owner closes it.

### Tracker and code host
- All tracker operations follow the tracker file written by `/setup-matt-pocock-skills`. The factory names no specific tracker.
- The code host is detected from the git remote, independent of the tracker: GitHub pull request, GitLab merge request, or a local review request file when there is no remote.
- The build prompt checks that the plugin is installed and stops if not. It ends by running `/setup-matt-pocock-skills`. If the Skill tool refuses it because it is user-invoked only, the agent reads its instructions and follows them in the same session. The factory labels are then created in the chosen tracker.

### Jobs, worktrees and ports
- Job ids come from the issue number and a short slug (`042-signup-rate-limit`), or the slug alone on a local tracker. Audit and retro jobs are named by date.
- The integration branch and each ticket get their own worktree next to the repo.
- Port: `20000 + 10 × (issue number mod 4000) + ticket`.

### Models
- Orchestrator: opus. Implementers: sonnet. Merger: sonnet. Bug diagnoser: opus. Code-review checks: opus. Audit and retro: opus.

### Dashboard
- Local HTML file polling every `job.json`. One swimlane per job type (feature, bug, audit, retro) with that type's stages. Feature cards show a chip per ticket and Standards/Spec chips. `needs-human` jobs stand out. Cards link to their issue and review request. Audit and retro cards open `report.html`.

## Testing Decisions

- A good test of this blueprint checks the factory's behaviour from the owner's side, not the wording of generated files.
- Build the factory from `BUILD_PROMPT.md` in a scratch repo with a test suite, once per tracker: GitHub, GitLab, local markdown. For each, check:
  - the build stops when the plugin is missing;
  - setup asks for the tracker and creates the factory labels;
  - `/factory grill` writes the spec into the same issue and creates tickets with blocking edges;
  - `/factory grill` applies the ready label after an explicit yes, and leaves it off after a no;
  - `/factory next` ignores issues without the ready label and claims with `factory:in-progress`;
  - two sessions running `/factory next` at once take different issues;
  - a feature with two independent tickets builds them in parallel;
  - `code-review.md` exists with both sections, and deleting it makes the job go to `needs-human`;
  - the review request has the `pr` skill's sections and `Closes <ref>`;
  - `/factory approve` refuses on red CI and merges on green;
  - a seeded bug goes through `diagnosing-bugs` and ships a regression test;
  - running the scheduled audit twice supersedes the first issue;
  - a retro with no new jobs skips its run.
- Prior art: the two example jobs in `factory/jobs/` show the expected shape of job folders.

## Out of Scope

- Automated security, UX and UI reviewers.
- An automated approver agent or any merge without `/factory approve`.
- A second `code-review` pass, and `code-review` during rework.
- Cloud-scheduled runs (Routines) and a dashboard that reads from the tracker. Scheduling and the dashboard are local.
- The factory applying the ready label without the owner's explicit yes, or applying retro or audit suggestions itself.
- Updating `README.md` and regenerating `.claude/` and `factory/`. The owner rebuilds those from the new prompt.

## Further Notes

### Where the owner overrode the recommended answer
- **Intake:** the recommendation was a grill stage inside each factory run. The owner chose a separate, issue-tracker-based intake that ends with the owner labelling the issue.
- **Reviewers:** the recommendation was to keep security, UX and UI alongside `code-review`. The owner chose `code-review` only.
- **Merging:** the recommendation was that the owner merges on the host. The owner chose `/factory approve` to merge.
- **Review passes:** the recommendation was up to two `code-review` passes. The owner chose one, with a check that it ran.
- **Audit entry:** the owner added `/factory codebase audit` (freshness check, optional rerun, then grilling).
- **Models:** the recommendation put implementers on opus and code-review on sonnet. The owner chose implementers on sonnet, code-review on opus, and the orchestrator on opus.
- **Ready label:** the recommendation was that only the owner applies it. The owner added the option of telling the factory to apply it at the end of `/factory grill`.
- **Trackers:** the owner required tracker independence through `/setup-matt-pocock-skills`, run at the end of the build.

### Open risks
- Copied skills drift from Matt's plugin. Re-copy them deliberately and update the recorded commits.
- Scheduled `claude -p` runs need permissions granted ahead of time. `/factory schedule` asks how.
- The port formula repeats after 4000 issues. Collisions only matter if two such jobs run at the same time.
