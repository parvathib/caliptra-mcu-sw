# Subsystem Attestation Architecture Review Categories

Purpose: group the open PR/RFC questions into a small set of discussion categories so the working group can review the architecture questions systematically.

---

## Question Categories

| Category | Count | What reviewers are asking |
| --- | ---: | --- |
| Measurement scope and policy classification | 5 | What “SoC TCB” / “SoC non-TCB” mean, what must be reported, whether inventory means infrastructure attestation, and how trademark requirements affect scope. |
| Static configuration and identifiers | 4 | How `fw_id` is used, uniqueness, static config authentication, SoC/Auth Manifest vs Attestation Manifest, and whether MCU RT loading paths can be bypassed. |
| Hardware/Software Target Environments measured by MCU | 1 | How hardware and software target environments measured by MCU should be identified and represented in evidence beyond firmware `fw_id` entries. |
| Measurement storage and provenance model | 4 | Why DPE vs Software PCR, whether MCU SRAM/Software PCR is mandated, RTS/transitive trust concerns, and why DPE-backed state may exist outside AK lineage. |
| EAT retrieval and transport interfaces | 3 | Relationship between OCP EAT and SPDM, how signed EAT is retrieved, and requester boundaries: BMC/pRoT, DOE, MCU mailbox. |
| Hitless update and measurement history | 2 | Current/journey appraisal, measurement logs, ROM-stashed immutable measurements. |
| DICE/DPE lineage and claim mapping | 3 | Which TCB claims are in DICE/DPE chain vs OCP EAT, how SVN values are reported, and DPE context lifecycle terminology. |
| Confidential compute / scenarios | 2 | Whether CC is in scope, and whether architecture should support non-CC attestation scenarios. |

---

## Category 1: Measurement Scope and Policy Classification

**Core question:** What is the reporting scope, and what do “SoC TCB” and “SoC non-TCB” mean?

**Explanation:**

- “SoC TCB” and “SoC non-TCB” are Caliptra subsystem policy classifications for downstream SoC components.
- The classification is chosen by integrator policy and verifier expectations.
- A SoC TCB entry is represented as DPE-backed TCI state.
- A SoC non-TCB entry is represented as a structured measurement record in protected MCU SRAM.
- AK-lineage contribution is separate from this classification.
- SoC TCB entries outside the selected AK lineage and SoC non-TCB entries are reported as OCP EAT claims.

**Discussion goal:** agree on policy terminology and avoid implying a universal TCB boundary.

---

## Category 2: Static Configuration and Identifiers

**Core question:** What configuration controls authorization, loading, and attestation routing?

**Explanation:**

- SoC/Auth Manifest metadata is set in Caliptra Runtime and drives authorization and `GET_IMAGE_INFO(fw_id)` metadata lookup.
- Attestation Manifest is embedded in the MCU Runtime image and authenticated as part of MCU Runtime verification.
- `SOC_IMAGE_LOAD_LIST` is embedded in MCU Runtime and provides the ordered `fw_id` list for MCU-managed image loading.
- In this context, `fw_id` identifies a firmware target environment and is the join key across these artifacts.
- `component_id` is loader/package metadata, not the attestation claim identifier.
- Hitless reuse requires policy/topology compatibility.

**Example: same `fw_id` ties the three artifacts together**

| SoC/Auth Manifest metadata<br>(Caliptra Runtime) | Attestation Manifest<br>(MCU Runtime image) | `SOC_IMAGE_LOAD_LIST`<br>(MCU Runtime image) |
| --- | --- | --- |
| `fw_id = 0x1000`<br>`component_id = 7`<br>`digest = H(FW_A)`<br>`load/staging addr = ...` | `fw_id = 0x1000`<br>`SOC_TCB_DPE = true`<br>`AK_TARGET = false` | `0x1000` |
| `fw_id = 0x1001`<br>`component_id = 8`<br>`digest = H(FW_B)`<br>`load/staging addr = ...` | `fw_id = 0x1001`<br>`SOC_TCB_DPE = true`<br>`AK_TARGET = false` | `0x1001` |
| `fw_id = 0x1002`<br>`component_id = 9`<br>`digest = H(FW_C)`<br>`load/staging addr = ...` | `fw_id = 0x1002`<br>`SOC_TCB_DPE = false`<br>`AK_TARGET = false` | `0x1002` |
| `fw_id = 0x1003`<br>`component_id = 10`<br>`digest = H(FW_D)`<br>`load/staging addr = ...` | `fw_id = 0x1003`<br>`SOC_TCB_DPE = false`<br>`AK_TARGET = false` | `0x1003` |

**How to read this example**

- Caliptra Runtime uses SoC/Auth Manifest metadata for authorization and `GET_IMAGE_INFO(fw_id)`.
- MCU Runtime uses `SOC_IMAGE_LOAD_LIST` order for image loading and DPE topology.
- Measurement API uses the Attestation Manifest to route each `fw_id` to DPE-backed or Software-PCR-backed measurement state.
- `component_id` locates the image payload; `fw_id` identifies the attested component and joins the artifacts.

**High-level state after loading**

```text
SOC_IMAGE_LOAD_LIST stages:
  1. 0x1000 (TCB, not AK)
  2. 0x1001 (TCB, not AK)
  3. 0x1002 (non-TCB)
  4. 0x1003 (non-TCB)

DPE-backed context tree:
Root("RTMR")
 └─ CCIV("CCIV")
    └─ ROM_Stash_1..N
       └─ SoC_Manifest_Vendor("SOMV")
          └─ SoC_Manifest_Owner("SOMO")
             └─ MCU_RT("MCFW")  [AK target]
                └─ 0x1000
                   └─ 0x1001

Software PCR Storage:
 ├─ record(0x1002)
 └─ record(0x1003)
```

The two TCB entries are represented in DPE-backed state, but neither is selected as the AK target. They are therefore reported as OCP EAT claims. The two non-TCB entries are recorded in Software PCR Storage and also reported as OCP EAT claims.

**Discussion goal:** align on which artifact owns which decision and where consistency is enforced.

---

## Category 3: Hardware/Software Target Environments Measured by MCU

**Core question:** How do we represent target environments that are not MCU-managed firmware images?

**Explanation:**

- `fw_id` currently covers MCU-managed firmware target environments.
- Hardware/software target environments measured by MCU need explicit identifiers, measurement sources, and claim schema.
- They may be DPE-backed or MCU-managed based on platform policy.
- Open item: add integration guidance for hardware/software target environments measured by MCU.

**Discussion goal:** define how hardware/software target environments measured by MCU are identified and represented without overloading firmware `fw_id`.

---

## Category 4: Measurement Storage and Provenance Model

**Core question:** Why use DPE-backed state for some measurements and protected MCU-managed records for others?

**Explanation:**

- The choice is about measurement-state provenance and verifier expectations, not only DPE context budget.
- DPE-backed TCI state provides Caliptra/DPE-backed provenance.
- A DPE-backed component does not have to be on the AK lineage.
- DPE-backed entries outside the selected AK lineage can still be reported as OCP EAT claims.
- Software PCR Storage is the current MCU implementation for structured MCU-managed measurement records.
- It is not meant to imply all non-DPE evidence claims are raw PCR values.

**Discussion goal:** clarify when DPE provenance is required versus when protected MCU-managed state is sufficient.

---

## Category 5: EAT Retrieval and Transport Interfaces

**Core question:** Why are OCP EAT and SPDM both involved, and how does a requester retrieve signed EAT?

**One evidence format, multiple retrieval paths**

```text
                         ┌──────────────────────────┐
                         │ MCU Runtime              │
                         │ assembles signed OCP EAT │
                         └────────────┬─────────────┘
                                      │
        ┌─────────────────────────────┼─────────────────────────────┐
        │                             │                             │
        ▼                             ▼                             ▼
 SPDM GET_MEASUREMENTS          SPDM VDM                      MCU mailbox
 block index 0xFD               GET_ATTESTATION               GET_ATTESTATION
 over MCTP / DOE                over MCTP                     SoC-local requester
```

| Requester | Retrieval path |
| --- | --- |
| BMC / pRoT | SPDM over MCTP |
| PCIe DOE requester | SPDM over DOE |
| AP OS / TEE | MCU mailbox |

**Discussion goal:** separate evidence format from transport/API surface.

---

## Category 6: Hitless Update and Measurement History

**Core question:** What remains valid across hitless update, and what does the verifier need for journey appraisal?

**What to look at**

```text
Hitless update
  ├─ Current state: DICE/DPE chain + signed OCP EAT
  ├─ Journey state: DICE/DPE chain + signed OCP EAT
  │                 + external replay/log context
```

| Area | Architecture position |
| --- | --- |
| Current-state appraisal | Covered by DICE/DPE chain and signed OCP EAT. |
| Journey appraisal | Requires replay context from outside MCU RT, such as BMC or update agent, in addition to DICE/DPE chain and signed OCP EAT. |
| ROM-stashed measurements | Fixed for the running Caliptra RT / hitless-update lifetime. |

**Hitless-update constraint**

The attestation policy is not allowed to change across MCU hitless updates. Target environments cannot be added or removed, and `SOC_IMAGE_LOAD_LIST` cannot be altered. The current design enforces this by storing a digest over the attestation policy/topology and requiring the digest to match after MCU hitless update.

**Discussion goal:** agree on current-state vs journey-state appraisal boundaries.

---

## Category 7: DICE/DPE Lineage and Claim Mapping

**Core question:** Which measurements appear in the DICE/DPE certificate chain and which appear in OCP EAT?

**Explanation:**

- The selected AK target determines the DPE ancestry that contributes to AK derivation.
- SoC TCB entries on the selected AK lineage appear in the DICE/DPE certificate chain.
- SoC TCB entries outside the selected AK lineage are reported as OCP EAT claims.
- SoC non-TCB entries are reported as OCP EAT claims from protected MCU-managed records.
- SVN reporting follows the Caliptra Core model: each applicable target environment reports current SVN and minimum SVN observed since cold boot.
- Caliptra-managed lineage contexts are internal to Caliptra; MCU-managed contexts remain active because MCU Runtime must use their handles.

**Discussion goal:** make AK lineage, DPE-backed state, SVN reporting, and EAT reporting explicit.

---

## Category 8: Confidential Compute and Attestation Scenarios

**Core question:** Is confidential compute part of this architecture, or should it be a separate proposal?

**Explanation:**

- Confidential compute can be treated as one possible attestation scenario.
- The base architecture should not be restricted to confidential compute.
- Attestation Manifest can carry authenticated policy inputs for configured scenarios.
- Detailed CC branch construction, fork-point semantics, and multi-AK handling need follow-on design discussion.
- The architecture should use broader wording such as “configured attestation scenarios” where appropriate.

**Discussion goal:** keep the base architecture general while identifying CC-specific follow-up work.

---
