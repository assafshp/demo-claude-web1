# ADR-014: Target-State Integration Architecture

**Status:** Accepted
**Date:** 2026-07-22
**Author:** Ronit Levy, Chief Enterprise Architect
**Reviewers:** Raj Mehta (Integration Lead), Alex Petrov (CISO), Jordan Kim (VP Applications)
**Supersedes:** ADR-006 (Single Integration Platform, 2023-11-30)

## Context

The June 2024 principle mandating that all core data flows pass through MuleSoft
succeeded beyond projection. Managed APIs grew from 61 at the end of 2023 to 390 by
September 2026, a sixfold increase. The platform became the authoritative path for
every integration between Salesforce, SAP S/4HANA, Workday, Snowflake and ServiceNow.

That success produced the failure mode of 8 May 2026. The API gateway reached the rate
limit of the tier purchased in 2023 and rejected 22% of inbound B2B order traffic in
China for four hours during the 618 festival preparation window. No capacity review had
been performed since the platform was selected.

Two structural problems are now visible. First, MuleSoft is a single point of failure
for the entire core-system estate; the DORA assessment of February 2025 had already
named it one of 23 critical providers without an exit plan. Second, the platform is
carrying two workload classes with incompatible characteristics: low-volume,
high-value system-of-record synchronisation, and high-volume, spiky external channel
traffic. Sizing for the second is what failed.

## Decision

We adopt a tiered integration topology that separates workload classes rather than
consolidating them further.

**Tier 1 — System-of-record integration.** Remains on MuleSoft. Covers
Salesforce to S/4HANA order and opportunity flows, Workday to S/4HANA employee and
payroll data, Oracle EBS to S/4HANA consolidated financial reporting, and all
ServiceNow CMDB asset feeds. These are low-volume, transactionally critical, and
benefit from the existing API catalogue and governance.

**Tier 2 — External channel ingress.** Moves off MuleSoft to a dedicated
API gateway with independent autoscaling, deployed per region. Covers B2B order intake,
partner portals and the e-commerce channels. Rationale: this traffic is spiky,
externally driven, and its failure mode must not propagate to Tier 1.

**Tier 3 — Bulk data movement.** Remains on Fivetran into Snowflake, with dbt
for transformation. Explicitly not an integration-platform concern.

## Consequences

Accepted costs. Two gateway technologies to operate instead of one, with an estimated
1.4 million dollars annually in additional licence and platform-engineering cost.
This reverses part of the consolidation rationale of ADR-006, and we state that
plainly: ADR-006 optimised for integration risk and did not price concentration risk.

Capacity governance becomes mandatory. Raj Mehta owns a quarterly capacity review for
both tiers, with headroom thresholds tied to the API catalogue. The absence of this
review is the direct cause of the May 2026 incident and its introduction is
non-negotiable.

Exit-plan progress. Splitting Tier 2 off MuleSoft reduces the blast radius of a
MuleSoft failure or vendor exit from the entire estate to system-of-record flows only.
This partially addresses the open DORA finding, though 22 other critical providers
still lack documented exit plans.

## Alternatives rejected

**Upgrade the MuleSoft tier and keep one platform.** Cheapest at roughly 600 thousand
dollars annually and operationally simplest. Rejected because it treats a capacity
symptom and leaves the single point of failure intact, which is the finding Alex Petrov
raised in the DORA assessment.

**Move everything to a cloud-native gateway.** Rejected on capability and people
grounds: the nine MuleSoft-certified engineers and the mature API governance were the
deciding factor in ADR-006 and remain valid for Tier 1. A full migration would discard
that while the Horizon programme is still mid-flight.

**Event-driven backbone (Kafka) for all integration.** Architecturally attractive and
rejected on sequencing, not merit. Revisit after Horizon Wave 3 completes in Q3 2027.
Introducing a third paradigm while the Americas ERP migration is in progress repeats
the Data Mesh mistake of December 2024: changing the operating model and the technology
at the same time, with no owner for the transition.
