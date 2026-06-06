---
title: "Four Lines of YAML: Adding Contributor Trust Checks to Every PR"
description: A trust score that lives in a dashboard nobody opens is just a tab. The DevTrace GitHub Action puts the same scoring in the PR comment thread, where reviewers already work --- in four lines of YAML.
date: 2026-06-06
---

Most supply chain controls fire after code lands. Vulnerability scanners run on
merged branches. SBOM generators run during release builds. License auditors run
nightly. By the time any of them flag a problem, the PR is closed, the change is on
`main`, and the conversation has moved on.

The point of contributor trust scoring is to move the check earlier --- to the
moment a reviewer first looks at a PR, when the decision to merge or to scrutinize
is still open. The dashboard at [devtrace.thingz.io](https://devtrace.thingz.io)
does that for the case where a reviewer thinks to check. The
[DevTrace GitHub Action](https://github.com/marketplace/actions/devtrace-pr-check)
does it for the case where they would not have.

Four lines of YAML. One secret. No infrastructure to run.

## The minimum viable workflow

```yaml
- uses: thingzio/devtrace-action@v1
  with:
    token: ${{ secrets.DEVTRACE_TOKEN }}
```

That is the working configuration. Wrapped in a job, with the standard
`pull_request` trigger and the two permissions the action needs, the full file is
this:

```yaml
name: DevTrace PR Check
on:
  pull_request:
    types: [opened, synchronize]

permissions:
  pull-requests: write
  checks: write

jobs:
  score:
    runs-on: ubuntu-latest
    steps:
      - uses: thingzio/devtrace-action@v1
        with:
          token: ${{ secrets.DEVTRACE_TOKEN }}
```

Get the `DEVTRACE_TOKEN` from the API tokens page at
[devtrace.thingz.io/settings](https://devtrace.thingz.io/settings) --- it starts
with a `dt_` prefix --- and add it as a repository secret. That is the install.

The action is published on the GitHub Marketplace. It pins to a major version (`@v1`),
runs on Node 20, ships as a single packaged JavaScript file, and has no other
dependencies a maintainer has to think about.

## What the PR comment shows

On every PR that triggers the workflow, the action posts a single comment. If the
comment already exists --- the PR was updated, the workflow re-ran --- it edits in
place rather than appending a new one. The thread does not fill up with stale
scores.

For a single-author PR, the comment leads with the contributor's grade and score:

```
### DevTrace: alice — A (0.84)
> Long-tenured contributor with broad repo diversity and consistent review
> participation. AI sensing: no synthetic flags.

<details><summary>Score breakdown</summary>

| Category        | Score |
|-----------------|-------|
| Code Provenance | 0.91  |
| Identity        | 0.88  |
| Engagement      | 0.79  |
| Community       | 0.82  |
| Behavioral      | 0.86  |

</details>
```

For a multi-author PR --- a contributor cherry-picking commits from someone else,
a co-authored change, a merged feature branch --- the comment becomes a table with
one row per contributor, each linked to their full scorecard.

The risk summary line is the part that does the most work. It is the one sentence
a reviewer can read while scanning the thread, before deciding whether to open the
scorecard. The category breakdown is collapsed by default. The link to the full
scorecard on devtrace.thingz.io is always there.

## Enforcing a minimum score

The comment alone is informational. To turn the score into a gate, add a single
input:

```yaml
- uses: thingzio/devtrace-action@v1
  with:
    token: ${{ secrets.DEVTRACE_TOKEN }}
    min-score: '0.5'
```

When `min-score` is set, the action also creates a GitHub Check Run named
**DevTrace Score** on the PR's head commit. The check passes if every non-bot
contributor scores at or above the threshold, and fails otherwise. Make it a
required status check in branch protection, and any PR with a contributor below
the threshold cannot be merged until either the threshold is lowered, the
contributor is removed from the diff, or someone with the right access overrides
the check.

The threshold is a judgment call. A `min-score` of `0.5` is roughly "no grade-F
contributors." `0.7` is "all contributors A or B." There is no universally correct
number. The first week of running the action with a sane default, watching what
passes and what fails, is how teams tune it.

## Why bot accounts are auto-excluded

Most repositories of any size have Dependabot, Renovate, or other GitHub Apps
opening PRs constantly. Those accounts have score `0` and grade `F` in DevTrace ---
not because they are untrustworthy, but because they are bots, and the scoring
model is for humans.

If `min-score` enforcement applied to them, every Dependabot PR would fail the
check, every Renovate PR would fail the check, and the threshold would be unusable.
The action recognizes the bot pattern (score 0 and grade F together) and skips
threshold enforcement for those contributors. They still appear in the PR comment
as a regular row with grade F and score 0.00. In the check run summary, the same
contributors are flagged with a `⊘ {username}: bot (skipped)` line, so a reviewer
looking at why the check passed can see which authors were excluded.

The corner case worth naming: a real contributor with a legitimate score of 0
would also be skipped. In practice this does not happen. A real human with any
visible GitHub activity at all scores above 0. If a reviewer needs to handle a
truly unscored human contributor, the comment surfaces it as a "No score available"
row, not a bot skip.

## Deduplicating commit authors

A PR with 30 commits from the same author should not produce 30 API calls.
A merge PR pulling in someone else's work should not produce one API call per
unique committer when the action could have done it in one.

The action collects the PR opener plus every commit author, deduplicates the set
by GitHub login, and scores each contributor exactly once. The single-author case
is the common one and costs one API call. The multi-author case --- the case that
actually matters for supply chain risk, where a maintainer has rebased an external
contributor's commits onto a feature branch --- scores everyone whose code is
about to land, with one call per unique person.

The result is that the cost of running the action scales with the number of
distinct contributors in a PR, not the number of commits.

## What the action does not do

There are three things the action deliberately leaves to the rest of the review
process.

**It does not block merges on its own.** It posts a comment and, when configured,
creates a check run. Whether that check is required is a branch protection
decision the repository owner makes. The action's job is to surface signal. The
gate is owned by the people who run the repo.

**It does not score the code in the diff.** That is what SCA scanners, container
scanners, and the rest of the existing supply chain pipeline do. DevTrace's lane
is the contributor, not the artifact. The two work together --- a low-trust
contributor and a flagged dependency change are a different kind of PR from
either signal alone --- but the action is not trying to replace any of them.

**It does not call out to anything except the DevTrace API and the GitHub API.**
No telemetry, no external services, no third-party rate limits. The token in the
workflow is the only credential it uses. On explicit API errors --- a bad token
(`401`), a forbidden request (`403`), or rate-limit exhaustion (`429`) --- the
step fails outright. On a transient network error or unreachable API, the action
warns, records each affected contributor as "No score available," and the check
run returns a `neutral` conclusion rather than `success`, so the gate is not
silently passed.

## What the first week looks like

The thing maintainers most often want to know before installing a check-run-style
action is what the first week of comments will actually look like.

On a typical repository with a mix of internal contributors and external PRs, the
distribution lands roughly as follows. The team's regular contributors --- people
with multi-year accounts, broad repo footprints, and consistent activity --- score
in the A-and-B range and pass any reasonable threshold without thought. Bot PRs
get the skip indicator and are ignored by the check. First-time external
contributors land across the full distribution: some are themselves long-tenured
open source developers from other communities (A-range), some are newer accounts
who happen to be doing good work (B-range), and a few --- the cases the action
exists for --- are sparse accounts with low repo diversity and no community
footprint (D or F).

The interesting comments are the ones in the C-to-D range. Not bots, not
synthetic, not obvious-A contributors --- just real people whose patterns are
sparse for legitimate reasons (new GitHub account, switched primary email,
contributes mostly via internal forges). The score is doing its job when it
surfaces them. The reviewer is doing their job when they read the scorecard,
decide the explanation is plausible, and merge anyway.

The wrong reaction to a low score is to treat the threshold as a verdict. The
right reaction is to use the score to budget review attention --- to spend more
on the PRs the action flagged and less on the PRs it did not.

## Tuning the threshold

The honest advice on `min-score`:

- Start without it. Run the action in comment-only mode for a week. Watch what
  the typical distribution looks like across the team's actual PR flow.
- Then enable it at a value just below the lowest score of the contributors the
  team considers trusted. If the team's regular contributors all score above
  `0.55`, set the threshold to `0.5`. That catches the synthetic case without
  blocking the team's own work.
- Move it up over time as the team gets used to the signal. There is no
  point setting it at `0.7` on day one and dealing with a week of friction
  before anyone understands what the number means.

The `trusted-orgs` input is the other tuning knob. Comma-separated list of
GitHub organization slugs whose members get a small boost in the score. The
boost is narrow: it floors the contributor's association sub-score (a 5%-weighted
component of the Identity category) at the level of a repository collaborator,
and it only fires when the action is also passing a `repo` input --- which it
does by default, set to the current repository. The effect is that members of
listed orgs are treated as if they had collaborator standing for the purpose of
scoring, even when they have not been explicitly granted it. It is not a blanket
pass, and on its own it will not move a contributor across grade boundaries.
What it does do is keep the team's own people from getting penalized for having
a quiet GitHub footprint outside the org.

## Where this fits in the review culture

The framing this post has avoided is "block bad PRs from being merged." That is
not the right framing.

The right framing is that contributor trust is a piece of context most reviewers
do not have, and the action puts it in the place where they already read
context --- the PR comment thread. Some teams will use the score as a gate. Most
will use it as a signal. Either is a reasonable use of the tool, and the action
supports both with the same install.

The smallest viable integration of contributor trust into existing review
culture is not a new dashboard, a new tool, or a new process. It is a comment in
the PR thread, written by an action that ran when the PR was opened, that tells
a reviewer one thing they did not know before. Four lines of YAML, one secret,
no infrastructure to run.

## Try it

Install from the GitHub Marketplace at
[github.com/marketplace/actions/devtrace-pr-check](https://github.com/marketplace/actions/devtrace-pr-check).
Source at [github.com/thingzio/devtrace-action](https://github.com/thingzio/devtrace-action).
The DevTrace API is free for public repository scoring at
[devtrace.thingz.io](https://devtrace.thingz.io) --- create a token, add the
workflow file, push.

If the reviewer reading the comment also wants context on the project the PR is
targeting --- whether the maintainers are overloaded, whether the review ratio
has been slipping, whether the project is one resignation away from
unmaintained --- [DevPulse](https://devpulse.thingz.io) is the project-level
companion. A low-score contributor opening a PR in a healthy project is a
different proposition from the same PR landing in a project running on fumes,
and the two views are most useful when read together.
