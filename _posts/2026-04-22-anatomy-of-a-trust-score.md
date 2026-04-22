---
title: "Anatomy of a Trust Score: What 23 Signals Tell You About an Open Source Contributor"
description: How DevTrace evaluates contributor trust across five categories, and what that looks like applied to the xz-utils timeline.
date: 2026-04-22
---

In early 2024, a contributor named Jia Tan planted a backdoor in xz-utils. It was
not a smash-and-grab. It was a two-year campaign: legitimate patches, earned trust,
commit access, then a carefully obfuscated payload that shipped to production
distributions before anyone noticed.

Every tool in the open source security ecosystem --- SCA scanners, container scanners,
SBOM generators --- detected the vulnerability after the fact. None of them evaluated
the contributor before it happened.

This is the problem [DevTrace](https://devtrace.thingz.io) tries to address. Not by
claiming it would have prevented xz-utils (we honestly do not know), but by asking a
different question: instead of scanning the package, what if you scored the person?

## What a trust score actually measures

DevTrace computes a numerical trust score between 0.0 and 1.0, mapped to a letter
grade (A+ through F). The score is built from 23 individual signals grouped into five
categories:

| Category | Weight | What it captures |
|---|---|---|
| Code Provenance | 15% | Are commits cryptographically signed? How mature is the account? |
| Identity | 25% | How old is the account? What is their role in the project? Is the profile populated? |
| Engagement | 25% | How much of the work in this repo is theirs? How recently? Are their PRs getting merged? |
| Community | 15% | Do other developers follow this person? Do they maintain their own projects? |
| Behavioral | 20% | Is their activity consistent over time? Do they review code? Do they contribute across repos? |

None of these categories alone tells you much. A brand-new account is not inherently
suspicious. A sparse profile does not mean bad intent. But the categories interact.
A new account with no profile, no code reviews, commits only to one repo, and a
sudden burst of merged PRs --- that pattern is worth looking at more closely.

## The 23 signals

Here is what actually feeds the model, organized by category.

### Code Provenance

This category only applies when DevTrace has repository context (i.e., you are
scoring a contributor in the context of a specific repo).

- **Verified commit ratio** --- what fraction of this contributor's commits are
  cryptographically signed. Signing does not prove good intent, but it ties commits
  to a verifiable identity. The xz-utils attacker's commits were not signed.
- **Account maturity** --- how old the account is, applied as a maturity factor on
  the verification signal. A two-year-old account with signed commits carries more
  weight than a two-week-old one.

### Identity

- **Account age** --- days since account creation, mapped through a logarithmic curve
  that saturates around two years. This means the difference between 30 days and 180
  days matters more than the difference between 3 years and 5 years.
- **Author association** --- the contributor's relationship to the repository: owner,
  member, collaborator, prior contributor, or first-time contributor. Each maps to a
  different trust level.
- **Org membership** --- whether the contributor belongs to the repository's
  organization, and whether they are in a trusted role.
- **Profile completeness** --- five boolean signals: bio, company, location, website,
  and public email. Each present field adds to the score. This is a weak signal
  individually (anyone can fill in a profile), but its absence in combination with
  other red flags becomes meaningful.

### Engagement

- **Commit proportion** --- what fraction of the repo's total commits belong to this
  contributor, adjusted for the number of contributors (a 10% share in a 3-person
  project means something different than 10% in a 200-person project).
- **Commit recency** --- days since last commit, modeled as exponential decay with a
  half-life that adjusts based on project size. Larger projects tolerate longer gaps.
- **PR acceptance rate** --- the ratio of merged PRs to total closed PRs. A high merge
  rate with a reasonable sample size suggests the contributor's work is consistently
  accepted by reviewers.

### Community

- **Follower ratio** --- followers divided by following, capped at a saturation point.
  This is a rough proxy for whether other developers know and trust this person.
  Not a strong signal by itself, but a useful data point.
- **Public repositories** --- how many repos the contributor maintains. More original
  repos (not forks) suggest someone who builds and ships their own work, not just
  drive-by patches.

### Behavioral

This is where things get interesting. These signals come from
[GH Archive](https://www.gharchive.org/) data, which gives DevTrace a longitudinal
view of contributor activity across all of GitHub --- not just the repo in question.

- **Consistency score** --- how regular is this contributor's activity over time?
  Steady, sustained contribution patterns score higher than erratic bursts followed
  by silence.
- **Review participation** --- how many code reviews has this contributor performed
  in the last 30 days? Contributors who review others' code tend to be embedded in a
  project's community, not just passing through.
- **Repo diversity** --- how many distinct repositories has this contributor been active
  in over the last 90 days? Broader engagement suggests a real developer with real
  interests, not a single-purpose account.
- **Burst rate** --- recent PR activity relative to account age. A six-month-old
  account that suddenly opens PRs across 15 repos is not the same as a five-year-old
  account doing the same thing.
- **Fork ratio** --- what fraction of the contributor's public repos are forks versus
  original work? Accounts that consist almost entirely of forks may not have a
  meaningful development history of their own.

Plus one hard gate: if an account is **suspended** by GitHub, the score is
automatically zero regardless of other signals.

## How the math works

Each signal is normalized to a 0-to-1 range using one of three functions depending
on the signal type:

- **Logarithmic curve** for signals with diminishing returns (account age, repo count).
  The first year of account age matters a lot; the tenth year barely moves the needle.
- **Exponential decay** for freshness signals (commit recency). Recent activity scores
  high; stale activity fades.
- **Linear ratio** for bounded signals (consistency, review count, diversity). Capped
  at a saturation ceiling so outliers do not distort the score.

The normalized signal values are multiplied by their category weights and summed. When
a contributor is scored without repository context (no specific repo provided), the
repo-dependent weights --- code provenance, commit proportion, recency, and author
association --- are excluded and the remaining weights are rescaled so they still sum
to 1.0.

The final number maps to a letter grade. A 0.73 is a B. A 0.57 is a C. Below 0.37
is an F.

## What this looks like on the xz-utils timeline

I want to be careful here. DevTrace did not exist during the xz-utils incident, and
I am not going to fabricate numbers. But I can walk through which signals would have
surfaced concerns based on what we know publicly about the "Jia Tan" account.

**Identity signals that would have flagged:**

The Jia Tan GitHub account was created specifically for the xz-utils campaign. At the
time commit access was granted, the account was roughly two years old --- which
actually clears the account age signal. This is exactly why no single signal is
dispositive. But the profile was sparse: no company, no website, no other visible
community presence. The profile completeness signal would have scored low.

**Engagement signals that would have flagged:**

The account's PR activity was concentrated in a single project. The commit proportion
in xz-utils would have been high relative to the small contributor base, which is not
inherently bad --- but in combination with limited activity elsewhere, it paints a
picture of a single-purpose account.

**Community signals that would have flagged:**

The account had minimal followers and no meaningful following network. No public repos
of their own beyond the target project. The follower ratio and public repo count
signals would both have been low.

**Behavioral signals that would have flagged:**

This is where the model is most relevant. The Jia Tan account showed a pattern that
DevTrace's behavioral category is specifically designed to detect:

- **Low repo diversity** --- activity concentrated in one or two repositories
- **No review participation outside the target project** --- a contributor embedded in
  a real community typically reviews code across multiple projects
- **Low consistency** --- the activity pattern was purpose-driven, not the organic
  rhythm of someone who codes across different projects over years

**What would NOT have flagged:**

Account age would have looked reasonable (two years is past the logarithmic inflection
point). The PR acceptance rate would have been high --- the patches were legitimate and
useful. This is the hard part: social engineering works precisely because the work is
real. No scoring model catches everything.

**The aggregate picture:**

No single signal would have raised an alarm. But the composite score --- a
single-project account with a sparse profile, no community footprint, no code review
activity outside the target repo, and low diversity --- would have landed well below
the scores of typical long-term maintainers. Not an automatic rejection, but a strong
signal that this contributor deserved more scrutiny before being granted commit access.

That is the point. DevTrace does not make access control decisions. It provides data
so that maintainers can make better-informed ones.

## AI sensing: the newer layer

On top of the 23 core signals, DevTrace includes an AI sensing layer that looks for
signs of AI-generated or AI-assisted contributions. This is a separate concern from
trust scoring, but it is increasingly relevant.

The first tier is metadata analysis: checking for co-authored-by trailers from known
AI tools, bot-associated PRs, and known tool signatures. The second tier, available
on higher plans, computes behavioral heuristics --- velocity anomaly ratios, active
hour spread, and a burst-vanish score that flags contributors who appear in a sudden
burst of activity and then disappear.

These are not about penalizing AI-assisted development. Copilot-assisted PRs are
normal. The concern is synthetic contributor accounts --- manufactured identities with
fabricated histories. AI lowers the cost of creating these, and the behavioral
heuristics are designed to distinguish a real developer who uses AI tools from an
account that only exists because of them.

## NIST SSDF mapping

For teams that need to demonstrate compliance with NIST SP 800-218 (the Secure
Software Development Framework), DevTrace maps its signals to eight SSDF practices
across three practice groups. These include code access controls, release integrity
verification, contributor identity provenance, code review participation, activity
consistency, anomaly detection, and contributor maturity assessment.

The mapping is intentionally conservative --- DevTrace describes where its signals
are "relevant to" a practice, never that they "satisfy" it. Compliance is a judgment
call for your organization, not something a tool can declare on your behalf. But
having the data organized against the framework saves time when the auditors come
around.

Full methodology is documented at [devtrace.thingz.io/compliance](https://devtrace.thingz.io/compliance).

## What the score is not

A trust score is not a background check. It does not tell you whether someone is
trustworthy as a person. It tells you whether their observable behavior on a platform
matches the patterns of established, embedded contributors --- or whether it
diverges in ways that warrant closer attention.

A low score does not mean "reject this contributor." A high score does not mean
"trust without review." The score is one input to a decision, not the decision itself.

We got into this because we watched the xz-utils story unfold and realized that
the open source ecosystem had robust tooling for scanning code and packages, but
almost nothing for evaluating the people who write them. DevTrace is our attempt to
close that gap. It is an imperfect attempt --- 23 signals cannot fully capture the
complexity of human behavior --- but it is a starting point.

## Try it

DevTrace is free to use. Score any public GitHub contributor at
[devtrace.thingz.io](https://devtrace.thingz.io). If you want contributor scoring
integrated into your PR workflow, there is a
[GitHub Action](https://github.com/thingzio/devtrace-action) that runs on every pull
request.

If you are also thinking about the project-level side of this ---
[DevPulse](https://devpulse.thingz.io) tracks the health metrics (bus factor,
contributor retention, review ratios) that often precede the kind of maintainer
burnout that made xz-utils vulnerable in the first place. The two tools are
complementary: DevPulse evaluates the project, DevTrace evaluates the people.
