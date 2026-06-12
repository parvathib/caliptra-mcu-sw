# [RFC] Caliptra Subsystem Attestation Architecture

## Abstract

This RFC proposes an attestation architecture for devices that integrate the Caliptra Subsystem, including Caliptra Core and MCU, to implement Attesting Environments for SoC identity, inventory, and confidential computing attestation.

Caliptra Core remains the hardware root of trust. It owns the DICE/DPE context chain, protects DPE state and AK material, and signs evidence in OCP EAT format for MCU RT. MCU RT is measured and authorized by Caliptra Core, acts as the Attesting Environment for downstream SoC firmware components, and assembles OCP EAT claims for SPDM evidence.

The proposal defines:

1. A common Caliptra DPE topology through `MCU_RT`.
2. SoC TCB firmware components represented as DPE contexts.
3. SoC non-TCB inventory claims managed by MCU RT in Software PCR-style storage backed by access-protected MCU SRAM.
4. Inventory and confidential-compute attestation models using platform-selected AK targets.
5. MCU RT-generated OCP EAT claims signed by Caliptra Core.

## Scope

This RFC applies to devices that use Caliptra Core and MCU RT to attest device identity, SoC firmware state, and inventory claims.

### Project Areas Affected

Existing surfaces used:

* `caliptra-sw`
  * Caliptra mailbox APIs used by MCU RT
  * existing DPE signing path used to sign evidence in OCP EAT format
  * existing DPE TCI-info access (`DPE_TAG_TCI`, `DPE_GET_TAGGED_TCI`)
  * existing `GET_IMAGE_INFO` mailbox command
* `caliptra-dpe`
  * existing `DeriveContext`
  * existing `UpdateContextMeasurement`
  * existing DPE tagged-TCI support

Updates needed:

* `caliptra-mcu-sw`
  * MCU RT image loading and firmware update integration with the measurement API
  * SPDM responder evidence generation from DPE-backed and Software PCR-backed claims
  * Measurement API surface
  * DPE handles Storage and Software PCR Storage capsules
  * Documentation and tests
* `caliptra-sw`
  * extend SoC Manifest IMC metadata and the `GET_IMAGE_INFO` response, if needed, so MCU RT can identify SoC TCB vs SoC non-TCB components, the configured AK target component, and the CC fork point. For backward compatibility, unused/reserved IMC metadata flag bits may be reused for these markers.

### Anticipated Specification / Documentation Changes

* `caliptra-mcu-sw`
  * Caliptra Subsystem attestation architecture documentation
  * MCU RT measurement API and evidence-generation documentation
  * Protected storage documentation for DPE handle management and Software PCR-style storage
  * SPDM/OCP EAT evidence-generation documentation
* `caliptra-sw`
  * SoC Manifest IMC metadata documentation for component class, AK target, and CC fork point markers
  * `GET_IMAGE_INFO` response documentation, if metadata is extended

### Security Posture per FIPS 140-3

No new cryptographic algorithms are introduced.

This proposal changes where evidence claims are assembled, not where root trust or signing keys are protected. Caliptra Core continues to protect DICE/DPE state and AK material. MCU RT assembles the OCP EAT claims and requests Caliptra Core to sign the evidence in OCP EAT format.

SoC non-TCB claims are stored in Software PCR-style storage backed by access-protected MCU SRAM. These claims do not participate in CDI or AK derivation and are appraised according to their non-TCB policy.

### Expected Impact to Memory Consumption

No significant Caliptra Core memory consumption increase is anticipated.

On the MCU side, the architecture requires a reserved MCU SRAM region that survives MCU hitless updates. This region is used for protected measurement state, including DPE context handle records and Software PCR-style records. The exact MCU memory impact is platform-dependent and scales with the number of SoC TCB contexts and non-TCB inventory records configured by the integrator.

### Expected Runtime Impact

Caliptra Core runtime impact is limited to existing mailbox/DPE operations used by MCU RT, including image authorization, DPE commands, TCI-info access, and signing evidence in OCP EAT format.

MCU runtime impact includes:

1. Additional bookkeeping during SoC image loading and firmware update.
2. Protected storage updates for SoC TCB DPE context handles.
3. Protected storage reads/writes for SoC non-TCB inventory claims.
4. Additional SPDM responder work to assemble OCP EAT evidence from stored TCB and non-TCB claims.

## Rationale

Caliptra Core provides device identity attestation through the DICE/DPE chain and remains the root of trust, DPE state owner, and signing authority. MCU RT loads, updates, and authorizes downstream SoC components, so it is the natural place to assemble SoC inventory evidence for those components.

Caliptra Core first measures and authorizes MCU RT, creating the MCU RT context in the DICE/DPE chain. After that point, MCU RT is trusted to collect and report measurements for downstream SoC components.

After MCU RT starts, MCU RT measures and authorizes SoC components against SoC Manifest metadata available through Caliptra Core. SoC TCB components are represented as DPE contexts created or updated by MCU RT. SoC non-TCB component claims are stored in MCU-managed Software PCR Storage backed by access-protected MCU SRAM. The SPDM responder in MCU RT reads both sources to assemble OCP EAT claims and requests Caliptra Core to sign the resulting evidence in OCP EAT format.

The common Caliptra-managed DPE topology is:

```text
Root("RTMR")
 └─ CCIV("CCIV")
    └─ ROM_Stash_1…N
       └─ SoC_Manifest_Vendor("SOMV")
          └─ SoC_Manifest_Owner("SOMO")
             └─ MCU_RT("MCFW")
                └─ <configured SoC TCB context tree>
```

For SoC inventory claims, the architecture separates two classes of firmware components:

1. **SoC TCB components** are represented as DPE contexts. Their measurements are protected by Caliptra DPE state. For inventory attestation, TCB contexts that are not already conveyed through the selected AK lineage are reported as OCP EAT claims.
2. **SoC non-TCB components** do not need DPE contexts. Their inventory claims are managed by MCU RT in Software PCR-style storage backed by access-protected MCU SRAM and are included directly in OCP EAT claims.

This avoids forcing every inventory item into the DPE tree while preserving a protected evidence path for TCB measurements.

## Implementation Tradeoffs

### SoC Manifest metadata encoding

MCU RT needs metadata to identify:

1. SoC TCB vs SoC non-TCB components.
2. The configured inventory AK target component.
3. The CC fork point.

This configuration must be authorized and signed, because it directly affects what
is measured into DPE, what is reported as OCP EAT inventory evidence, and which
DPE context is used as the AK target. Therefore, the configuration should come
from the signed SoC Manifest IMC metadata rather than from an unsigned MCU-local
configuration file or runtime policy table.

The preferred approach is to reuse unused or reserved SoC Manifest IMC metadata
flag bits. This keeps the existing IMC structure backward compatible: older
metadata remains valid, while newer platforms can opt in to the additional
meaning by setting the reserved bits. `GET_IMAGE_INFO` can then expose those
signed flag values to MCU RT.

Alternatives considered:

| Option | Tradeoff |
| --- | --- |
| Reuse unused/reserved IMC metadata flag bits | Preferred. Preserves structure compatibility and keeps the routing configuration inside signed SoC Manifest metadata. |
| Add new mandatory IMC fields | More explicit, but changes the signed metadata structure and can break backward compatibility for existing manifests/tools. |
| Use an MCU-local configuration table | Not preferred. The classification, AK target, and CC fork point must be cryptographically bound to the signed SoC Manifest chain, including the vendor and owner manifest authorities, not supplied by unsigned MCU-local policy. |

### AK target selection

The default inventory AK target is `MCU_RT`. Platforms may configure a downstream SoC TCB context as the AK target when that context must be part of AK derivation.

## Implementation Timeline

Target release: TBD

Expected work can be split across Caliptra Core and MCU workstreams.

Caliptra Core / `caliptra-sw`:

1. Define SoC Manifest IMC metadata flags for component class, AK target, and CC fork point.
2. Extend `GET_IMAGE_INFO`, if needed, to expose the required metadata to MCU RT.
3. Add tests and documentation for the metadata behavior.

MCU / `caliptra-mcu-sw`:

1. Implement Tock capsules for access-protected DPE context handle storage and Software PCR-style storage.
2. Implement the MCU measurement API and integrate it with image loading, firmware update and attestation flows
3. Update SPDM responder evidence generation to assemble OCP EAT claims.
4. Add tests and documentation for MCU storage, measurement, and SPDM evidence flows.

## Test Plan

End-to-end validation should cover:

* Cold boot inventory attestation flow with MCU RT measured by Caliptra Core.
* SoC TCB image load/update flow with DPE context creation and update.
* SoC non-TCB image load/update flow with Software PCR-style claim storage.
* Comprehensive Caliptra attester evidence containing both the DICE/DPE certificate chain and OCP EAT inventory claims.
* MCU hitless update flow preserving MCU-managed attestation state.
* Confidential-compute branch flow, if enabled, including branch isolation from the inventory/platform branch.

## Maintenance

Primary contacts: MCU RT owners (Parvathi Bhogaraju, Vishal Mhatre)

Affected components:

* MCU RT image loading and firmware update flows
* MCU RT SPDM responder
* Measurement API
* DPE Storage and Software PCR Storage capsules
* Caliptra Runtime DPE and signing interfaces
* SoC Manifest metadata definitions

Ongoing maintenance should be shared between Caliptra Runtime, caliptra-dpe, MCU RT owners.
