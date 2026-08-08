---
rfc: "0033"
title: The content index
status: Proposed
authors: ["@Maximilian-Nesslauer"]
created: 2026-08-07
discussion: https://github.com/KSAModding/content-manager-design/pull/33
supersedes: []
superseded-by: []
---

# RFC 0033: The content index

## Summary

One canonical community index, stored as two GitHub repositories and consumed by clients as one snapshot artifact at a stable address.

- **An authored repository** holds the authored TOML documents defined by [RFC 0031](0031-content-metadata-format.md), one per listing, plus the pack documents. Humans write here, and a new listing whose ownership verifies automatically merges itself.
- **A generated repository** holds the stamped release files, one JSON document per release. A scheduled watcher writes here unattended. Releases of content the watcher does not watch enter by validated pull request.

A new release on a watched host needs no human.

Listing your own content is self-service through an automatically validated pull request.
Humans act on disputes, takedowns, delisting, and on the listings whose ownership cannot be verified automatically.

Everything runs on GitHub repositories and scheduled GitHub Actions, so nothing needs a server anyone has to keep paying for ([RFC 0025](0025-scope.md)).
The index is central in authority and decentralized in survivability: one arbiter for the id namespace, and a full mirror is `git clone`.

This RFC settles the discussion in [#27](https://github.com/KSAModding/content-manager-design/discussions/27).

## Motivation

RFC 0031 defines what the metadata says and deliberately leaves where it lives, who writes it, and how clients get it to this RFC.

A client needs one catalogue that is allowed to say what exists.

The game makes the mod id namespace global, because the folder name is the identity (`Mod.MakeUsing` overwrites any declared id, see [research/ksa-mod-loading.md](../research/ksa-mod-loading.md)), and RFC 0031 extends that namespace across all content types so every reference to an id stays type-free.

A global namespace needs exactly one arbiter for uniqueness.

Dependency resolution needs the same thing: without a shared catalogue, "what does this mod depend on" means downloading archives, and discovery becomes crawling.

The constraints are set by [RFC 0025](0025-scope.md): free infrastructure only, no hosting of content files, and the index format is our own definition.

## Guide-level explanation

### Listing your content

You write the authored file from RFC 0031 and run the listing tool, which opens a pull request against the authored repository for you.
Opening it by hand works the same way.

Checks validate the file: schema, id rules, the license field, the forums link, and an inspection of your latest release archive.

A further check verifies ownership, that the account opening the pull request controls the release host the listing points at.

When everything is green and ownership is verified, the pull request merges itself.

When ownership cannot be verified automatically, a steward has to look instead.

From then on you choose, per listing, how releases enter the index:

- With a `[releases]` section, the watcher stamps every release you publish, unattended. This is the recommended default.
- Without one, the same tool stamps a release synchronously at publish time and opens a pull request for it. This gives you the no-race property that the stamp can never predate your authored file, and it is the only path for content hosted anywhere the watcher does not watch.

### What a client does

A client fetches one URL and gets the whole index: every listed document, the release files that belong to listed entries, the index status data, and the game release list, merged into one snapshot artifact.

It polls with normal HTTP caching, so an unchanged index costs one 304 response.

Everything downstream, search, dependency resolution, compatibility evaluation, happens locally and offline, consistent with [RFC 0017](0017-game-version-ordering-and-compatibility.md) and RFC 0031.

### When something goes wrong

A broken release gets yanked or its bounds tightened by its author, through the amendment class RFC 0031 defines, and the index is where those amendments happen.
A broken pack version gets retracted through the index status data.
A listing that has to go away gets delisted by a steward and disappears from the snapshot with the next build, which the delisting commit itself triggers.
An id dispute is decided by a steward, with the required forums link as the tiebreaker for who was first.

## Reference-level explanation

### One canonical index

A **listing** is any entry the index arbitrates: a mod, a mod loader, or a pack.
All listings share one id namespace, evaluated case-insensitively with the authored casing preserved, per the id rules in RFC 0031.

The index is the authority for that namespace and for what is listed.

Possession is decided mechanically by the listing checks: first come, first served.

A steward may reassign an id only through the dispute path, with the required forums link as the tiebreaker for who was first.

Reassignment delists the previous holder's listing and its releases (they remain in history as with any delisting), and the dispossessed content may relist under a new id.

To a client, the reassigned id is new content, because the folder name on disk is the identity the game sees.

Multi-source stays a client capability: a client may consume additional sources next to the index (Borea already tags every entry with its source), but only the index arbitrates ids.

A listing is not blocked by dependencies that are not listed.
The CKAN rule that a mod cannot be indexed while a dependency is unlisted guarantees resolvable installs, but it also puts a human in front of every listing.
We deliberately do not copy it.
Clients detect an unlisted dependency by its absence from the snapshot and warn, per RFC 0031's client behavior.

### Two repositories

Proposed names: `content-index` (authored) and `content-index-releases` (generated).

Review requirements can be scoped per folder with CODEOWNERS, but the ability to push without a pull request cannot: a bypass actor on a branch bypasses that branch entirely.
Giving the watcher unattended write to a branch that also holds the authored documents would give it unattended write to those documents too.
Release commits also outnumber authored commits by an order of magnitude, and in one repository they bury the history of the half humans actually read.

The split has two costs: a listing's data lives in two places joined by id, which the snapshot artifact hides from clients, and the automation needs one identity with access to both repositories.
That identity is an org-owned GitHub App, whose installation token is minted per run: `contents: write` on the generated repository, `issues: write` and `actions: write` on the authored one.

### The authored repository

One TOML document per listing under `listings/<id>.toml` for the watched types (`mod`, `mod-loader`), one document per pack version under `packs/<id>/<version>.toml`, and the steward-owned `index-status.toml` described under moderation.
A validation check rejects a document whose `type` does not match its location.
The layout is internal business: the fetch contract below is what shields clients from it, so it can be reorganized without breaking anyone.

The branch protection this design needs, stated exactly because auto-merge dies on any deviation:

- Require a pull request before merging, with **required approving reviews set to zero**, because any higher value makes auto-merge unreachable for every pull request.
- Require review from code owners, with CODEOWNERS covering **only** the steward-owned paths: `index-status.toml`, `.github/` (the workflows and check definitions), and CODEOWNERS itself. `listings/` and `packs/` are deliberately not code-owned, so checks alone gate them.
- The validation checks below as required status checks. "Require branches to be up to date" stays off, since every merged listing would otherwise invalidate every other open pull request.
- Auto-merge enabled in the repository settings, and the Actions fork policy set to "require approval for first-time contributors who are new to GitHub", the least restrictive setting GitHub offers for public repositories.

### The listing flow

The checks run in two trust domains, and no job both handles author-supplied bytes and holds a write token:

**Unprivileged validation**, on `pull_request` with a read-only token and no secrets:

- schema validation against RFC 0031, including the id rules and reserved names,
- id collision, case-insensitive, against every id in the index, `listings/` and `packs/` alike, because the namespace is global across types. A new document under an existing `packs/<id>/` is a new version of that pack, not a collision,
- scope: an auto-merge candidate touches exactly one listing document or one pack document and nothing else, and anything wider waits for a steward,
- `license` present and a valid SPDX expression, `links.forums` present,
- archive inspection of the latest release: the archive downloads, its hash computes, and the install root is derivable or authored. A listing whose host has no release yet passes vacuously, and the first stamp performs the same inspection.

**Privileged decision**, in a separate workflow triggered by `workflow_run`, which reads only the unprivileged jobs' structured verdicts and never unpacks or executes pull request content: it runs the ownership check and, when everything passed and ownership verified, enables auto-merge on the pull request.
GitHub has no native merge-on-green: enabling auto-merge is an explicit API call, and this workflow is the only place that makes it.
A pull request whose ownership does not verify simply never gets auto-merge armed and waits for a steward, with all checks green.
Nothing needs a protection bypass.

Every check reports one of three outcomes: **pass**, **reject**, and **could-not-evaluate** for the cases where the check could not run to a verdict, a release host down, a rate limit, an archive momentarily unavailable.
Could-not-evaluate never auto-merges and never auto-rejects.
The watcher's sweep picks it up.

### Ownership

Ownership binds to the release authority: the host named in `[releases]`, or by `authority` when several are named.
When the listing has no `[releases]` section, `links.repository` is the basis instead.

For a GitHub authority, "controls" means write access, proven rather than looked up, because no GitHub API answers "does account X control repository Y" for a repository the caller has no access to, and the naive owner-name comparison passes forks and renamed repositories:

- The check reads a marker file at a well-known path in the target repository, `.github/ksa-content-index.toml`, carrying the listing id and the claiming account's login. Only someone with write access to that repository can have placed it, and the listing tool writes it automatically with the author's own token.
- A fork is rejected outright, since forks inherit files.
- As a fast path, a non-fork repository whose numeric owner id equals the pull request author's needs no marker.
- A redirect on the repository lookup means the repository moved. The listing is treated as stale and does not verify until the pointer or the marker is updated.

For a SpaceDock authority there is no comparable proof today, so those listings wait for a steward.
The unresolved question below tracks it.

Ownership is not checked once and forgotten:

- A maintainer handover is a pull request to the listing, verified against the new authority or reviewed by a steward.
- When the authority host stays unreachable or the marker stays invalid across consecutive ticks, the watcher keeps a single open issue per listing current instead of opening a new one per tick, and a listing that stays dead is eventually marked in the index status data.

### Packs

Packs have no release host, so nothing mechanical can verify a first claim: the first account whose pull request creates `packs/<id>/` owns the id, applied by a steward, and later versions are verified against that account automatically.
A check on `packs/` enforces what RFC 0031 declares: an existing pack document is never modified or deleted, a new version is a new file.
A broken pack version is retracted through the index status data, scoped to that version.
Clients treat a retracted pack version like a yanked release, per RFC 0031's client behavior.

### The generated repository

One JSON document per release under `releases/<id>/<version>.json`.
Three paths write here:

**The watcher** commits directly to the default branch, unattended.
The branch is governed by a ruleset requiring a pull request plus the amendment check, with the GitHub App the watcher runs as listed as bypass actor.
The bypass is scoped to that identity, and the watcher's own code enforces the same limit, it writes release files and nothing else.

**Release pull requests**, the path for listings without a `[releases]` section.
The listing tool stamps the release file and opens a pull request adding exactly one new file whose path and version do not exist yet.

The checks do not trust the submitted document: they download the archive from its `download.url`, recompute the hash, the sizes, and the install root, re-run the dependency merge against the authored file, and reject the submission when any stamped field disagrees with what the check computed.

Nobody hand-writes a release file, and this path is no exception: the tool writes it, and the checks re-derive it (see [research/prior-art-ckan.md](../research/prior-art-ckan.md) for what a hand-written release file did).

Ownership and auto-merge work exactly as in the listing flow.

**Amendment pull requests**, for the class RFC 0031 defines.

An amendment touches release files of exactly one listing, one or several, and a check enforces the invariant per file, mechanically: a release file can never become more permissive after publish.

Several files matter for the common case, a mod found to break at a given game build needs `game_max` added to every affected past release, and that is one pull request, not N.

Who may amend: the verified owner of the listing, through the same ownership check, or a steward.

Yanking is an amendment like any other.

The repository also maintains the game release list, updated by a scheduled job polling the master server, the same endpoint the game itself asks (`VersionInfo.GetServerVersionAsync`), the pattern `builds.json` has proven since 2026-07-02.
[RFC 0017](0017-game-version-ordering-and-compatibility.md) names such an external list as the fallback for periods a user's install does not cover and leaves "whether a manager consumes an external release list at all" open.

This RFC answers that question by shipping the list inside the snapshot, so a client needs no separate request, while `Content/Versions/` stays the local source.

The watcher uses the list to resolve authored month bounds to revisions at stamp time.
A bound naming a month that is not over yet cannot resolve to its last revision.
The watcher stamps it as open and re-resolves it once the month completes, which is a stamp correction by the watcher, not an amendment.

### The watcher

A scheduled GitHub Actions workflow in the generated repository, running as the org App.
The intended tick is every ten minutes, a tuning parameter that needs no RFC to change.
GitHub's floor for schedules is five minutes.

The scan rule is set membership, defined once: each tick asks the authority host for its releases and stamps every release that has no file under `releases/<id>/` yet, optionally bounded by a lookback window.

A release published out of version order, a patch for an older line tagged after a newer version exists, is therefore still stamped.

Per new release:

1. Download the archive from the authority host and compute the sha256.
2. For `mod` listings, read the code dependencies out of the archive's own `mod.toml` (`[[StarMap.ModDependencies]]`). An archive without a `mod.toml`, which is what a `mod-loader` archive looks like, contributes no code dependencies and is not an error.
3. Merge with the authored document per RFC 0031, resolve month bounds to revisions, derive the install root, and check the non-authority hosts for an archive with identical bytes to populate `download.mirrors`.
4. Stamp the release file and commit it. A version that does not parse, or an install root that cannot be derived or found authored, is rejected, and the watcher surfaces the error by keeping one issue per listing on the authored repository current.

`download.mirrors` is the one field the watcher may append to after publish, when the identical archive appears on a further host later.
It is watcher-only, append-only, and adds no permission the hash does not already gate, since any source whose bytes match `download.sha256` was already acceptable (RFC 0031).

A version is stamped exactly once.
When a host's tag for an already stamped version reappears with different bytes, the watcher rejects it and records it in the listing's issue on the authored repository, naming the version and both hashes, which is the flagging RFC 0031 defers to this RFC.

The watcher is resilient by construction, and the design keeps it so:

- **Its state is what is stamped in the repository**, never an event queue. A cancelled or dropped tick costs latency, not data: the next tick rescans. This is an argument from shape, not a measured property of a system that does not exist yet. What the [2026-08-06 GitHub Actions incident](https://stspg.io/rcz3fcm83sff) did show is the two halves' difference in kind: the scheduled `builds.json` job kept running while event-driven pull request checks in this repository lost their trigger with nothing to retry them.
- **Every tick is idempotent.** Re-running a tick stamps nothing twice, because "already stamped" is read from the repository itself.
- **Each tick also sweeps the event-driven half.** The authored repository's validation workflow additionally accepts a manual dispatch carrying the pull request number, validates the merge ref, and reports through a commit status under the required check's name. The sweep re-dispatches every open pull request whose head commit has no check suite or whose latest run ended in could-not-evaluate, and pings a steward for runs sitting in GitHub's waiting-for-approval state, which a dispatch cannot release.
- **Schedules are best-effort.** GitHub delays or drops scheduled runs under load, worst at the top of the hour, so crons sit off the hour boundary. GitHub also disables schedules in a repository with no activity for 60 days, which the watcher's own commits prevent in the generated repository. A scheduled job placed in the authored repository would not have that protection, so none is.

The tick has a request budget.
The App's installation rate limit is a multiple of the default token's, and the watcher polls each listing with a conditional request against a stored per-listing ETag, so unchanged listings cost no rate limit at all.
That ETag store is derived cache, not state, and losing it costs one expensive tick and nothing else.
Fan-out stays well under GitHub's concurrency limits.

### The fetch contract

Clients get one index base URL with a fixed snapshot path, serving a single snapshot artifact, and never read the repositories directly.

- The snapshot merges every listed authored document, the release files of listed entries, the index status data, and the game release list into one JSON document carrying its own format version.
- A delisted listing is present only as a tombstone, its id and its status and nothing else, so a client can tell "removed" from "never listed". A disputed listing ships whole, with its status, so the client can warn.
- The snapshot is rebuilt on every change to either repository, with a scheduled run as backstop, and served with a strong `ETag` honoring `If-None-Match`, so an unchanged index costs one 304.
- Published metadata stays evaluable offline: the snapshot is sufficient for search, resolution, and compatibility without any further request.
- Per-file raw fetch stays possible as a secondary path for tooling, but it is explicitly not the contract, so storage can be reorganized without breaking any client.

The address must actually deliver those semantics, which constrains the hosting: GitHub Pages and raw.githubusercontent.com serve strong ETags and 304s.
A release-asset URL answers with a redirect per poll, and a codeload archive URL is not byte-stable across GitHub's toolchain and is excluded, which also retires the tarball address the CKAN research document points at.
The CDN in front of the chosen address has a staleness floor of a few minutes that nothing can purge, and a cache miss re-downloads the whole snapshot, since there is no delta path.
Both bounds appear below where latency is claimed.
The concrete address is an unresolved question, bound by these constraints.

### Moderation, takedown, and index status

The index has its own voice about a listing, distinct from the author's `status` field, exactly as RFC 0031 requires: the author cannot write it.
It lives in the authored repository as `index-status.toml`, owned by the stewards via CODEOWNERS, one entry per affected id:

- `delisted`: the listing and its releases leave the snapshot, a tombstone remains. Installs stop being offered, and nothing is deleted from anyone's machine.
- `disputed`: the listing stays in the snapshot and clients show a warning, for id disputes and contested ownership while they are being resolved.
- `retracted`, scoped to a pack version: the pack-version equivalent of a yank, since a pack has no release file to amend.

So that one steward can act alone at any hour, the stewards team is a bypass actor for exactly this path, and a change to it triggers the snapshot rebuild directly.
The honest latency of a delisting is therefore one snapshot build plus the address's cache floor plus the client's poll interval, minutes, not "instant", and this RFC does not pretend otherwise.
Mirrors and raw-path readers are outside the index's enforcement entirely.
For content that must disappear from hosting, the host's own takedown process is the lever, since the index never hosts files.
Scrubbing repository history is a heavier, separate operation reserved for when it is legally required.

What the index owes is the documented path: a takedown and a dispute issue form on the authored repository, and the delisting mechanism above.
The required forums link doubles as moderation infrastructure, as raised in the metadata discussions: it ties a listing to an Ahwoo account, serves as the tiebreaker in id disputes, and a taken-down thread or banned author is a tripwire that flags the listing for review.

Humans act on disputes, takedowns, delisting, and on the listings whose ownership cannot be verified automatically, today the SpaceDock-only listings and first pack claims.
Everything else in this RFC runs without one.

### Resilience and survivability

Central in authority, decentralized in survivability.

Everything above is plain git plus static files on free infrastructure.
Mirroring the entire index is `git clone` on two repositories, anyone can do it at any time without asking, and a client's index base URL is configuration, so the community can re-root the index if its host ever has to change.

## Drawbacks

- A listing's data lives in two repositories joined by id. The snapshot hides that from clients but not from tooling and stewards.
- The automation stands on an org-owned GitHub App. That identity is part of the system's bus factor, next to GitHub itself as the single point of dependency for storage, automation, and ownership proofs. The git-mirrorability limits the damage to availability, but the automation is Actions-shaped and would need porting.
- Releases become installable typically minutes after tagging, longer when GitHub delays scheduled runs, and a cache-missing client re-downloads the whole snapshot.
- A residual steward queue exists by design: SpaceDock-only listings, first pack claims, listings whose ownership fails to verify, and the one-click release of a brand-new account's first pull request.
- Auto-merge on green means a schema-valid malicious listing can land without human eyes, and moderation is reactive, not preventive. That is deliberate: pre-screening every listing is the review queue this design exists to avoid, and the index never hosts the files themselves.

## Alternatives

**A hosted platform with accounts and an upload API**, the Modrinth and CurseForge shape.
Their "declare at publish time" model rides on an upload step the author performs against a server with accounts.
We have no upload step, no accounts, and no server, and RFC 0025 rules out infrastructure someone has to keep paying for.
The listing tool plus the pull request paths reproduce the property that matters, synchronous validated publishing, on free infrastructure, which is the resolution reached with @MrJeranimo in the RFC 0031 thread: both paths exist per listing, and the watcher is the recommended default.

**CKAN's hosted pipeline**: webhook and scheduler driven services on AWS (see [research/prior-art-ckan.md](../research/prior-art-ckan.md)), which can index within seconds of a release on hosts that fire webhooks.
Rejected for cost and operations: it is exactly the always-on paid service RFC 0025 forbids, and the only benefit over scheduled polling is latency we can afford to lose.

**One repository instead of two.**
Rejected in [#27](https://github.com/KSAModding/content-manager-design/discussions/27): the ability to push without a pull request is granted per branch and per actor, never per path, so the watcher's unattended write cannot be contained to the generated half, and release commits bury the authored history.

**Full decentralization, no canonical index.**
Rejected: without a shared catalogue there is no uniqueness arbiter for a namespace the game itself makes global for mods, no offline resolution, and discovery becomes crawling.
The survivability motivation behind decentralization is answered by mirrorability instead.

**Per-file fetch as the contract.**
Rejected: it welds every client to the storage layout and makes a consistent read of the index a many-request operation.
The snapshot keeps clients offline-first and the layout reorganizable.

**Pre-publication human review of listings.**
Rejected on @averageksp's account of working the CKAN indexing queue ([#27](https://github.com/KSAModding/content-manager-design/discussions/27)): the unlisted-dependency rule puts a human in front of every listing, and that queue is worked daily by very few people.
Structurally, a review queue makes listing latency a function of reviewer availability, which is the failure mode this project exists to avoid.
Validation is automated at publish time instead, with humans reserved for the cases automation cannot judge.

## Prior art

**CKAN and NetKAN** are the closest prior art in the half that matters here: an index that is static files in git rather than an API, though its automation runs on paid infrastructure, which is the half rejected above.
The KSA fork of that stack is the field study this design draws on throughout (see [research/prior-art-ckan.md](../research/prior-art-ckan.md)): the authored-generated split, publish-time validation in front of the author, and the `builds.json` job that has kept itself current at zero cost since 2026-07-02.
Its structural lessons are also why this design self-hosts its automation and gates nothing on a single reviewer.

**Modrinth and CurseForge** demonstrate author-declared publish-time metadata, adopted here as the release pull request path, with the hosted-platform half rejected above.

## Unresolved questions

- **SpaceDock ownership verification.** GitHub ownership is provable with a marker, while SpaceDock offers no comparable proof from a pull request.
- **The concrete snapshot address**, bound by the fetch contract's constraints (strong ETag, If-None-Match, no per-poll redirect): GitHub Pages and raw.githubusercontent.com qualify today. This also means the fetch URL depends on the repository names unless a redirect fronts it.
- **The takedown and dispute policy text**: response expectations and escalation, to be written as a plain document in the authored repository rather than decided here.
- **Repository names**: `content-index` and `content-index-releases` are proposals. The fetch URL is the one thing that depends on them.
- **The vehicle and save content types**: their listing documents follow RFC 0025 and a future RFC. The layout above gains a folder per type when they land.

## Future possibilities

- **Aggregated trust signals**: download counts from GitHub and SpaceDock collected by a scheduled job into a static file, and a curated featured list in the index. Per-user state such as favorites stays out, since it needs a server.
- **Release signing**: if RFC 0031's future signature field lands, the index is where keys would be published and rotated.
