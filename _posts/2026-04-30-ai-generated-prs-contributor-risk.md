---
title: "AI-Generated PRs and the New Shape of Contributor Risk"
description: AI lowers the cost of manufacturing convincing fake contributor histories. Here is what signals still hold up, and what DevTrace looks for in the metadata and the behavior.
date: 2026-04-30
---

AI-generated code has a 2.7x higher vulnerability density than human-written code.
That figure --- from
[CodeRabbit's analysis of 470 real-world pull requests](https://www.coderabbit.ai/blog/state-of-ai-vs-human-code-generation-report),
and consistent with Veracode's 2025 GenAI Code Security Report across more than 100
LLMs --- gets quoted a lot in supply chain conversations. It changes the threat
model on its own.

But the code is the easier half of the problem. The harder half is the contributors.

AI lowers the cost of manufacturing a convincing fake contributor history at scale.
Plausible commit messages. Reasonable-looking patches. Activity patterns that mimic
a real developer over months instead of hours. The same tools that make legitimate
developers more productive make social engineering campaigns cheaper to execute. A
campaign that took two years for xz-utils could, in principle, be parallelized across
a hundred targets by one person with a script and an API key.

This is not an argument against AI-assisted development. AI in the PR pipeline is
normal and net positive. It is an argument that contributor vetting has to evolve
alongside the tools contributors are using --- because the cheap, plentiful version
of "looks like a real developer" is now widely available.

## What changed

Three things shifted in the last 24 months that matter for contributor risk:

**The cost of plausible activity dropped.** Generating a year's worth of believable
commit history --- spread across multiple repos, with appropriate language, with
realistic PR descriptions --- is no longer labor-intensive. It is a prompt and a loop.

**The cost of plausible identity dropped.** Bios, READMEs, blog posts, and even
conversational issue replies can be produced at volume. The textual artifacts that
used to take time to fabricate now take minutes.

**Detection got harder for humans.** Reviewers cannot reliably distinguish
AI-generated code from human-written code in a single PR. Pattern-matching at scale
across a contributor's entire history is the only reliable place to look, and humans
do not do that pattern-matching by default.

What still holds up: longitudinal behavioral signals. The thing AI is bad at faking
is the texture of how a real developer interacts with the open source ecosystem over
years --- the rhythm of activity across timezones, the messy inconsistency of real
human attention, the fingerprints of being embedded in multiple unrelated communities.

That is where [DevTrace](https://devtrace.thingz.io)'s AI sensing layer focuses.

## Two tiers of detection

DevTrace's AI sensing operates in two tiers. Tier 1 is metadata --- explicit and
implicit signatures that an AI tool was involved. Tier 2 is behavioral --- patterns
that distinguish a real developer (with or without AI assistance) from a synthetic
account.

The two tiers answer different questions. Tier 1 asks "was this PR produced with AI
help?" Tier 2 asks "is this contributor a real person?" Those questions are not the
same, and conflating them is how teams end up either flagging legitimate Copilot
users or missing actual synthetic accounts.

## Tier 1: metadata analysis

Tier 1 runs on every contributor scan. It looks at the artifacts that AI tooling
leaves behind in commits, PRs, and account configuration.

**Bot detection.** GitHub exposes an account type field that distinguishes user
accounts from bot accounts. Known bots --- Dependabot, Renovate, GitHub Apps --- are
flagged automatically and excluded from contributor trust analysis. This is table
stakes, but it matters because dashboards that ignore bot status tend to inflate
contributor counts.

**Commit trailers.** AI coding assistants increasingly add `Co-authored-by:` trailers
identifying themselves. Claude Code, Cursor agents, GitHub Copilot Workspace, and
several others use predictable trailer formats. DevTrace parses commit messages for
these trailers and tracks the proportion of a contributor's commits that were
co-authored by an AI tool.

This is informational, not punitive. A contributor whose commits are 80% co-authored
by Claude Code is not less trustworthy --- they are just transparent about their
workflow. The signal becomes interesting when combined with other tier 2 signals.

**Tool signatures.** Beyond trailers, AI tools leave fingerprints in commit message
formatting, branch naming conventions, and PR description structure. These signatures
are weak individually but useful as a population-level indicator: a contributor whose
PRs all match a single AI tool's default output format is using that tool heavily,
which is a fact worth knowing.

**PR authenticity classification.** This combines the above into a per-PR
classification: human-authored, AI-assisted, or AI-generated. The classification is
probabilistic, not absolute. The point is to give reviewers context, not to gate
merges.

## Tier 2: behavioral analysis

Tier 2 is the harder problem and the part that actually distinguishes real from
synthetic. These signals come from
[GH Archive](https://www.gharchive.org/) data, which gives DevTrace longitudinal
visibility into contributor activity across all of GitHub --- not just the repo in
question.

**Velocity anomaly ratio.** Real developers have rhythm. They commit in bursts during
focused work, go quiet during meetings, sleep, take weekends. Their velocity over a
30-day window has texture --- peaks, valleys, patterns that correlate with
calendar days.

A synthetic account often does not. Either the velocity is too smooth (an automated
loop producing one PR per day at consistent intervals) or it is too spiky in
suspicious ways (zero activity for weeks, then a flood of PRs across multiple repos
in 48 hours). DevTrace computes the ratio of actual velocity to expected velocity for
that account's age and activity level. Ratios well outside the typical range are
flagged.

**Active hour spread.** A real developer working a normal job tends to commit during
a recognizable subset of the 24-hour day --- their working hours, plus or minus
their personal habits. The histogram of their commit timestamps has a shape: usually
two or three peaks (morning, afternoon, occasional evening) with quiet zones for
sleep and lunch.

A synthetic account often has either a uniformly flat distribution (commits at every
hour, because the script does not sleep) or a too-narrow distribution (commits only
in a 2-hour window, because that is when the operator runs the script).

This signal is noisier than it sounds --- contributors travel, change jobs, work
across timezones with collaborators --- so DevTrace uses it as one input among many.
But the failure modes (flat or narrow distributions) are distinctive enough to be
useful.

**Burst-vanish score.** This is the signal most directly aimed at the xz-utils
playbook: an account that appears, builds visible activity rapidly, then either
vanishes or pivots to a single high-value target. DevTrace computes a burst-vanish
score that captures the ratio of active days to total account age, weighted by the
concentration of activity in specific time windows.

A real developer with five years of patchy activity gets a low burst-vanish score.
An account that is six months old, was active for the last 60 days, has commits in
12 unrelated repos, and is now requesting commit access to a critical library --- that
account gets a high score.

**Synthetic contributor flag.** When tier 2 signals combine in ways consistent with a
manufactured account (low diversity, high burst-vanish, narrow active hour spread,
no review participation, sparse profile), DevTrace surfaces a synthetic contributor
flag. The flag is not a verdict. It is a "this contributor warrants closer review
before granting elevated trust" signal.

## What a real high-velocity contributor looks like

The hardest population to score correctly is the legitimate high-velocity
contributor. These are real developers --- often working full-time on open source,
or using AI assistants aggressively --- who can produce more PRs in a week than a
typical contributor produces in a quarter.

They share several characteristics that distinguish them from synthetic accounts:

- **Long account history with consistent rhythm.** The high velocity is part of a
  multi-year pattern, not a recent spike.
- **Activity across many repositories.** They review code, file issues, and comment
  in unrelated communities. They are embedded.
- **Realistic active hour distribution.** A shape, with quiet zones, that holds up
  over months.
- **Other signals of human presence.** Followers, profile, blog posts, conference
  talks, real-world identity that can be cross-referenced.
- **Conversational depth.** Their issue comments and PR replies show familiarity
  with project history, with the maintainers, with prior decisions. Synthetic
  accounts struggle here because the context window is too small to fake.

DevTrace's behavioral category is specifically tuned to surface this distinction. A
contributor with 50 PRs in 30 days but a five-year account history, broad repo
diversity, consistent review participation, and a normal active-hour shape will score
well. The same 50 PRs from a 6-month-old account with no review participation, no
repo diversity, and a flat hourly distribution will score poorly --- and trigger the
synthetic contributor flag.

## What still slips through

No detection layer catches everything. The honest list of what does not work yet:

**Hybrid accounts.** A real developer with a mature account who decides to run a
malicious campaign is the hardest case. Their behavioral fingerprint looks normal
because it is normal. This is where DevTrace's repo-context signals --- code
provenance, commit signing, organizational role --- carry more weight than the
behavioral category.

**Slow-burn synthetic accounts.** A patient adversary willing to invest two years in
building authentic-looking history can defeat the burst-vanish signal by simply not
bursting. The xz-utils attack worked partly because the timeline was patient. The
trade-off is that this kind of attack does not scale --- it is back to being
expensive, which is itself a deterrent.

**Coordinated networks.** A cluster of synthetic accounts that interact with each
other to fake community signals (followers, code reviews, mutual mentions) can
defeat per-contributor analysis. Detecting these requires graph-level analysis that
DevTrace does not yet do at the public-tier level. It is on the roadmap.

This is the same dynamic as any other detection problem. The cheap, common attacks
get caught. The expensive, patient attacks remain hard. The point is to raise the
floor.

## How to use this in practice

For most teams, the practical question is not "did AI write this code?" It is "is
this contributor someone whose elevated access I should trust?"

A reasonable workflow:

**For first-time contributors:** run a DevTrace scan as part of PR review. The trust
score and any synthetic contributor flags surface in the GitHub Action output. Most
first-time contributors are fine. The point is to catch the cases that warrant a
second look before merging.

**For contributors requesting commit access:** run a deeper scan that includes the
behavioral signals. Anyone whose burst-vanish score is high or whose active hour
spread looks synthetic should not get commit access on the strength of a few good
PRs alone, regardless of how good those PRs are.

**For your own contributor base:** periodically audit the contributors with elevated
access against current trust scores. Account behavior changes. A contributor who
scored well three years ago may have changed accounts, gone dormant, or been
compromised. Trust is not a one-time check.

## What this is not

DevTrace's AI sensing is not a tool for penalizing AI-assisted development. The
proportion of legitimate PRs touched by AI tools is high and rising, and any
detection system that treats AI involvement as suspicious by default will produce
mostly false positives.

It is also not a substitute for code review. The point is to surface contributors
whose pattern of behavior diverges from what real developers look like --- so that
human reviewers know where to spend their attention. The reviewer still does the
review.

The asymmetry that matters: synthetic contributor accounts are cheap to create and
expensive to vet manually. Behavioral signals are the only way to keep the vetting
cost from blowing up as the creation cost falls. That is the gap DevTrace is trying
to close.

## Try it

DevTrace is free to use. Score any public GitHub contributor at
[devtrace.thingz.io](https://devtrace.thingz.io) --- the AI sensing layer runs
automatically as part of the trust score. Tier 1 metadata signals are available on
the free tier. Tier 2 behavioral signals are available on higher plans. The
[GitHub Action](https://github.com/thingzio/devtrace-action) integrates the same
scoring into your PR workflow.

If you are also watching the project-level side of this ---
[DevPulse](https://devpulse.thingz.io) detects sudden shifts in project contribution
patterns that can signal automated activity at the repository level: contributor
mix changes, velocity spikes, review ratio drops. Project-level anomalies and
contributor-level anomalies are usually correlated. Looking at both at once is how
you tell the difference between "the project just got popular" and "something is
off."
