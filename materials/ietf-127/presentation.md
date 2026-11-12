---
title: EAT Profile for Trustworthy Device Assignment
theme: Next
paginate: true
footer: EAT DA — RATS WG — IETF 127
autoscale: true
---

# EAT DA 00

## [`draft-ietf-rats-eat-da-00`](https://www.ietf.org/archive/id/draft-ietf-rats-eat-da-00.html)

### IETF 127, San Francisco, RATS WG

---

## Where we are coming from / where we are

Starting point: DAT as simplified, abstract view of the device technology (SPDM, CXL, etc.)

Verifiers asked to peek into the protocol (they want to provide own nonce) => we bolted on raw pieces (e.g., VCA, signature)

-00: one `spdm-claims` bag mixing raw and extracted claims

---

## Where we want to go

Proposal for -01: explicitly split raw from extracted

| | `spdm-raw-claims` | `spdm-extracted-claims` |
|---|---|---|
| What | Verbatim transcript of relevant protocol transactions, lightly EAT-ified | Abstract, but still faithful, view of core properties |
| Contents | VCA (capability negotiation), measurement exchanges (nonces, signature), certificate chain, MEL | Parsed measurements, certificates (device identity), MEL |
| Target | Verifiers that re-check the protocol run | Verifiers that want identity + state |
| Main feature | Own-nonce freshness | Simpler appraisal logic |

---

## From:

```
   spdm-claims = {
     &(eat_profile: 265) => "tag:linaro.org,2025:device-spdm#1.0.0"
     &(measurements: 3802) => spdm-measurements
     &(certificates: 3803) => spdm-certificates
     ? &(challenge: 3807) => {
       &(slot: 1) => 0 .. 7,
       &(requester-nonce: 2) => bytes .size 32,
       &(responder-nonce: 3) => bytes .size 32,
       &(combined-spdm-prefix: 4) => bytes .size 100,
       &(IL1: 5) => bytes,
       &(base-hash-algo: 6) => hash-algorithm-type,
       &(signature: 7) => bytes,
     },
     ? &(device-interface-report: 3808) => tdisp-device-interface-report
     ? &(vca: 3804) => bytes
   }
```

---

## To:

```
   spdm-raw-claims = {
     &(eat_profile: 265) => "tag:ietf.org,2026:eat/da/spdm-raw#1.0"
     &(vca: TBD1) => bytes            ; negotiation
     &(measurements: TBD2) => bytes   ; GET_MEASUREMENTS|MEASUREMENTS...
     &(certificate-chain: TBD3) => bytes
     ? &(measurement-extension-log: TBD4) => bytes
   }

   spdm-extracted-claims = {
     &(eat_profile: 265) => "tag:ietf.org,2026:eat/da/spdm-extracted#1.0"
     &(extracted-measurements: TBD5) => { + block-id => spdm-measurement }
     ? &(raw-measurements: TBD6) => { vca, measurements }  ; optional raw for dbg
     &(certificates: TBD7) => [ + bytes ]
     ? &(measurement-extension-log: TBD8) => mel-data
   }
```

---

## Supported flows

- **Collector** gets SPDM from the device, emits **raw**
- **Extractor** turns raw into **extracted**
- **Lead Attester => Verifier/RP:** raw, extracted, or both
- **Event Log** keeps raw (replay, audit)
- **Sealing Log + Secret Release Policy** consume extracted only (stable)

![right fit](img/usecases.pdf)

---

## Use case 1: SPDM direct to the remote Verifier

Lead Attester forwards Verifier nonce to the Device Attester (via Collector)

Collector hands the raw SPDM transcript straight to the Lead Attester, which forwards it

![right fit](img/usecase1.pdf)

---

## Use case 2: Capture raw "events"

Device event log (attach/detach) uses the raw format as events payload

![right fit](img/usecase2.pdf)

---

## Use case 3: extracted to the remote Verifier

Extractor abstracts the raw transcript; the Verifier/RP gets identity + state only

Lead Attester takes responsibility for signing the Verifier nonce

![right fit](img/usecase3.pdf)

---

## Use case 4: sealing (local)

Extracted (stable) claims feed the sealing log and the secret release policy

![right fit](img/usecase4.pdf)

---

## Questions

- Is the split reasonable?
- Do you have further use cases that we should consider?
