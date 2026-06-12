# Caliptra Subsystem Attestation Architecture: Summary

## Purpose

This architecture defines how the Caliptra subsystem builds an end-to-end attestation story for platform and SoC inventory. It describes the trust chain, DPE context topology, AK derivation choices, and how DICE/DPE certificate evidence is combined with OCP EAT claims.

## Core Roles

| Component | Role |
| --- | --- |
| Caliptra Core | Hardware Root of Trust. Owns DICE/DPE state, protects AK material, verifies MCU RT, processes DPE commands, and signs the `COSE_Sign1` envelope requested by MCU RT. |
| MCU RT | DPE client and Attesting Environment for downstream SoC components after Caliptra Core measures and authorizes it. The SPDM responder in MCU RT assembles OCP EAT claims. |
| SoC TCB components | Security-relevant SoC components that may be represented as DPE contexts in Calitpra RT when protected measurement state or AK lineage is required. |
| SoC non-TCB components | Components whose inventory/compliance claims are managed by MCU RT and included as OCP EAT claims but do not participate in DPE/CDI derivation. |

## Evidence Model

The verifier receives two complementary forms of evidence:

1. **DICE/DPE certificate chain** proving lineage to the selected AK target node. Used as Identity attestation (DICE)
2. **OCP EAT claims** assembled by MCU RT and signed by Caliptra Core as a `COSE_Sign1` envelope. Used for Inventory attestation (SPDM)

TCB components on the selected AK lineage contribute to AK derivation and are not repeated as inventory claims. Other configured TCB contexts and non-TCB inventory claims can be reported as flat OCP EAT claims and appraised by policy.

## DPE Context Topology

The common DPE chain is:

```text
Root("RTMR")
 └─ CCIV("CCIV")
    └─ ROM_Stash_1…N
       └─ SoC_Manifest("SOCM")
          └─ MCU_RT("MCFW")
             └─ <configured SoC TCB context tree>
```

The chain above `MCU_RT` is common across integrations. Product-specific topology appears below `MCU_RT`, where SoC TCB contexts are created according to platform policy.

## AK Target Principle

The configured AK target determines what contributes to AK derivation:

| AK target | Effect |
| --- | --- |
| `MCU_RT` | AK covers platform boot chain through MCU RT. Downstream SoC TCB contexts may still be reported as EAT claims but do not contribute to this AK. |
| SoC TCB context | AK covers the path from root through MCU RT to that SoC TCB node. |
| CC branch leaf | AK covers the confidential-compute branch lineage. |

Descendant contexts below the AK target do not contribute to that AK unless the platform configures the AK target at that descendant.

## Inventory Attestation

Inventory attestation reports platform identity, SoC firmware/configuration state, and inventory claims for owner or fleet-management verifiers.

It includes:

- DICE/DPE lineage for the configured inventory AK target.
- TCB measurement records represented in Caliptra DPE state.
- TCB measurement claims assembled by MCU RT for configured contexts outside the selected AK lineage.
- Non-TCB inventory claims collected in the MCU realm and reported as flat OCP EAT claims.

## Non-TCB Claims

Non-TCB components are not represented as DPE contexts. Their claims are collected by MCU RT, stored through the Software PCR Storage path, and included directly in the flat OCP EAT claims set.

These claims are still attestation evidence: they require authorized collection, integrity protection before inclusion in EAT, and verifier appraisal rules.

## Update Model

| Context / claim class | Update model |
|---|---|
| Caliptra-owned and Caliptra-managed contexts, including Root, CCIV, ROM-stashed measurements, SoC manifest contexts, and MCU_RT | Updated by Caliptra-controlled flows using the backdoor mechanism. Because the MCU_RT context is updated through the backdoor path, the MCU_RT DPE context handle held by MCU RT is not rotated. |
| Downstream SoC TCB contexts | Created by MCU RT with `DeriveContext` during image loading and updated with `UpdateContextMeasurement` during component update. MCU RT updates stored DPE handles when DPE returns rotated parent or component handles. |
| SoC non-TCB claims | Updated by MCU RT in Software PCR Storage and reflected in OCP EAT inventory claims. |

For sequential SoC TCB chains, handle tracking is important because update operations can rotate parent and child handles.

## Confidential Compute Scenario

Confidential compute attestation is modeled as a branch in the DPE context tree. The platform selects a CC fork point, and the CC AK target is the leaf of the CC branch.

Example topology:

```text
Root("RTMR")
 └─ CCIV("CCIV")
    └─ ROM_Stash_1…N
       └─ SoC_Manifest("SOCM")
          └─ MCU_RT("MCFW")
             └─ CC_Fork_Point
                ├─ Inventory_Branch
                │  └─ SoC_TCB_A
                │     └─ SoC_TCB_B
                └─ CC_Branch
                   └─ CC_Runtime
                      └─ CC_Workload   <- CC AK target
```

Inventory/platform branches and CC branches can be updated independently below the fork point. Updates at or above the fork point affect both branches.

For example:

```text
Update SoC_TCB_B
  -> affects inventory/platform branch only
  -> does not change the CC AK lineage

Update CC_Workload
  -> affects CC branch only
  -> does not change inventory AK lineage

Update CC_Fork_Point or anything above it
  -> affects both inventory/platform and CC branch AK lineages
```

## Open Questions

1. How does MCU RT learn the platform-level CC fork point?
2. For CC attestation, is the DICE/DPE certificate chain sufficient, or is a CC EAT also required for nonce binding, verifier policy, or additional claims?
