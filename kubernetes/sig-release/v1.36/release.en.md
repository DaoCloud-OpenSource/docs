# Kubernetes v1.36 Preview: Key Changes to Watch Before GA

As of **March 18, 2026 (code freeze day)**, Kubernetes v1.36 is in the final phase before release.
This post summarizes the highest-signal changes for operators and platform teams, based on upstream release tracking, release-note drafts, and enhancement status.

> Status note: this is a pre-GA preview draft. The final upstream release is currently scheduled for **April 22, 2026**.

## Release Timeline (v1.36)

- Release cycle start: January 12, 2026
- Enhancements freeze: February 11, 2026 (AoE)
- Code/Test freeze: March 18, 2026 (AoE)
- Docs freeze: April 8, 2026 (AoE)
- Planned GA date: April 22, 2026

Reference: [SIG Release v1.36 README](https://github.com/kubernetes/sig-release/blob/master/releases/release-1.36/README.md)

## At a Glance

From the current tracked v1.36 enhancement set (milestone `v1.36`):

- 78 tracked items
- 18 stable/GA targets
- 25 beta targets
- 31 alpha targets

Most active areas include SIG Node, SIG Scheduling, SIG API Machinery, SIG Auth, and SIG Storage.

## High-Impact Changes To Watch

### 1) API Machinery and Control Plane Maturity

#### Mutating Admission Policies move to GA

`MutatingAdmissionPolicy` is one of the most visible graduation items for v1.36 and is now expected to be enabled by default in the current release-note drafts.

- KEP: [#3962](https://github.com/kubernetes/enhancements/issues/3962)

#### Kubernetes API protobuf cleanup continues

The removal of historical `gogo protobuf` dependency patterns is a large ecosystem-facing cleanup effort with compatibility implications for projects relying on internal serialization assumptions.

- KEP: [#5589](https://github.com/kubernetes/enhancements/issues/5589)

### 2) Storage and DRA Momentum

Storage and Dynamic Resource Allocation (DRA) continue to be a strong theme in 1.36.

#### Volume Group Snapshot targeting GA

A major data-protection and stateful workload capability to snapshot a set of volumes consistently.

- KEP: [#3476](https://github.com/kubernetes/enhancements/issues/3476)

#### Mutable CSI node allocatable targeting GA

Allows CSI drivers to update attach limits dynamically, reducing scheduling mismatch from static assumptions.

- KEP: [#4876](https://github.com/kubernetes/enhancements/issues/4876)

#### CSI token delivery via secrets field targeting GA

Adds an optional secure path for service-account token delivery to CSI drivers.

- KEP: [#5538](https://github.com/kubernetes/enhancements/issues/5538)

### 3) Node and Scheduling Evolution

#### Node log query targeting GA

Improves built-in operational debugging workflow directly from Kubernetes APIs.

- KEP: [#2258](https://github.com/kubernetes/enhancements/issues/2258)

#### User namespaces and node-side hardening efforts continue

The v1.36 set includes additional node/runtime work across resource managers, authorization hardening, and scheduling behavior.

- Example KEP: [#127](https://github.com/kubernetes/enhancements/issues/127)

### 4) Network and Validation Safety

#### IP/CIDR validation improvements heading to Beta

Upstream highlight discussions call out tighter canonical validation for IP/CIDR formats to reduce ambiguity and security risk.

- KEP: [#4858](https://github.com/kubernetes/enhancements/issues/4858)

## Upgrade and Compatibility Notes (Important)

Based on current changelog and release-note draft signals, operators should proactively validate the following before production upgrades:

1. Scheduler plugin interfaces: recent changes around `PreBind` parallelization and `PostFilterResult` require plugin implementers to review integration behavior.
2. kubeadm + flex-volume support removal: integrated flex-volume support in kubeadm has been removed; environments still relying on this path should plan migration or custom handling.
3. `gitRepo` volume plugin disablement: the `git-repo` plugin is disabled by default and cannot be re-enabled in the current release-note direction.
4. API/client-go behavior shifts: multiple API machinery and informer behavior improvements are targeting correctness and performance, but may expose assumptions in custom controllers.

## What Is Still Pending Before Final Release Blog

Because this is a pre-GA draft, re-check these items right before publication:

- Final release theme and logo assets
- Final `CHANGELOG-1.36.md` GA section
- Final release-note wording for breaking/urgent items
- Any late-cycle known-issue disclosures

## References

- [Kubernetes v1.36 release cycle overview](https://github.com/kubernetes/sig-release/blob/master/releases/release-1.36/README.md)
- [Kubernetes v1.36 release team](https://github.com/kubernetes/sig-release/blob/master/releases/release-1.36/release-team.md)
- [Kubernetes v1.36 tracking links](https://github.com/kubernetes/sig-release/blob/master/releases/release-1.36/links.md)
- [Kubernetes v1.36 milestone](https://github.com/kubernetes/kubernetes/milestone/69)
- [Kubernetes v1.36 changelog draft](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.36.md)
- [Kubernetes v1.36 release notes draft](https://github.com/kubernetes/sig-release/blob/master/releases/release-1.36/release-notes/release-notes-draft.md)
- [v1.36 release highlights discussion](https://github.com/kubernetes/sig-release/discussions/2958)
