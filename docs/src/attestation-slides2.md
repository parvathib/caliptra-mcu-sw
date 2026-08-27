# SoC Attestation with Caliptra Subsystem

---

## Slide 1: Attestation

**Prove a Caliptra-integrated SoC is authentic, in a known configuration, and running known firmware.**

| Claim | Caliptra mechanism |
|-------|--------------------|
| Authentic | IDevID / LDevID derived from fuse UDS; manufacturer endorses IDevID |
| Known configuration | Caliptra ROM measures fuses : lifecycle state, debug state, SVN, etc. |
| Measured firmware | Caliptra ROM → FMC → Caliptra RT → MCU RT → SoC components, each measured before it runs, at boot and on hitless update |

**Evidence = device identity + device-signed measurements.**

- **Identity** — a DICE certificate chain binds the device public key to a PKI trust anchor. Caliptra protects the private key on behalf of the SoC, and key material never leaves the Caliptra security boundary
- **Measurements** — signed with that protected key, so a remote verifier authenticates them against the same chain

A verifier — BMC, peer device, or fleet manager — appraises the evidence against reference values and policy before granting access.

---

## Slide 2: Consistent Evidence Format

**A verifier must not need to know how we split work between Caliptra Core and MCU RT.**

- The Core/MCU split is an internal deployment choice — only the evidence crossing the device boundary is a verifier's concern
- The architecture therefore emits standard formats only, with no Caliptra-specific evidence encoding

| Evidence | Format |
|----------|--------|
| Identity and boot lineage | X.509 DICE certificate chain with `TcbInfo` / `MultiTcbInfo` |
| SoC component claims | OCP EAT claims in a `COSE_Sign1` envelope |
| Platform integrity | Signed PCR Quote |
| Conveyance | SPDM over MCTP / DOE, MCU mailbox |

- Any verifier that understands DICE, EAT/COSE, and SPDM appraises this device without knowing our internal partitioning

---

## Slide 3: Attestation Architecture

**Caliptra's DICE identity extends to MCU RT and SoC components**

- Add MCU RT as a DPE-managed PL0 context in the Caliptra DICE chain, so its firmware measurement and SVN are bound into attestation key derivation and DICE certificates
- MCU RT runs the **SPDM responder (RTR — Root of Trust for Reporting)**, responsible for collecting claims about the entire SoC and reporting them as attestation evidence (eg: OCP EAT claims)
- Caliptra Core owns DPE state and attestation key material — MCU RT assembles the evidence and asks Core to sign it, never holding the signing key itself
- Integrator static configuration, authenticated as part of the MCU RT image, sets the scope of reporting for SoC claims — which components are attested and how

---

## Slide 4: Attester Functions

The device's attester role splits into three functions. The rest of this deck follows them in order.

```
 ┌──────────────────────┐   ┌──────────────────────┐   ┌──────────────────────┐
 │  1. COLLECT CLAIMS   │──▶│  2. PROTECT CLAIMS   │──▶│  3. CONVEY EVIDENCE  │
 ├──────────────────────┤   ├──────────────────────┤   ├──────────────────────┤
 │ Who measures whom    │   │ Where measurements   │   │ What formats leave   │
 │ AE / TE layering     │   │ live, and who is     │   │ the device, and over │
 │ When measurement     │   │ allowed to mutate    │   │ which transports     │
 │ happens              │   │ them                 │   │                      │
 │                      │   │ DPE vs Software PCR  │   │ SPDM / DOE / mailbox │
 └──────────────────────┘   └──────────────────────┘   └──────────────────────┘
        Caliptra Core              Caliptra Core               MCU RT
        + MCU RT                   + MCU RT                    (SPDM responder)
```

---

<!-- _class: lead -->

# Part 1: Collecting Claims

**Who measures whom, and when**

---

## Slide 5: DICE Layered Attestation

Each boot layer **measures** the next layer, **derives a key** bound to that measurement, and **certifies** it. Trust is built bottom-up from hardware.

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│  Hardware (UDS)                                                                  │
│    │                                                                             │
│    ▼                                                                             │
│  Caliptra ROM  ──measures──▶  FMC                                                │
│    │  Derives IDevID (CSR), LDevID                                               │
│    │  Issues LDevID cert, AliasFMC cert                                          │
│    │    AliasFMC MultiTcbInfo:                                                   │
│    │      DEVICE_INFO: fwid=device_info_hash, svn=fuse_svn, flags=lifecycle/debug│
│    │      FMC_INFO:    fwid=FMC image digest, svn=fw_svn                         │
│    ▼                                                                             │
│  FMC  ──measures──▶  RT                                                          │
│    │  Issues AliasRT cert                                                        │
│    │    RT_INFO TcbInfo: fwid=RT image digest, svn=fw_svn                        │
│    ▼                                                                             │
│  RT  ──measures──▶  MCU RT                                                       │
│    │  Creates "MCFW" DPE context (digest + SVN)                                  │
│    │  DPE chain: "RTMR" → "CCIV" → ROM meas → "SOMV" → "SOMO" → "MCFW"           │
│    │  CertifyKey at the configured AK target issues the DPE leaf cert            │
│    │    MultiTcbInfo per context in chain, each with:                            │
│    │      fwids=tci_current, integrityRegisters=tci_cumulative, svn, tci_type    │
│    ▼                                                                             │
│  MCU RT  ──measures──▶  SoC Components                                           │
│    │  Assembles attestation evidence (EAT claims)                                │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## Slide 6: Attesting Environment / Target Environment Layers

| Layer | Attesting Environment | Target Environment | Evidence Produced |
|-------|----------------------|-------------------|-------------------|
| 1 | Caliptra ROM | FMC | AliasFMC cert — MultiTcbInfo: DEVICE_INFO (fwid=device_info_hash, svn=fuse_svn, flags=lifecycle/debug) + FMC_INFO (fwid=FMC image digest, svn=fw_svn) |
| 2 | FMC | RT | AliasRT cert — TcbInfo (RT_INFO): fwid=RT image digest, svn=fw_svn |
| 3 | RT | MCU RT<br />+<br />SoC TCB (if any) | DPE leaf cert — MultiTcbInfo: one TcbInfo per context in chain, each with fwids=tci_current, integrityRegisters=tci_cumulative, svn, tci_type |
| 4 | MCU RT (SPDM Responder as RTR) | SoC Components | OCP EAT claims |

Each layer transitions from Target Environment to Attesting Environment once it boots.

MCU RT is the last attesting environment in the chain — it gathers claims from every downstream SoC component and reports them as one signed evidence blob.

---

## Slide 7: The DPE Context Tree

The chain above `MCU_RT` is common to every integration. Device-specific variation happens **below** `MCU_RT`.

```
Root("RTMR")                            ← DPE root, Caliptra-managed
 └─ CCIV("CCIV")                        ← Caliptra Runtime context
    └─ ROM_Stash_1..N                   ← ROM-stashed measurements, immutable
       └─ SoC_Manifest_Vendor("SOMV")   ← vendor SoC manifest preamble
          └─ SoC_Manifest_Owner("SOMO") ← owner SoC manifest preamble
             └─ MCU_RT("MCFW")          ← MCU RT; MCU holds the handle
                └─ <integrator-selected SoC TCB contexts>
```

- Levels 0–4 are **Caliptra-managed lineage contexts** — internal DPE state, not command targets for MCU RT, so MCU RT cannot forge or replace them
- `MCU_RT` and everything below it are **active** contexts whose handles MCU RT holds and rotates on every use
- `SOMV` / `SOMO` put the vendor and owner manifest preambles into the lineage, so the authorization policy that admitted MCU RT is itself part of the measured chain

---

## Slide 8: What Is Measured, and How It Is Identified

`fw_id` is the join key that ties three separate artifacts to one attested component.

| SoC/Auth Manifest metadata<br />(Caliptra Runtime) | Attestation Manifest<br />(MCU RT image) | `SOC_IMAGE_LOAD_LIST`<br />(MCU RT image) |
|---|---|---|
| `fw_id = 0x1000`, `component_id = 7`,<br />`digest = H(FW_A)`, load addr | `fw_id = 0x1000`<br />`SOC_TCB_DPE = true`, `AK_TARGET = false` | `0x1000` |
| `fw_id = 0x1001`, `component_id = 8`,<br />`digest = H(FW_B)`, load addr | `fw_id = 0x1001`<br />`SOC_TCB_DPE = true`, `AK_TARGET = false` | `0x1001` |
| `fw_id = 0x1002`, `component_id = 9`,<br />`digest = H(FW_C)`, load addr | `fw_id = 0x1002`<br />`SOC_TCB_DPE = false`, `AK_TARGET = false` | `0x1002` |

- Caliptra RT uses SoC/Auth Manifest metadata for authorization and `GET_IMAGE_INFO(fw_id)`
- MCU RT uses `SOC_IMAGE_LOAD_LIST` order for image loading and DPE topology
- The Measurement API uses the Attestation Manifest to route each `fw_id` to DPE-backed or Software-PCR-backed state
- `component_id` locates the image payload; **`fw_id` identifies the attested component**

---

## Slide 9: Recovery Boot Flow (MCU RT)

```
1. SoC downloads Auth Manifest → Caliptra RT
   └─ RT validates vendor/owner signatures, stores SVN in PersistentData

2. SoC downloads MCU RT image → MCU SRAM
   └─ RT computes SHA-384, verifies digest against Auth Manifest

3. RT creates MCU RT DPE context
   └─ DeriveContext(measurement=digest, svn=manifest_svn, tci_type="MCFW")
   └─ Extends PCR31 with the measurement

4. MCU boots with verified firmware
   └─ Loads further SoC components via PLDM fw loading process.
   └─ May create more DPE contexts for security-critical components in PL0
      (depending on integrator choice)
```

**Result:** MCU RT identity is now captured in DPE (for DICE certs) and PCR31 (for PCR Quote).

Claims are collected at **load and authorization time**, not at request time — so answering an attestation request is cheap, and measurement is bound to the event that actually changes device state.

---

## Slide 10: Hitless Update Flow (MCU RT)

```
1. MCU sends new Auth Manifest → RT validates, updates SVN in PersistentData

2. MCU sends ACTIVATE_FIRMWARE → RT resets MCU, copies new image, verifies digest

3. Caliptra RT updates MCU RT DPE context in place (AUTHORIZE_AND_STASH with UPDATE_EXISTING)
   └─ RECURSIVE DeriveContext: tci_current = new digest, tci_cumulative = SHA384(old || new)
   └─ Extends PCR31 with new measurement
   └─ SVN in DPE context remains at recovery boot value until next full boot
```

- `tci_current` answers **what is running now** — appraised against reference values
- `tci_cumulative` answers **what this device has been** — the measurement journey since cold boot
- The instant a component changes, previously reported evidence is stale; the update is what triggers re-collection

**Constraint:** attestation policy cannot change across a hitless update. Target environments cannot be added or removed and `SOC_IMAGE_LOAD_LIST` cannot be altered. A digest over the policy and topology is stored and must match after update, so preserved measurement state can never be reused under a different policy.

---

<!-- _class: lead -->

# Part 2: Protecting Claims

**Where measurements live, and who is allowed to mutate them**

---

## Slide 11: Claim Protection in Caliptra Core

Caliptra Core both **holds the keys** used to sign evidence and **observes target environment state**, so claims cannot be spoofed on behalf of the environment being measured.

| Asset | Protection |
|-------|-----------|
| UDS, DICE secrets | Never leave Caliptra Core hardware |
| DPE context state (TCI values, lineage) | Owned and mutated only by Caliptra RT; MCU RT holds opaque handles, not state |
| Attestation key material | Derived inside Core via `CertifyKey`; private key never leaves Core |
| Caliptra-managed lineage contexts (`RTMR`, `CCIV`, ROM stash, `SOMV`, `SOMO`) | Not command targets for MCU RT |
| PCR31 | Extend-only, maintained by Caliptra Core |

- **Signing model:** MCU RT assembles the claims and the COSE signing structure, then asks Core to sign the bytes with the configured AK
- Core never has to parse OCP EAT or COSE, and MCU RT never possesses the signing key — a compromised MCU RT cannot mint evidence for a different device

---

## Slide 12: Claim Protection in MCU RT

Measurement state owned by MCU RT is split by **provenance requirement**, not by importance.

| | **SoC TCB** | **SoC non-TCB** |
|---|---|---|
| Backing | DPE-backed TCI state in Caliptra | Structured record in access-protected MCU SRAM (Software PCR Storage) |
| Provenance | Caliptra / DPE-backed | MCU-managed |
| Mutated by | Caliptra RT, via MCU RT DPE commands | MCU RT Measurement API only |
| Reported in | DICE/DPE cert chain if on AK lineage, else OCP EAT | OCP EAT |

- Classification is **integrator policy**, chosen per `fw_id` in the Attestation Manifest — it does not imply a universal TCB boundary
- Only the Measurement API layer may mutate either store; image loading, firmware update, the OCP EAT encoder, and the SPDM responder all go through it and never write the stores directly
- "Software PCR Storage" is the current MCU implementation of structured MCU-managed measurement records — it does not imply non-DPE claims are raw PCR values

---

## Slide 13: Storage Class vs AK Lineage

Being DPE-backed does **not** mean contributing to the attestation key. The AK target selects the lineage; the storage class selects the provenance. They are independent.

```
SOC_IMAGE_LOAD_LIST stages:        DPE-backed context tree:
  1. 0x1000 (TCB, not AK)          Root("RTMR")
  2. 0x1001 (TCB, not AK)           └─ CCIV("CCIV")
  3. 0x1002 (non-TCB)                  └─ ROM_Stash_1..N
  4. 0x1003 (non-TCB)                     └─ SOMV
                                             └─ SOMO
Software PCR Storage:                           └─ MCU_RT("MCFW")  [AK target]
  ├─ record(0x1002)                                  └─ 0x1000
  └─ record(0x1003)                                     └─ 0x1001
```

| Claim | Conveyed by |
|-------|------------|
| TCB components **on** the AK lineage | DICE/DPE certificate chain |
| TCB components **outside** the AK lineage | OCP EAT claims |
| Non-TCB components | OCP EAT claims, from Software PCR Storage |

`CertifyKey` walks from the selected node **upward to root** and never includes descendants. Above, `0x1000` and `0x1001` are DPE-backed but sit below the AK target, so they are reported as OCP EAT claims.

SVN reporting follows the Caliptra Core model: each applicable target environment reports current SVN and minimum SVN observed since cold boot.

---

<!-- _class: lead -->

# Part 3: Conveying Evidence

**What leaves the device, and over which transport**

---

## Slide 14: Attestation Evidence Formats

A verifier must validate the DICE certificate chain **first**, then appraise the evidence.

```
                    ┌───────────────────────────────────────────────────────────────────┐
                    │                      DICE Certificate Chain                       │
                    │                       (Common Trust Anchor)                       │
                    │ Root CA → IDevID → LDevID → AliasFMC → AliasRT → DPE Leaf (MCFW)  │
                    └──────────────────────┬────────────────────┬───────────────────────┘
                                           │                    │
                    ┌──────────────────────▼──┐  ┌──────────────▼──────────────────────┐
                    │ OCP EAT Claims          │  │ PCR Quote                           │
                    ├─────────────────────────┤  ├─────────────────────────────────────┤
                    │ Assembled by: MCU RT    │  │ Extended by: Caliptra Core          │
                    │ Signed by: DPE leaf key │  │ Signed by: FMC Alias key            │
                    │ Contains: per-component │  │ Contains: Signed  PCR Quote         │
                    │   digest, SVN,          │  │                                     │
                    │   structured claims     │  │                                     │
                    │ Use case: identity &    │  │ Use case: platform integrity        │
                    │   policy decisions      │  │   verification                      │
                    └─────────────────────────┘  └─────────────────────────────────────┘
```

---

## Slide 15: The DICE Certificate Chain

```
Root CA
  └─ IDevID (CSR — cert issued by external Root CA)
       └─ LDevID
            └─ AliasFMC
                 └─ AliasRT
                      └─ DPE Leaf (RTMR, CCIV, ROM meas, SOMV, SOMO, **MCFW**, SoC TCB if any)
```

- Each certificate signs the next and embeds the TcbInfo of the layer it attests
- The chain is rooted in hardware (UDS → IDevID) and traces through every boot layer
- A verifier validates this chain **first** — this is the trust foundation for all evidence
- The DPE leaf carries `MultiTcbInfo`, one `TcbInfo` per context in the AK lineage, which makes the measurements behind the key explicit rather than implied

---

## Slide 16: Evidence Retrieval Paths

**One evidence format, multiple retrieval paths.**

```
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
|-----------|---------------|
| BMC / pRoT | SPDM over MCTP |
| PCIe DOE requester (eg: confidential-compute device) | SPDM over DOE |
| AP OS / TEE | MCU mailbox |

There is exactly **one** attestation report — the signed OCP EAT. SPDM carries it; SPDM does not create a second, competing attestation report.

---

## Slide 17: Verifier Flow

```
 Device                             Verifier                      Relying Party
 ──────                             ────────                      ─────────────
 DICE chain + signed
 OCP EAT (+ PCR Quote)  ─────────▶
                                    1. Validate DICE chain
                                       → authenticate the device
                                    2. Verify EAT / Quote signature
                                       against the AK anchored in
                                       that chain
                                       → authenticate the claims
                                    3. Compare tci_current / digests
                                       against reference values
                                       and policy
                                    4. Replay tci_cumulative
                                       → appraise the journey
                                    5. Attestation result  ────────▶
                                                                  6. Admit / deny /
                                                                     remediate / log
```

**Verifier flow:** Validate DICE chain → Retrieve evidence (EAT or PCR Quote) → Verify signature → Compare measurements against reference values → Produce attestation result.

Step 4 needs a replay log of the extended measurements. That context comes from **outside MCU RT** (eg: BMC or update agent) — see Open Items.

---

## Slide 18: Open Items

1. **SoC Manifest Claims** — The attestation evidence should include claims about the Authorization Manifest itself (vendor policy, signing keys). `SOMV`/`SOMO` put the manifest preambles into the DPE lineage, but the claim representation that tells a verifier *who authorized* MCU RT is still TBD.

2. **SoC Component Locality** — Decide per-component PL0 vs PL1 placement. May need `AUTHORIZE_AND_STASH` enhancement to support target locality in one call.

3. **Hardware/Software Target Environments** — `fw_id` covers MCU-managed firmware only. Target environments that are not firmware images need their own identifiers, measurement sources, and claim schema, without overloading `fw_id`.

4. **Journey Replay Log** — Journey appraisal needs a log of the extended measurements. Producing and conveying it is outside MCU RT today and is not yet defined.

5. **Confidential Compute** — Treated as one possible attestation scenario, not the base case. Branch construction, fork-point semantics, and multi-AK handling need follow-on design.

6. **Open for discussion** — reference value format and tooling; the endorsement artifact that endorses the Caliptra RoT; who hosts the verifier in the target deployment.
