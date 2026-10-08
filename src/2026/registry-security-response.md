# Registry security response: hold, withdraw, publish-time checks


| Metadata             |                                    |
| :--                  | :--                                |
| Contact              | @jlizen                            |
| Status               | Proposed                           |
| Zulip channel        | N/A                                |
| [crates-io] champion | @Turbo87                           |
| [cargo] champion     | @eh2406                            |
| [docs-rs] champion   | @syphar                            |
| [infra] champion     | @ubiratansoares                    |

## Summary

Registry administrators see supply chain attacks increasing in both in volume and sophistication.
We need to make sure we have the right security primitives to respond to new threats,
and that we have the confidence to use them when uncertain. We also need to raise the floor
of our front-line defenses so that our operators can focus on the most subtle attacks.

Today, we have a tight crates.io security response team, that reacts quickly, but our
mitigations are mostly manual and frequently destructive to the point of delaying response.
From recent attacks, we see tools that are missing from our toolkit which we want for the future.

An illustrative example: 
> In a recent security incident, a stolen token published malware across an account of ~250 crates with hundreds of automated versions. The responder had to "wade through LLM spam of hundreds of versions" and chain together a script with ~70 yank & delete commands. In fact, one deletion failed and the related version lasted for a few extra days. According to the operator: "I kinda wish we had an intermediate step here. Deletion is only semi-reversible, but I would like to publicly nuke the account while we investigate."

This goal builds some of those tools, in three phases:
1. Add registry support for "freezing" extremely suspicious packages for manual investigation, and holding their bytes
2. Gather shadow-mode data on supply chain detection systems to validate strategies
3. Build systems that detect likely attacks prior to publish and freeze them for human reviews

## Motivation

### The status quo

#### The big picture

Most language ecosystems have recently experienced supply chain attacks that compromised significant infrastructure.
Attacks are evolving in sophistication to include two stage attack payloads, hiding primary attacks underneath
poisoned dependencies, and manipulation of the resolver to increase delivery ([arrayref August 2026](https://blog.rust-lang.org/2026/08/20/supply-chain-attack-on-arrayref/)).
We also see new attacks such as targeting high-impact individuals [via spearphishing](https://blog.rust-lang.org/2026/09/17/targeted-attacks/).
Other projects see similar attacks ([such as Django in October 2026](https://frankwiles.com/posts/i-got-targeted/)).

Our current security responses are fairly tight, and we have largely avoided broad impact.
(`arrayref` was the worst attack we know of to date, and it was live for ~90 minutes with little evidence of further 
spread via compromised environments). 

#### Our current tools

We have quite a few detection systems built already, running offline. In fact, for the `arrayref` incident, several Rust 
Foundation security systems did in fact trigger (for instance, the namesquatting detection). They largely operate based
on scheduled jobs. Some fire notifications, but many are false-positive-prone, so they must be reviewed manually.

We are also building client-side hardening via `min-publish-age`. It adds Cargo-side cooldowns for extra scrutiny.
This could become set by default and relieve some of the urgency of security responses, at least for some swathe of our
consumers. Though, it still leaves registry administrators in a poisition of, "press this button in time or else there is an
incident", which is still a psychologically stressful operator role.

For operator responses, crates.io and other registries have three lifecycle states:
- public and available
- public but "yanked" (Flagged as yanked in the index, but still distributed and live on the crates.io CDN. Installable,
but Cargo and some build tools will avoid resolving yanked cordinates. Reversible.)
- deleted (Bytes inaccessible, redacted from registry index, admin notifications are manual. Semi-reversible but partially destructive.)

A gap in our existing mitigation (deletion) is that malicious bytes persisent in local Cargo caches after install.
This means that deleted crates are still buildable locally until the cache expires or is revoked. Ongoing [Verifiable Mirroring] work will address this gap without action by this goal, because it includes cheap verification of freshness of
index data (via merkle subtree anlysis).

#### Peer approaches

One thing that we are missing, that our peers in PyPI have built, is [a quarantine state](https://blog.pypi.org/posts/2024-12-30-quarantine/)
that explicitly *freezes* bytes and stops serving them, even if they are in lockfiles. This is the primitive we were missing
in the Summary's quoted incident.

PyPI maintainers are also [discussing automated detection + hold systems](https://github.com/python/peps/pull/5070). 
[npm has similar systems](https://github.com/orgs/community/discussions/203413), though has faced criticism on maintainer 
impact via its current implementation.

Maven Central has similar systems downstream of the registry via a paid product, [Firewall](https://help.sonatype.com/en/firewall-quarantine.html).


### What we propose to do about it

We expect three phases of work. Each will have one or more design discussions via RFC or team repo, followed by implementation.

Support for manual quarantines:
 - [RFC 1](https://github.com/jlizen/rfcs/pull/1): A registry quarantined/withdrawn state, matching Cargo behavior
 - Crates.io issue/PR: crates.io support for quarantines and withdrawals

Detection system experiments:
- Run existing detection systems against the crates.io event feed
- Build a couple new detection systems aimed at very-high-confidence checks that usually require human review
- Analyze the results of these experiments, including if they flagged on future supply chain attacks, as well as on
syntheic attack traffic, and prepare recommendations

Publish-time mitigation systems:
- RFC 2: A registry unreleased state, matching Cargo behavior. This includes support publishing against unreleased crates
to avoid breaking release train workflows.
- RFC 3: crates.io policies and practices to enable mitigation, related governance
- Crates.io issue/PR: crates.io support for publish-time scan and hold, manual review queue


## Team asks

@jlizen plans to do most of the implementation work for this across all relevant systems, possibly delegating some to
contractors or Rust Foundation teammates. The team asks are for feedback on approach and reviews.

We expect to have funding to support reviews.

| Team | Support level | Notes |
|------|---------------|-------|
| [cargo] | Medium | Review and approve RFC 1 (Cargo/registry support for quarantine); review implementation of RFC 1; Review and approve RFC 2 (Unreleased state + Cargo publish support); review implementation of RFC 2 |
| [crates-io] | Medium | Co-review and approve RFC 1 (Cargo/registry support for quarantine); review and approve PR issue on manual admin APIs, byte management, authorization; review and approve RFC 2 (unreleased state); review and approve RFC 3 (publish-time detection systems); review implementation of RFC 3 |
| [docs-rs] | Small | Review and approve RFC 1 and RFC 2 with regard to changes to docs.rs build conditions and release state tracking; review the implementation of RFC 1 and 2|
| [infra] | Small | Advisory consult on crates.io implementation of quarantine APIs and RFC 3 (publish-time scans) |

Beyond Rust Project teams, we will also want to consult with the Rust Foundation, particularly its security team. We
also will want to consult with outside build tools (Bazel, Buck2, Yocto). Our designs should maintain security boundaries
without external build tool changes, but they could make UX improvements if they were aware of our new index states.


## Frequently asked questions

### What's the point of holding already-released malware?

Users continuously install software in CI and otherwise. Even if a version has been released, there is benefit in
holding it to prevent subsequent installs. Lowering the threshold for taking admin action via a softer mitigation will
allow faster responses. Even better, we're working towards holding software before it is even released in the first place.

### Does this add work for maintainers?

For quarantine support is strictly a softening of the current mitigation available (ie, delete). It's also a quality
of life improvement for incident responders via bulk actions.

For publish-time detections and automated holds, there is a potential for maintainer impact. The RFC will go more in 
depth around the risks and controls for this. In general, the guiding principle will be, run in shadow mode for a 
while, make sure we have acceptable false-positive rates, focus on serious signals that are obviously suspicious, and 
only then promote a detection to acting. This protects  both maintainers from toil, and the crates.io and security 
teams reviewing the queue from overload. Similarly we will need to specify a SLA, an appeal mechanism, and other nuts 
and bolts.

I'm confident that we can find a not-perfect solution that everybody is comfortable with, even if it doesn't cover
all possible threat that we want to handle. We can keep iterating once we land a base set of systems.

### Who decides what gets held? What is the trust model?

This concern should be decided at the registry level. For the manual quarantine, we can use the existing
criteria that we use for admin deletion. We also can reuse the [existing malicious crate channels](https://blog.rust-lang.org/2026/02/13/crates.io-malicious-crate-update/) (along with new tombstones in the index files).

For the automatic hold we need to hash this out still. I imagine starting narrow, with deterministic checks,
and shadow mode to judge impact, will be a good place to start. We will need to balance Project values around
shared decisionmaking and transparency with the cat and mouse of detection efficacy. I suspect we will ultimately
land on a model where a small delegated group, inside a security-related project team, is responsible for approving
new detections, but with publicly agreed-upon criteria for such decisions that include data, values, etc.

### Why split this up?

We'd prefer each individual RFC to be relatively small to keep it manageable to review and find consensus. Specifically,
we are designing the quarantine registry spec and Cargo behaviors (RFC 1), separately from the actual distributed systems work that crates.io will implement on top of it (crates.io issue/PR).

And then, we are separating the Cargo/registry spec baseline support for publish-time holds (RFC 3) from
the policies around detections that actually apply such holds (RFC 4), and the crates.io-side implementation of those
policies.

### What are we leaving out?

We aren't trying to build a whole lot of detection systems, or the best detection systems. We mostly want to have
at least one detection, however thin, that is reliable enough to turn on. It's easy to discuss further systems once we have 
the base platforms.

We also are not going deep into crowdsourcing reports. Right now, we have a relatively crude process involving email
intake. One could imagine crowdsourcing flags [like PyPI has discussed](https://blog.pypi.org/posts/2024-12-30-quarantine/#future-improvement-automation),
for instance via cargo-vet. But, that deserves its own discussion and Project Goal. This one is already fairly large :)

## Funding

Funding will go to team reviewers and champions. Excess funds
will be contributed to the Rust Foundation Maintainer Fund to
support ongoing maintenance of related features.

| Purpose | Cost | Funded | Sponsor(s) |
|---------|------|--------|------------|
| Reviews + design support + champion cycles | $10,000 | No | |

