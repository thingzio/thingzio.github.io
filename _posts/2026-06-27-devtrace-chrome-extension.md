---
title: "DevTrace in a GitHub Tab You Already Have Open via Chrome Extension"
description: A new Chrome extension puts the DevTrace contributor trust score next to every GitHub username --- inline, in the page, with no API token required for the basic score.
date: 2026-06-27
---

The dashboard at [devtrace.thingz.io](https://devtrace.thingz.io) is where the
full scorecard lives. The [GitHub Action](https://github.com/marketplace/actions/devtrace-pr-check)
is where the score shows up automatically on a PR. Between those two surfaces
there is a gap: the case where a reviewer is reading a GitHub page --- an issue
thread, a contributor list, a PR description with a `@mention` --- and wants to
know who they are looking at, right now, without leaving the tab.

The new [DevTrace for GitHub](https://chromewebstore.google.com/detail/devtrace-for-github/aphijidiljkmfonlfmjfogfnkkcdfjjj)
Chrome extension fills that gap. Install it, browse `github.com` like you always
do, and a small gauge badge appears next to every contributor link. Click the
badge and a card opens in place with the contributor's letter grade, numeric
score, and one-line risk summary. The link out goes to the full breakdown on
devtrace.thingz.io. That is the whole interaction.

**No DevTrace account is required to use the extension. No API token is required
to see a score. The basic experience --- badge, click, grade, summary --- works
anonymously out of the box.**

That last sentence is the one that matters. The rest of this post is about why
it is true, and what changes (and what does not) when you choose to add a token.

## The two-second loop

The interaction the extension is optimizing for is short enough to describe in
two sentences. A reviewer reading a GitHub page sees a username they do not
recognize; they click the gauge badge next to it; a card opens with the
contributor's score; they read the one-line risk summary and either move on or
follow the link to the full scorecard.

That is it. There is no settings flow on first install. There is no sign-up
screen. There is no "connect your GitHub account" step. The extension scans the
page, finds the contributor links, attaches a badge next to each one, and waits
for a click.

The basic score comes from the same DevTrace API that backs the dashboard and
the GitHub Action. The grade, the numeric value, and the risk summary are the
same numbers a reviewer would see if they opened `devtrace.thingz.io/score/<user>`
in a new tab --- the extension just saves them the tab.

## Why anonymous works

The DevTrace API serves a public, anonymous endpoint for every GitHub
contributor it can score. The basic response includes the three fields the card
displays: grade, score, and risk summary. No authentication is required because
no private data is being returned --- everything in the basic response is
derived from public GitHub signals that anyone could reproduce with enough patience
and rate-limit budget. DevTrace's job is to do the reproduction once and serve
the result.

The anonymous tier is free. It is rate-limited per IP, which means the badge
will not melt the API if a reviewer happens to scroll through a contributor
graph with five hundred names on it --- the extension caches results for five
minutes in the background service worker, and the API itself rejects bursts.
But the rate limit is generous enough for normal browsing. A reviewer opening a
handful of cards across an afternoon will never see a `429`.

The token-gated fields exist for a different use case: deeper review, repo
context, the full signal surface. None of them are necessary to answer the
question the badge is trying to answer in the moment, which is "what grade does
this person have."

## What the token adds (and when you would want it)

The extension's options page accepts an optional DevTrace API token (a string
prefixed with `dt_`, generated at [devtrace.thingz.io/settings](https://devtrace.thingz.io/settings)).
When present, the extension attaches it to every score request, and the card
expands accordingly.

With a token, three things change:

**The card shows the per-category score breakdown.** Code Provenance, Identity,
Engagement, Community, Behavioral --- the five categories that compose the
trust score, rendered as labeled bars inside the card. The basic anonymous
response does not include the breakdown; the authenticated response does.

**The card includes repo context when the page has one.** If you click the
badge on a page that is inside an `owner/repo`, the extension passes that
context to the API and the card reflects it: commits-in-repo, contributor count,
verified-commit ratio, whether the contributor is an org member. Without a
token, the repo is not sent at all --- DevTrace will not accept the parameter
on an unauthenticated request, and the extension does not try.

**The AI sensing surface and behavioral signals appear when relevant.** The
synthetic-contributor flags and the AI-authenticity narrative are token-gated
on the API side, so they only show up in the card when an authenticated request
returns them.

If you are using the extension to read a contributor's basic trust signal on
the fly, the anonymous mode is enough. If you are using it as a fast lookup for
deeper review --- skimming a contributor graph for synthetic-account patterns,
checking repo-specific context inline --- the token is worth setting up.
Either is a valid use of the extension, and the install path is the same in
both cases.

## What the extension can see, and what it cannot

The privacy story is short enough to fit in a paragraph: the extension only
talks to `devtrace.thingz.io`. It does not have permission to talk to anything
else. The only data sent to DevTrace is the username you clicked (and, if you
have configured a token, the `owner/repo` of the page you are on). No browsing
history. No page content. No GitHub credentials. No cookies. No form data.

That is enforced by the manifest, not by promises. The extension declares
exactly two permissions: `storage` (to persist the optional token in
`chrome.storage.sync`) and `host_permissions` for `https://devtrace.thingz.io/*`.
There is no `tabs` permission, no broad host access, no content-script injection
into anything other than `github.com`. The Chrome permission model means the
extension cannot reach anything it did not declare, and the declaration is
public in the
[repository](https://github.com/thingzio/devtrace-extension/blob/main/manifest.json).

Inside the page, the score card renders into an isolated Shadow DOM and uses
`textContent` for every dynamic value. Even if a malicious response somehow made
it past DevTrace's own validation, it could not inject markup into the GitHub
page around it. The Shadow DOM also means the card's styling does not interact
with GitHub's stylesheet --- which is what makes the badge look the same across
GitHub's light theme, dark theme, dimmed theme, and the high-contrast variants.

The full [privacy policy](https://github.com/thingzio/devtrace-extension/blob/main/PRIVACY.md)
lives in the repository alongside the code. It is short, written in plain
language, and reads the same as the actual behavior because the extension is
open source and the behavior is verifiable.

## Where the badge appears

The extension scans every page on `github.com` for contributor links --- the
specific selectors it looks for are `a.author` and any anchor with
`data-hovercard-type="user"`, which together cover essentially every place
GitHub renders a username with a link to a profile. That includes PR authors,
issue authors, comment authors, the contributor avatars on the sidebar, the
network graph, the contributors page, the org members page, and the `@mention`
links inside issue and PR bodies.

The badge does not attach to avatar-only links --- the small profile images
that wrap a username link without their own text. Avatars and text links often
point to the same profile, and attaching a badge to both clutters the page
without adding signal. The extension picks the textual link and ignores the
duplicate avatar one.

A `MutationObserver` re-scans the page whenever GitHub's own SPA navigation
loads new content, so the badge appears on dynamically-loaded sections (a long
issue thread the user is scrolling, an infinite-scrolling list of PRs) the same
way it appears on the initial page load. The scanning is debounced behind a
`requestAnimationFrame` so it does not fight GitHub's own rendering loop.

## What it does not do

Three things the extension deliberately does not do, in case the absence is
surprising.

**It does not auto-open cards.** A badge is a badge --- a small visual marker
next to the username. The score card only renders when the user clicks. No
hover-to-open, no auto-loading the score for every name on the page, no
background fetching of usernames the user never asked about. The API call only
happens on the click. The reasoning is partly about rate-limit budget, partly
about respecting the user's attention, and partly about respecting DevTrace's
own infrastructure --- a contributor graph with five hundred usernames should
not trigger five hundred score fetches just because the user scrolled past it.

**It does not modify GitHub.** The badge is appended next to a username link;
the link itself is untouched. Removing the extension reverts the page to
unmodified GitHub. There is no setting in the options page that changes
GitHub's behavior or appearance beyond the badge and the card.

**It does not gate the score behind a sign-up.** Some browser extensions for
GitHub require an account before they will show anything useful. This one does
not. The most common path through the extension --- install, click a badge,
read the score --- never asks the user for an email address, a token, or any
other identifier. The token exists for the deeper-review use case and is
strictly opt-in.

## How it composes with the rest of DevTrace

The extension is the third surface for the same scoring engine, and the three
surfaces are deliberately complementary.

The [dashboard](https://devtrace.thingz.io) is the place to go when a reviewer
wants the full picture: every category, every signal, the AI-sensing narrative,
the cross-VCS identity check, the license footprint, the repo-context table.
It is the deep-read surface.

The [GitHub Action](https://github.com/marketplace/actions/devtrace-pr-check)
is the place where the score shows up without anyone having to think about it.
A PR opens, the action runs, the comment lands in the thread, and any reviewer
reading the thread sees the score whether or not they sought it out. It is the
ambient-signal surface.

The Chrome extension is the in-between case --- the reviewer who is already
on GitHub, already reading the page, who wants the score for someone the action
did not run for (because the page is not a PR, or because the repo does not use
the action). It is the on-demand-lookup surface.

A team using all three sees the same scoring engine in three places: in the
PR comment when the action runs, in the inline badge on every other GitHub
page, and on the dashboard for the deeper read. The extension is the cheapest
of the three to install (one click on the Web Store, no configuration), and the
basic experience is free.

## Try it

The extension is in the Chrome Web Store at
[chromewebstore.google.com/detail/devtrace-for-github](https://chromewebstore.google.com/detail/devtrace-for-github/aphijidiljkmfonlfmjfogfnkkcdfjjj).
Install it, open any GitHub page with a contributor link, click the gauge
badge. No account, no token, no signup --- the basic score is there.

If you want the per-category breakdown, repo context, and the AI-sensing
fields, create a token at [devtrace.thingz.io/settings](https://devtrace.thingz.io/settings)
and paste it into the extension's options page. The basic score keeps working
either way.

The source is on GitHub at
[github.com/thingzio/devtrace-extension](https://github.com/thingzio/devtrace-extension)
under MIT. The build is reproducible: `npm ci && npm run build` from a clean
checkout produces the same `dist/` that is published to the Web Store. The
manifest, the permission declarations, and the privacy policy are all in the
repository.

If you are evaluating the projects the contributors work on rather than just
the contributors themselves, [DevPulse](https://devpulse.thingz.io) is the
project-level companion. A high-trust contributor in a struggling project is a
different read from the same contributor in a healthy one, and the two views
are most useful together. DevTrace tells you who. DevPulse tells you how the
project is holding up around them.
