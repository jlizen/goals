# Registry security response: hold, withdraw, publish-time checks


| Metadata             |                                    |
| :--                  | :--                                |
| Contact              | @jlizen                            |
| Status               | Proposed                           |
| Zulip channel        | N/A                                |
| [crates-io] champion | ??                                 |
| [cargo] champion     | ??                                 |
| [infra] champion     | ??                                 |
| [docs-rs] champion   | ??                                 |

## Summary

We want an auditable way for crates.io administrators to **hold** (reversibly "freeze" releases for investigation, 
preventing normal fetching of bytes) and **withdraw** (remove release bytes, leaving behind a tombstone). This is necessary to 
quickly and temporarily "freeze" a situation during a security investigation. It also lets us set up
auditable mechanisms to automatically hold supply chain attacks before they are released to the mirror.

We will use this capability in two ways:
1. manual admin action, via API or console, similar to current "admin delete" workflows
2. automated publish-time checks that, upon serious suspicious signals, place releases on hold and into a "manual review" queue

Our focus here is on building security primitives, with some simple detections wired up. Further detections and more sophisticated usage of these primitives will require separate Project Goals and/or RFCs.

## Motivation

### The status quo

crates.io today has three states:
- fully public
- public but "yanked" (Flagged as yanked in the index, but still distributed and live on the crates.io CDN, thus installable via Cargo.lock coordinates, but don't resolve as valid crates otherwise)
- deleted (bytes inaccessible, redacted from registry index, admin notifications are manual)

This makes life difficult from an incident response point of view. In a recent security incident, a stolen token published malware across an account of ~250 crates with hundreds of automated versions. The responder had to "wade through LLM spam of hundreds of versions" and chain together a script with ~70 yank & delete commands. In fact, one deletion failed and the related version lasted for a few extra days. According to the operator: "I kinda wish we had an intermediate step here. Deletion is only semi-reversible, but I would like to publicly nuke the account while we investigate."
 
Meanwhile, we recently saw a [successful supply chain attack on the arrayref crate](https://blog.rust-lang.org/2026/08/20/supply-chain-attack-on-arrayref/), which is present in ~75% of Rust environments and has ~250 million downloads. 
Among other attack elements, a malicious, namesquatting crate was published to crates.io, and then a compromised 
credential cut `arrayref` over to depending on it. There are a number of deterministic signals here that are clearly suspicious: a popular crate adding a new build dependency, a popular crate taking a dependency on a typosquat, a crate bearing base64 encoded URLs in its build script and other obvious malware signs, and so on.

In fact, for the `arrayref` incident, several Rust Foundation security systems did in fact trigger (for instance, the namesquatting detection). But, no automated action is taken by default, and signal quality is not high enough to page ourselves on those signals that we did detect. Instead, we manually pulled the malicious crate 86 minutes after publication, in response to an external vulnerability report from a security researcher.

We [recently stabilized min-publish-age in cargo](https://github.com/rust-lang/cargo/pull/17335), which allows setting a "cooldown" to give time for security reports and admin action before clients uptake malicious releases.
I expect that we will soon set a default min-publish-age for cargo. This will be useful and complementary change. 
Most reasonable default cooldowns would be longer than 86 minutes, meaning the arrayref attack would have much more limited impact. 

However, this still produces a "firedrill" for security responders where our release process fails open in case of a 
delayed response. This is a concerning operational posture given that we expect the supply chain attacks to grow both (for instance the recent [spearphishing attacks targeting RustLang members](https://blog.rust-lang.org/2026/09/17/targeted-attacks/)). It also relies on client-side configuration that does not extend to other build tools (example: Yocto), or tools that override the default cargo configuration.

### What we propose to do about it

We expect four phases of work. Each will have a design discussion via RFC or team issue, followed by implementation.

- [RFC 1](https://github.com/jlizen/rfcs/pull/1): **A registry quarantined/withdrawn state, matching Cargo behavior**: We can
start by adding Cargo/registry spec support for administrative quarantines and withdrawals. This lets us model
unreachable bytes at the build tool level. Any registry can signal that certain releases are quarantined, and optionaly
serve the withheld bytes for researcher use.
- crates.io issue/PR: **crates.io support for quarantines and withdrawals** crates.io will need distributed systems
additions to support quarantined bytes (CDN cache invalidations, etc), and we also will need new APIs to use during
security incidents. While we are at it, we can add some other nicer bulk admin APIs to reduce the amount of database
queries and elevated access needed during incidents.
- RFC 2: **A registry unreleased state, matching Cargo behavior**: Next comes general support for registries flagging
new releases "unreleased" and thus unavailable. This will involve changes to publish workflows to allow fetching
unreleased bytes as part of multi-crate release trains. It also will include forensic build support.
- RFC 3: **crates.io detection systems / automated holds / manual review queue / related governance**: Lastly, we
want to build the platform that lets us plug supply chain attack systems into crates.io's publish handling, and
pre-emptively freeze new releases for manual review. This is partly a technical problem, but even more a policy
and governance question. See "FAQ: Does this add work for maintainers?"

## Team asks

@jlizen plans to do most of the implementation work for this across all relevant systems, possibly delegating some to
contractors or Rust Foundation teammates. The team asks are for feedback on approach and reviews.

We expect to have funding to support reviews.

| Team | Support level | Notes |
|------|---------------|-------|
| [cargo] | Medium | Review and approve RFC 1 (Cargo/registry support for quarantine); review implementation of RFC 1; Review and approve RFC 2 (Unreleased state + Cargo publish support); review implementation of RFC 2 |
| [crates-io] | Medium | Co-review and approve RFC 1 (Cargo/registry support for quarantine); review and approve PR issue on manual admin APIs, byte management, authorization; review and approve RFC 2 (unreleased state); review and approve RFC 3 (publish-time detection systems); review implementation of RFC 3 |
| [docs-rs] | Medium | Review and approve RFC 1 and RFC 2 with regard to changes to docs.rs build conditions and release state tracking; review the implementation of RFC 1 and 2|
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

And then, we are separating the Cargo/registry spec baseline support for publish-time holds (RFC 3) separately for
the policies around detections that actually apply such holds (RFC 4).

### What are we leaving out?

The biggest thing is revocation of already-cached copies of crates. Verifiable mirroring will cover a lot of this out
of the box since it can invalidate stale merkle subtrees (ie index file caches). We also erred on the side of simple UX in a few other places. Details are in the RFCs.

## Funding

| Purpose | Cost | Funded | Sponsor(s) |
|---------|------|--------|------------|
| Reviews + design support + champion cycles | $10,000 | No | |
