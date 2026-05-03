---
title: "Cross-Platform Identity: When the Same Private Key Signs Two Forges"
description: Why SSH-key fingerprint matches across GitHub, GitLab, Codeberg, and Sourcehut are the strongest cross-platform identity signal in open source --- and what they cannot prove.
date: 2026-05-03
---

Anyone can register a popular username like `torvalds` on GitLab. Or Codeberg. Or Sourcehut. The
GitHub account belongs to who you think it does. The others are first-come, first-served.
Bios are free-text. The "linked accounts" section on a profile is whatever the
contributor typed in. None of these are proof of anything beyond what the contributor
chose to claim.

The same problem applies in reverse. When [DevTrace](https://devtrace.thingz.io) is
asked whether a GitHub contributor also has a GitLab presence, name matching is a guess.
Profile-link extraction is a declaration. Neither is verifiable in the cryptographic
sense.

There is one signal that is. If the same SSH public key appears on a contributor's
GitHub account and on their GitLab, Codeberg, or Sourcehut account, then the same private
key signed operations on each platform. That private key is held by exactly one person
--- or, at most, exactly one breach. Producing it on two systems means controlling it.

This post walks through how DevTrace surfaces cross-VCS identity matches, what the
signal proves, and where its limits are.

## The hierarchy of identity claims

Most open source identity tooling treats one of three signals as "verified":

- **Declared** --- the contributor put a link to their GitLab profile in their GitHub
  bio. Anyone can claim this. Worth treating as a hint, not a fact.
- **Asserted by a third party** --- a Keybase proof, a verified email domain, an OIDC
  issuer. Stronger, because the assertion depends on a system that has its own checks.
  But the chain is only as strong as the attester, and most of the historically popular
  attesters are out of scope or defunct.
- **Cryptographic** --- the same private key signs on both platforms. This is the only
  signal that requires the contributor to demonstrate control, not just claim it.

DevTrace's cross-VCS check sits in the third category. It is not a name match dressed up
as a fingerprint comparison. It is the actual fingerprint comparison.

## How `.keys` endpoints work

Every major code-hosting platform except Bitbucket exposes a public endpoint that
returns the SSH public keys a user has uploaded to their account. The keys are intended
for `ssh` and `git` clients to verify identities at the protocol level, but the
endpoints are unauthenticated and machine-readable:

```
https://github.com/{user}.keys
https://gitlab.com/{user}.keys
https://codeberg.org/{user}.keys
https://meta.sr.ht/~{user}.keys
```

Each returns one line per key in OpenSSH format:

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIK... user@host
ssh-rsa AAAAB3NzaC1yc2EAAAA... id_rsa
```

DevTrace fetches each platform's `.keys` response for the contributor's handle, computes
the SHA-256 fingerprint of every public key, and compares the sets. A match is a match
--- the same `SHA256:...` value appearing in two different platforms' responses. No
platform involvement, no API key, no rate-limit dependency on a vendor's terms.

The result is cached weekly per platform, so the marginal cost of re-scoring a
contributor whose cross-VCS surface has not changed is a single cache lookup, not four
HTTP calls.

## Why fingerprint matches are categorically stronger

Three reasons, in order of importance.

**The signal is not declared, it is demonstrated.** A bio link is a string the user
typed. A fingerprint match required the user to upload the same public key on each
platform. To do that, they held the corresponding private key when they did it.

**Spoofing requires compromise, not registration.** To fake a GitHub--GitLab match, an
attacker would have to either obtain the target's private key --- equivalent to the
target reusing a compromised key --- or upload their own key to both accounts, which
means they already control both accounts, which is a different problem entirely. There
is no cheap trick.

**The check is reproducible by anyone.** The `.keys` endpoints are public. A reviewer
who wants to verify the DevTrace match by hand can run `curl github.com/USER.keys` and
`curl gitlab.com/USER.keys` and compare the SHA-256 fingerprints themselves. The signal
is not DevTrace's claim. DevTrace just consolidates it into one place.

## What a strong cross-VCS profile looks like

A typical long-tenured open source contributor --- someone who has been writing code in
public for a decade --- will often have keys on three or four forges. The kernel
mailing-list contributors, the longtime KDE developers, the people who maintain
infrastructure projects across multiple ecosystems. Their GitHub `.keys` matches at least
one entry on GitLab and frequently on Codeberg as well.

In the scorecard, this surfaces as a Cross-VCS section showing each platform with a
checkmark next to the matched fingerprint. Three matches is a strong signal. Two matches
is solid. One match is meaningful but circumstantial --- the contributor has at least
proven control of the same key on two systems, which is more than a declaration but
less than a cross-ecosystem footprint.

A contrasting profile shows zero matches. Either the contributor has no presence on
forges other than GitHub, has not uploaded a key to those forges, or rotated keys
recently enough that previous matches are gone. None of those is automatically
suspicious, but combined with a sparse profile, low repo diversity, and no community
footprint, the absence becomes part of the picture.

## What this signal does not prove

A cross-VCS match is not a real-name identity check. It does not tell you whether the
person behind the keys is who they claim to be in the social sense --- only that the
same person controls all the matched accounts. The Jia Tan account in the xz-utils
incident was a single coherent identity across the surface the attacker chose to
maintain. A cross-VCS match between Jia Tan on GitHub and a hypothetical Jia Tan on
GitLab would have been real, in the cryptographic sense. It would not have been correct,
in the social-engineering sense.

Three other limits are worth naming explicitly:

- **Bitbucket has no public `.keys` endpoint.** Atlassian gates SSH key access behind an
  authenticated API. DevTrace cannot include Bitbucket in the cross-VCS check today. If
  the endpoint ever becomes public, it goes in.
- **The handle is assumed to be the same.** DevTrace v1 fetches `.keys` for the same
  handle across platforms. A contributor who is `alice` on GitHub and `alice-h` on
  Codeberg will not show a match even if the underlying keys are identical. This is a
  known limitation that handle-mapping will address in a later release.
- **GPG fingerprint correlation is not yet included.** GitHub exposes signing keys at
  `github.com/USER.gpg`. The same approach --- fetch, SHA-256, compare --- applies. It
  is on the roadmap but not in the v1 cross-VCS surface.

## How DevTrace exposes the signal

Cross-VCS identity is a Pro-tier capability. Every Pro scorecard for an authenticated
contributor includes the section automatically when the relevant handles exist. The
matches are cached weekly per platform, the call budget is bounded, and the
unauthenticated `.keys` lookup does not depend on any platform's API quota.

For programmatic use, the same data is returned in the JSON response from
`/api/v1/score/{username}` on a Pro plan, in a `cross_vcs` field with one entry per
platform and the matched fingerprints listed alongside.

The reason to surface this at all is straightforward. "Verified identity" in open
source has been treated as a marketing concept for too long. The strongest signal that
does not require a human in the loop already exists, in the public infrastructure of
every major forge. DevTrace consolidates it into the scorecard so that a reviewer
reading a PR can see "matched on three forges" alongside the rest of the trust signal
--- without opening four tabs and computing fingerprints in a terminal.

## Try it

DevTrace is free to score any public GitHub contributor at
[devtrace.thingz.io](https://devtrace.thingz.io). The cross-VCS identity signal sits on
the Pro tier alongside the behavioral signals and synthetic-contributor flags --- but
the rest of the scorecard, including the 25+ trust signals, AI risk narrative, and
enrichment surface, is available on every authenticated plan.

If you are evaluating the projects those contributors maintain rather than just the
contributors themselves, [DevPulse](https://devpulse.thingz.io) tracks the project-level
signals that complement contributor trust. A project whose top maintainers verify across
multiple forges is a different proposition from one whose maintainers do not, and the
two views are most useful when read together: DevTrace tells you who, DevPulse tells you
how the project is holding up around them.
