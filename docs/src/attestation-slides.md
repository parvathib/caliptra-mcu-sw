# SoC Attestation with MCU RT

---

## Slide 1: Proposed Attestation Architecture

**Extend Caliptra's DICE identity to MCU RT and SoC components**

- Add MCU RT as a DPE-managed PL0 context in the Caliptra DICE chain, so its firmware measurement and SVN are bound into attestation key derivation and DICE certificates
- MCU RT runs the **SPDM responder (RTR — Root of Trust for Reporting)**, responsible for collecting claims about the entire SoC and reporting them as attestation evidence (eg: OCP EAT claims)

---

## Slide 2: DICE Layered Attestation

Each boot layer **measures** the next layer, **derives a key** bound to that measurement, and **certifies** it. Trust is built bottom-up from hardware.

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│  Hardware (UDS)                                                                          │
│    │                                                                                     │
│    ▼                                                                                     │
│  Caliptra ROM  ──measures──▶  FMC                                                        │
│    │  Derives IDevID (CSR), LDevID                                                       │
│    │  Issues LDevID cert, AliasFMC cert                                                  │
│    │    AliasFMC MultiTcbInfo:                                                           │
│    │      DEVICE_INFO: fwid=device_info_hash, svn=fuse_svn, flags=lifecycle/debug        │
│    │      FMC_INFO: fwid=FMC image digest, svn=fw_svn                                   │
│    ▼                                                                                     │
│  FMC  ──measures──▶  RT                                                                  │
│    │  Issues AliasRT cert                                                                │
│    │    RT_INFO TcbInfo: fwid=RT image digest, svn=fw_svn                                │
│    ▼                                                                                     │
│  RT  ──measures──▶  MCU RT                                                               │
│    │  Creates "MCFW" DPE context (digest + SVN)                                          │
│    │  DPE chain: "RTMR" → "CCIV" → ROM meas → "MCFW"                                    │
│    │  CertifyKey on "MCFW" issues DPE leaf cert                                          │
│    │    MultiTcbInfo per context in chain, each with:                                    │
│    │      fwids=tci_current, integrityRegisters=tci_cumulative, svn, tci_type             │
│    ▼                                                                                     │
│  MCU RT  ──measures──▶  SoC Components                                                   │
│    │  Assembles attestation evidence (EAT claims)                            │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Slide 3: Attesting Environment / Target Environment Layers

| Layer | Attesting Environment | Target Environment | Evidence Produced |
|-------|----------------------|-------------------|-------------------|
| 1 | Caliptra ROM | FMC | AliasFMC cert — MultiTcbInfo: DEVICE_INFO (fwid=device_info_hash, svn=fuse_svn, flags=lifecycle/debug) + FMC_INFO (fwid=FMC image digest, svn=fw_svn) |
| 2 | FMC | RT | AliasRT cert — TcbInfo (RT_INFO): fwid=RT image digest, svn=fw_svn |
| 3 | RT | MCU RT<br />+<br />SoC TCB (if any) | DPE leaf cert — MultiTcbInfo: one TcbInfo per context in chain, each with fwids=tci_current, integrityRegisters=tci_cumulative, svn, tci_type |
| 4 | MCU RT (SPDM Responder as RTR) | SoC Components | OCP EAT claims |

Each layer transitions from Target Environment to Attesting Environment once it boots.

---

## Slide 4: The DICE Certificate Chain

```
Root CA
  └─ IDevID (CSR — cert issued by external Root CA)
       └─ LDevID
            └─ AliasFMC
                 └─ AliasRT
                      └─ DPE Leaf (RTMR, CCIV, ROM meas, **MCFW**, SoC TCB if any)
```

- Each certificate signs the next and embeds the TcbInfo of the layer it attests
- The chain is rooted in hardware (UDS → IDevID) and traces through every boot layer
- A verifier validates this chain **first** — this is the trust foundation for all evidence

---

## Slide 5: Recovery Boot Flow (MCU RT)

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
   └─ May create more DPE contexts for security-critical components in PL0 (depending on integrator choice)

```

**Result:** MCU RT identity is now captured in DPE (for DICE certs) and PCR31 (for PCR Quote).

---

## Slide 6: Hitless Update Flow (MCU RT)

```
1. MCU sends new Auth Manifest → RT validates, updates SVN in PersistentData

2. MCU sends ACTIVATE_FIRMWARE → RT resets MCU, copies new image, verifies digest

3. Caliptra RT updates MCU RT DPE context in place (AUTHORIZE_AND_STASH with UPDATE_EXISTING)
   └─ RECURSIVE DeriveContext: tci_current = new digest, tci_cumulative = SHA384(old || new)
   └─ Extends PCR31 with new measurement
   └─ SVN in DPE context remains at recovery boot value until next full boot
```

---

## Slide 7:  Attestation Evidence Formats

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

**Verifier flow:** Validate DICE chain → Retrieve evidence (EAT or PCR Quote) → Verify signature → Compare measurements against reference values → Produce attestation result.

---

## Slide 8: Open Items

1. **SoC Manifest Claims** — The attestation evidence should include claims about the Authorization Manifest itself (vendor policy, signing keys). Without this, verifiers know *what* MCU RT is running but not *who authorized it*. Mechanism TBD.

2. **SoC Component Locality** — Decide per-component PL0 vs PL1 placement. May need `AUTHORIZE_AND_STASH` enhancement to support target locality in one call. 
