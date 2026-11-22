---
title: "An EAT Profile for Trustworthy Device Assignment"
abbrev: "EAT DA"
category: std

docname: draft-ietf-rats-eat-da-latest
submissiontype: IETF
number:
date:
consensus: false
v: 3
area: "Security"
workgroup: "Remote ATtestation ProcedureS"
keyword:
 - attestation
 - device assignment
 - EAT
venue:
  group: "Remote ATtestation ProcedureS"
  type: "Working Group"
  mail: "rats@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/rats/"
  github: "ietf-rats-wg/draft-ietf-rats-eat-da"
  latest: "https://ietf-rats-wg.github.io/draft-ietf-rats-eat-da/draft-ietf-rats-eat-da.html"

author:
 -
    fullname: Mathieu Poirier
    organization: Linaro
    email: mathieu.poirier@linaro.org
 -
    fullname: Henk Birkholz
    organization: Fraunhofer SIT
    email: henk.birkholz@ietf.contact
 -
    fullname: Thomas Fossati
    organization: NVIDIA
    email: tfossati@nvidia.com

normative:
  RFC9711: rats-eat
  RFC5280: pkix
  RFC4514: dn-string-rep
  IANA.cwt:
  STD94:
    -: cbor
    =: RFC8949
  RFC8392: cwt
  RFC9781: uccs
  RFC9782: eat-media-types
  RFC9052: cose
  RFC9053: cose-algs
  RFC9334: rats-arch
  RFC9999: cmw
  I-D.ietf-rats-epoch-markers: epoch-markers
  RFC10013: eatmc

informative:
  I-D.ietf-rats-corim: corim
  SPDM:
    -: spdm
    target: https://www.dmtf.org/sites/default/files/standards/documents/DSP0274_1.3.2.pdf
    author:
    - org: DMTF
    title: "Security Protocol and Data Model (SPDM) Specification Version: 1.3.2"
    date: 2024-08-21
  PCIe:
    -: pcie
    author:
    - org: PCI-SIG
    title: "PCI Express Base Specification Revision 7.0"
    date: 2025-06-05

entity:
  SELF: "RFCthis"

--- abstract

In confidential computing, device assignment (DA) is the method by which a device (e.g., network adapter, GPU), whether on-chip or behind a PCIe Root Port, is assigned to a Trusted Virtual Machine (TVM).
For the TVM to trust an assigned device, the device must provide the TVM with attestation Evidence confirming its identity, the state of its firmware and configuration.

Since Evidence claims can be processed by 3rd party entities (e.g., Verifiers, Relying Parties) external to the TVM, there is a need to standardize the representation of DA-related information in Evidence to ensure interoperability.
This document defines an attestation Evidence format for DA as an EAT (Entity Attestation Token) profile.

--- middle

# Introduction

In confidential computing, device assignment (DA) is the method by which a device (e.g., network adapter, GPU), whether on-chip or behind a PCIe Root Port, is assigned to a Trusted Virtual Machine (TVM).
Most confidential computing platforms (e.g., Arm CCA, AMD SEV-SNP, Intel TDX) provide DA capabilities.
Such capabilities prevent execution environments or software components that are untrusted by the TVM (including other TVMs and the host hypervisor) from accessing or controlling a device that has been assigned to the TVM.
This includes, for example, protection of device MMIO interfaces and device caches.
From a trust perspective, DA allows a device to be included in the TVM's Trusted Computing Base (TCB).
For the TVM to trust the device, the device must provide the TVM with attestation Evidence confirming its identity and the state of its firmware and configuration.

{{fig-ratsd}} gives an overview of the architecture targeted by this specification.
Devices assigned to a TVM must be authenticated by a 3rd party verifier before being accepted into the TCB.

~~~ aasvg
      .---------------.
     | Verifier / RP   |
      '---------------'
              ^
  .-----------+------------------------------------.
 |            |                                     |
 |  Attester  |                .-----------------.  |
 |            |              .-+---------------. |  |
 |  .---------+---------.  .-+---------------. | |  |
 |  | TVM     |         |  | Assigned Device | | |  |
 |  |         v         |  |                 | | |  |
 |  | .---------------. |  |                 | | |  |
 |  | | Lead Attester | |  |                 | | |  |
 |  | '---------------' |  |                 | | |  |
 |  |     ^       ^     |  |                 | | |  |
 |  |     |       |     |  |                 | | |  |
 |  |     v       v     |  | .-------------. | | |  |
 |  | .---+-------+---. |  | | Device      | | | |  |
 |  | | Guest Kernel  | |  | | Attester    | | | |  |
 |  | '---------------' |  | '-------------' | +-'  |
 |  |     ^       ^     |  |     ^           +-'    |
 |  '-----|-------|-----'  '-----|-----------'      |
 |        |       |              |                  |
 |  .-----|-------|-----.  .-----|---------------.  |
 |  |     |       |     |  | .---|-------------. |  |
 |  |     v        '---------+--'  Host Kernel | |  |
 |  | .---------------. |  | '-----------------' |  |
 |  | | Platform      | |  |                     |  |
 |  | | Attester      | |  |                     |  |
 |  | '---------------' |  |                     |  |
 |  |                   |  |                     |  |
 |  | Confidential      |  | Untrusted           |  |
 |  | Platform          |  | Platform            |  |
 |  '-------------------'  '---------------------'  |
 |                                                  |
  '------------------------------------------------'
~~~
{: #fig-ratsd title="Confidential VM with Trusted Device(s)" align="center" }

This document defines an attestation Evidence format for DA as an EAT {{-rats-eat}} profile.
The format is designed to be generic, extensible and architecture-agnostic.
Ongoing work on DA concentrates on PCIe devices that support the SPDM protocol {{-spdm}}.
As such, this document focuses on establishing the overall framework and formalizing an Evidence format for SPDM-compliant PCIe devices.
It does not support on-chip SPDM-compliant device, something that is left for future consideration.

The TVM uses the platform, i.e., PCIe Data Object Exchange (DOE), to transport SPDM packets, but the SPDM protocol uses its own mechanisms and messages to guarantee the authenticity and privacy of the information it carries.
It defines specific messages for creating and exchanging cryptographic keys between requester and responder to establish secure communication.
The cryptographic algorithms are selected based on the pre-negotiated capabilities of both parties (Section 10.16 of {{-spdm}}).
The SPDM protocol does not rely on the hypervisor or host for anything else other than the physical delivery of content between the trusted and non-trusted worlds.

This format is based on the information provided by the SPDM protocol without imposing additional security constraints.
It is incumbent upon other entities to describe, select and enforce those additional security constraints based on operational requirements.

Since other bus architectures and protocols are expected to be supported as the technology gains wider adoption, provisions have been made for the definition of other Evidence formats such as Compute Express Link (CXL) and the Coherent Hub Interface (CHI).
This list is by no means exhaustive and is expected to expand.
{{extend}} outlines the requirements for incorporating new bus technologies into the Device Assignment Token (DAT) framework.
Lastly, live migration of a TVM from one host to another is currently not addressed by the SPDM specification and therefore not covered herein.

As discussed in {{Section 3.4 of -rats-arch}}, a lead Attester may also act as an internal Verifier.
For example, the lead Attester would verify the identity of the SPDM responder locally using a provisioned trust anchor.
If satisfied, it would then repackage the sub-Attester claims in the format defined in this document.
In other words, the lead Attester would use the framework and claims defined in this document to create Attestation Results.
In this sense, DAT can be considered to provide both an Attestation Result format and an Evidence format.
Generally, the distinction between Evidence and Attestation Results, when applied to Conceptual Messages generated by a composite device, may not always be completely transparent to the receiver.
Note that when using CMW {{-cmw}} to wrap the collected claims, the `ind` field could be used to refine the Conceptual Message type expressed by the EAT media type {{-eat-media-types}}.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

PCIe devices maximize hardware efficiency by splitting a single physical card into multiple independent logical devices called functions (see Section 9 of {{-pcie}}).
Because each physical or virtual function operates in isolation, multiple VMs can directly and safely share the same underlying hardware simultaneously.
In this document, the terms "*device*", "*interface*" and "*device interface*" refer to the physical or virtual function of a device.

# Device Assignment Token (DAT) Claims {#dat-claims}

The Device Assignment Token (DAT) is the encompassing envelope for the individual device claims to be presented.
A DAT can be used as a standalone entity but can also be embedded in a larger, platform-specific attestation token.
A DAT consists of an EAT profile identifier, a nonce and an EAT submodule ({{Section 4.2.18 of -rats-eat}}) that contains any number of individual device claims.
Each individual submodule is the combination of a device name and a standard claims-set based on the bus or protocol the device supports.
The syntax of the device name depends on the type of bus or protocol used.
Each name consists of two parts joined by a colon: a namespace and a bus-specific name.
See {{spdm-submod-name}} for SPDM devices, and {{pcie-legacy-submod-name}} for legacy PCIe devices.
As previously mentioned, this draft currently defines the claims-sets (raw and extracted) for SPDM compliant devices and PCIe legacy devices that do not support the SPDM protocol.
Careful consideration was also given to the overall design in order to leave room for future expansion.

~~~ cddl
{::include-fold cddl/da-token.cddl}
~~~

## SPDM Claims {#spdm-claims}

A claims-set is expected to be present for each SPDM compatible device to be attested.
SPDM claims come in two flavours, each with its own EAT profile identifier and claims-set:

* Raw claims ({{spdm-raw-claims}}) carry a verbatim transcript of the relevant SPDM message exchanges, each carried in a dedicated claim.
  They target Verifiers that want to re-check the protocol run, including the device's signature, which covers the nonce supplied by the Verifier.
* Extracted claims ({{spdm-extracted-claims}}) carry a synthesised, yet faithful, view of the core properties of the device: its identity and the state of its firmware and configuration.
  They target Verifiers that only need identity and state, and benefit from simpler appraisal logic.

Typically (see {{fig-spdm-flow}}), an Evidence Collector obtains the SPDM artefacts from the device and emits raw claims.
An Evidence Extractor can then turn the raw claims into extracted claims.
A Lead Attester can forward raw claims or extracted claims to the Verifier.
Extracted claims can optionally carry the raw artefacts they were derived from ({{spdm-raw-artefacts}}).

~~~ aasvg
                 .-------------.
                | Verifier / RP |
                 '-------------'
                        ^
                        | raw/extr
========================|=====================================
                        |                   .----------------.
                .-------+-------.  raw      | Secret Release |
      .-------->| Lead Attester |<-----.    | Policy (PCR)   |
     |          '---------------'       |   '-------+--------'
     |                  ^               |           |
     |                  | extr          |       .---+---.
     | log      .-------+------------.  |      |         |
     | raw      | Evidence Extractor +--)----->|'-------'|
     |          '--------------------'  | extr | Sealing |
     |                  ^               |      | Log     |
 .---+---.              | raw           |       '-------'
|         |             |               |
|'-------'|     .-------+------------.  |
| Event   |<----+ Evidence Collector +-'
| Log     | raw '--------------------'
 '-------'              ^
                        | SPDM
========================|=====================================
                        |
                .-------+---------.
                | Device Attester |
                '-----------------'
~~~
{: #fig-spdm-flow title="SPDM Collection/Extraction Flows" align="center" }

A DAT MUST NOT contain both a raw and an extracted claims-set for the same device.
An Attester that needs to convey both forms for a device uses the extracted claims-set and includes the raw artefacts in it ({{spdm-raw-artefacts}}).

### SPDM Raw Claims {#spdm-raw-claims}

The raw claims-set conveys the SPDM artefacts as opaque byte strings, without interpreting its content.
It consists of:

* `vca`: the Negotiated State Preamble, i.e., the concatenation of messages GET_VERSION, VERSION, GET_CAPABILITIES, CAPABILITIES, NEGOTIATE_ALGORITHMS, and ALGORITHMS last exchanged between the SPDM Requester and Responder.
  It conveys the full negotiated state (version, capabilities and algorithms).
* `raw-measurements` (optional): the transcript of the GET_MEASUREMENTS requests and MEASUREMENTS responses.
  It is absent if the device does not support measurements (see the MEAS_CAP field in the CAPABILITIES response of {{-spdm}}).
  Since the transcript includes the nonces and the signature generated by the device, a Verifier can check the signature, and thus freshness, by itself.
* `certificate-chain`: the certificates provided by the device, in the format defined in {{spdm-certificates}}.
* `raw-measurement-extension-log` (optional): the Measurement Extension Log (MEL), as defined in {{-spdm}}, either inlined as a byte string or detached ({{mel-detached}}).
* `tdisp-device-interface-report` (optional): the TDISP Device Interface Report ({{interface-report}}).

~~~ cddl
{::include-fold cddl/spdm-raw-claims.cddl}
~~~

### SPDM Extracted Claims {#spdm-extracted-claims}

The extracted claims-set conveys information that has been parsed out of the SPDM artefacts.
It consists of:

* `extracted-measurements` (optional): the measurements ({{spdm-measurements}}).
  It is absent if the device does not support measurements (see the MEAS_CAP field in the CAPABILITIES response of {{-spdm}}).
* `certificates`: the certificates ({{spdm-certificates}}), from which the identity of the device is derived.
* `raw-artefacts` (optional): the raw transcripts from which the claims above were extracted ({{spdm-raw-artefacts}}).
* `extracted-measurement-extension-log` (optional): the Measurement Extension Log, either inlined or detached ({{mel-detached}}).
  The inlined form (`mel-data`) carries, together with the log itself, the metadata needed to interpret it that would otherwise be found in the VCA: the selected MEL specification (`mel-specification-sel`) and the selected measurement specification (`measurement-specification-sel`).
* `tdisp-device-interface-report` (optional): the TDISP Device Interface Report ({{interface-report}}).

~~~ cddl
{::include-fold cddl/spdm-extracted-claims.cddl}
~~~

#### Measurements Claim {#spdm-measurements}

The SPDM specification reserves measurement indexes between 240 and 255, leaving space for 239 measurements per device.
It should be noted that measurements formalized herein follow the DMTF measurement specification (see Table 55 of {{-spdm}}).

~~~ cddl
{::include-fold cddl/spdm-measurements.cddl}
~~~

##### Measurement

SPDM measurements start with a component type that reflects one of the 10 categories defined by the SPDM specification.
Following is the measurement itself represented by either a raw bitstream or a digest.
The size of the digest value is derived from the measurement hash algorithm conveyed by the SPDM ALGORITHMS message response.

~~~ cddl
{::include-fold cddl/spdm-measurement.cddl}
~~~

#### Certificate Claims {#spdm-certificates}

According to the specification, SPDM compliant devices should support at most 8 slots, with slot 0 populated by default.
Slot 0 SHALL contain a certificate chain that follows the Device certificate model or the Alias certificate model.
Regardless of the certificate model used, a certificate chain comprises one or more DER-encoded X.509 v3 certificates {{-pkix}}.
The certificates MUST be concatenated with no intermediate padding.
As described in paragraph 355 of Section 10.6.1 of {{-spdm}}, the certificates in the chain are sorted in descending order, with the end-entity certificate (also known as the "leaf") appearing last.
According to the same paragraph, a trust anchor may appear as the first certificate in the chain.
If so, it should not be used to validate the chain without first confirming its presence in a trust anchor store.

~~~ cddl
{::include-fold cddl/spdm-certificates.cddl}
~~~

#### Raw Artefacts Claim {#spdm-raw-artefacts}

The `raw-artefacts` claim carries the `vca` and `raw-measurements` claims defined in {{spdm-raw-claims}}, with the same semantics.
All its members are optional.
It is intended for debugging, and for Verifiers that want to cross-check the extracted claims against the transcript.

#### Detached Measurement Extension Log {#mel-detached}

A MEL can be conveyed detached from the claims-set, e.g., because of its size.
In this case, the claim holds a map with a URI (`href`) from which the CBOR-encoded inlined form can be retrieved, and the digest (`thumbprint`) of that encoding, in the `digest` format of {{Section 4.2 of -eatmc}}.
The Verifier MUST check the retrieved MEL against the thumbprint.

#### TDISP Device Interface Report Claim {#interface-report}

A TDISP Device Interface Report can only be obtained if the device interface has transitioned to the CONFIG_LOCK or RUN state of the TDISP state machine.
Once in either of these states, the device's firmware ensures that modifications to the interface can't be made by an untrusted party.
The claim is an optional member of both the raw and the extracted claims-sets.
When it is present in the raw claims-set, trust in the device firmware's implementation is imparted by its measurement and by the signature that the device generates over the measurement transcript, both verifiable by a 3rd party entity.
This statement does not apply to the extracted claims-set, which does not carry the signature.

A TDISP Device Interface Report begins with various bitfields indicating the state and characteristics of the PCIe device interface.
Next are 3 register fields pertaining to MSI-X (Message Signalled Interrupts), LNR (Lightweight Notification Requester) and TPH (TLP Processing Hints) capabilities.
MMIO ranges are assigned from PCIe BAR(s) and provide information about the memory areas a device is working with.
More information on the MMIO range bitfields and the ones defined as part of the device interface field (above) can be found in the TDISP section of the PCI Express specification.
The last field is device-specific and optionally included to convey additional configuration information about the device.

The `tdisp-device-interface-report` map MUST NOT be empty.

~~~ cddl
{::include-fold cddl/tdisp-device-interface-report.cddl}
~~~


### Submodule Naming {#spdm-submod-name}

The namespace used for SPDM submodules is "spdm".

The name associated with an SPDM submodule is extracted from the leaf certificate of the relevant device.

* If the leaf certificate contains a Subject Alternative Name of type DMTFOtherName, the submodule name is the value contained in `ub-DMTF-device-info`.
For example: "spdm:ACME:WIDGET:0123456789".
* Otherwise, the submod name is the string representation of the certificate Subject, as described in {{-dn-string-rep}}.
For example: "spdm:C=CA,O=ACME,OU=Widget,CN=0123456789".

## PCIe Legacy Device Claims {#pcie-legacy-device}

A definition is provided for a device claims-set for PCIe legacy devices that do not implement the necessary extensions to attest to their provenance and configuration.
This makes it possible to keep using current assets as secured ones are being provisioned.
This legacy device claims-set simply mirrors the type 0/1 common registers of the PCIe configuration space, mandating only that the vendor and device identification code be provided.
Other fields of the configuration space header may optionally be included should they add value.
A binary format of the PCIe configuration space is made available for processing by existing PCIe configuration space tools.
Implementers may optionally choose to include both text and binary versions should there be a use case to support this representation.

~~~ cddl
{::include-fold cddl/pcie-legacy-claims.cddl}
~~~

### Submodule Naming {#pcie-legacy-submod-name}

The namespace used for legacy PCIe submodules is "legacy-pcie".

The name is any arbitrary string chosen by the implementation.
For example, "legacy-pcie:0000:01:02.0" where "0000" is the domain, "01" the PCI bus id, "02" the device on the bus and "0" the device function.

# DAT EAT Profile {#profile}

## Encoding

A DAT is encoded in CBOR {{-cbor}}.
The CBOR representation of a DAT MUST be "valid" according to the definition in {{Section 1.2 of -cbor}}.
Only definite-length strings, arrays, and maps are allowed.
Since a DAT emitter may be found in a constrained environment, it may not be able to emit CBOR preferred serializations ({{Section 4.1 of -cbor}}).
Therefore, the Verifier MUST be a variation-tolerant CBOR decoder.

## Cryptographic Protection

Cryptographic protection can be obtained by wrapping the `dat` claims-set in a COSE Web Token (CWT) {{-cwt}}.
In this case, the signature structure MUST be a tagged (18) COSE_Sign1.
Alternatively, a DAT can be part of a Conceptual Message Wrapper (CMW) {{-cmw}} collection.
In this case, the DAT claims-set can be a UCCS {{-uccs}} and the protection is provided by the signed CMW.

The flexibility provided by the COSE {{-cose}} format should be sufficient to adapt to the level of cryptographic agility required for specific use cases.
It is RECOMMENDED that commonly adopted algorithms, such as those discussed in {{-cose-algs}}, are used.
While receivers are expected to accept a wide range of algorithms, Attesters will produce DAT using only one such algorithm.

## Use with Conceptual Message Wrappers

When used in a CMW, the collector will wrap the serialised COSE_Sign1 or UCCS with the appropriate media type or CoAP Content-Format defined in {{-eat-media-types}}.

## Freshness Model

DAT supports the freshness models for attestation Evidence based on nonces and epoch IDs (see {{Section 10.2 and Section 10.3 of -rats-arch}}) using the `eat_nonce` claim to convey the nonce or epoch ID supplied by the Verifier.
No further assumptions are made about the specific remote attestation protocol.

Note that the use of epoch IDs is subject by the type restrictions imposed by the `eat_nonce` syntax.
For use in DAT, the epoch ID MUST be encodable as an opaque binary string of between 8 and 64 octets; an Epoclet can be used for this purpose (see {{-epoch-markers}}).

## Synopsis

{{tbl-profile}} presents a concise view of the requirements described in the preceding sections.

| Issue | Profile Definition |
| CBOR/JSON | CBOR MUST be used  |
| CBOR Encoding | Definite length maps and arrays MUST be used |
| CBOR Encoding | Definite length strings MUST be used |
| CBOR Serialization | Variant serialization MAY be used |
| COSE Protection | COSE_Sign1 MUST be used (directly or via CMW) |
| Algorithms | {{-cose-algs}} SHOULD be used |
| Detached EAT Bundle Usage | Detached EAT bundles MUST NOT be sent |
| Verification Key Identification | Any identification method listed in {{Appendix F.1 of -rats-eat}} |
| Freshness | nonce or epoch ID based ({{Section 10.2 and Section 10.3 of -rats-arch}}) |
| Claims | Those defined in {{dat-claims}}. As per general EAT rules, the receiver MUST NOT error out on claims it does not understand. |
{: #tbl-profile title="DAT Profile Synopsis"}

# Extending the DAT Framework {#extend}

An extension to the DAT framework that introduces support for a new bus technology MUST provide the following information in a public document (e.g., an Internet-Draft):

* A precise definition of the new claims-set,
* A naming convention for the `submod` map entry,
* The registration of any new claims with IANA.

## Claims-set Definition

The new claims-set MUST be specified clearly and unambiguously, ideally using CDDL, with a separate prose description of each claim.
The claims-set MUST include a suitable `eat_profile` value.

See {{spdm-claims}} for the blueprint.

## Naming Conventions for the `submod` Key

A new claims-set MUST define a suitable naming convention for the `submod` keys associated with it.
When creating this convention, ensure that it does not clash with any existing ones.

See {{spdm-submod-name}} for the blueprint.

The concrete name derived by the naming scheme is anticipated to be used to map the Attester to a corresponding CoRIM {{-corim}} environment map.
This map can then be used to locate matching Reference Values and Endorsements during the appraisal process.
This type of "evidence transformation" is outside the scope of this document and will be covered by an associated CoRIM profile.

## Claims Registrations

A new claims-set can reuse any number of already registered claims.
If the claims-set needs to define new claims to express the desired semantics, and if these claims have generally applicable semantics, they SHOULD be registered with IANA.

See {{iana-claims-regs}} for the blueprint.

# Collated CDDL

~~~ cddl
{::include-fold cddl/da-token-autogen.cddl}
~~~

# Security Considerations

As this specification reuses the EAT specification {{-rats-eat}}, it also reuses the CWT specification {{-cwt}}.
The security and privacy considerations of these specifications therefore apply here too.
In particular, the considerations discussed in {{Sections 9.1 (Claim Trustworthiness), 9.4 (Multiple EAT Consumers) and 9.5 (Detached EAT Bundle Digest Security Considerations) of -rats-eat}} apply fully.

When DAT is an UCCS, the considerations in {{-uccs}} also apply.

PCIe devices are assigned to a TVM via their physical or virtual functions.
The hardware implementation of a device guarantees that functions are mutually exclusive; in other words, operations performed on one function do not affect other functions.
The TDISP standard also guarantees that, while a device function is in the CONFIG_LOCK or RUN state, its characteristics cannot be modified by untrusted parties.
However, faulty hardware or an inadequate implementation of the TDISP specification could be exploited by an attacker for side-channel attacks.

# Privacy Considerations

A DAT can include a great deal of detail about the execution environment associated with the TVM and, therefore, the workload being executed within it.
This can provide insight into the type of computation being carried out by the workload.
It can also enable tracking of a given workload across multiple TVM instances in both the temporal dimension (e.g., across reboots of the hosting hardware) and the spatial dimension (e.g., across migrations between hardware, either within or between data centres).

A DAT is usually one component of a composite evidence payload.
In such cases, multiple Verifiers may be involved in the appraisal process.
The differential encryption considerations discussed in {{Section 9.4 (Multiple EAT Consumers) of -rats-eat}} therefore apply.

# IANA Considerations

## New CWT Claims Registrations {#iana-claims-regs}

IANA is requested to register the following claims in the "CBOR Web Token (CWT) Claims" registry {{IANA.cwt}}.

### SPDM Extracted Measurements Claim

* Claim Name: spdm-extracted-measurements
* Claim Description: SPDM Extracted Measurements
* JWT Claim Name: N/A
* Claim Key: CPA3802
* Claim Value Type(s): map
* Change Controller: IETF
* Specification Document(s): {{spdm-measurements}} of {{&SELF}}

### SPDM Certificates Claim

* Claim Name: spdm-certificates
* Claim Description: SPDM Certificates
* JWT Claim Name: N/A
* Claim Key: CPA3803
* Claim Value Type(s): map
* Change Controller: IETF
* Specification Document(s): {{spdm-certificates}} of {{&SELF}}

### SPDM VCA Claim

* Claim Name: spdm-vca
* Claim Description: SPDM Version, Capabilities and Algorithms
* JWT Claim Name: N/A
* Claim Key: CPA3804
* Claim Value Type(s): bytes
* Change Controller: IETF
* Specification Document(s): {{spdm-raw-claims}} of {{&SELF}}

### PCIe Legacy Device Text Claim

* Claim Name: pcie-legacy-device-text
* Claim Description: PCIe Legacy Device Textual Representation
* JWT Claim Name: N/A
* Claim Key: CPA3805
* Claim Value Type(s): map
* Change Controller: IETF
* Specification Document(s): {{pcie-legacy-device}} of {{&SELF}}

### PCIe Legacy Device Binary Claim

* Claim Name: pcie-legacy-device-binary
* Claim Description: PCIe Legacy Device Binary Representation
* JWT Claim Name: N/A
* Claim Key: CPA3806
* Claim Value Type(s): bytes
* Change Controller: IETF
* Specification Document(s): {{pcie-legacy-device}} of {{&SELF}}

### SPDM Raw Measurements Claim

* Claim Name: spdm-raw-measurements
* Claim Description: SPDM Raw Measurements Transcript
* JWT Claim Name: N/A
* Claim Key: CPA3809
* Claim Value Type(s): bytes
* Change Controller: IETF
* Specification Document(s): {{spdm-raw-claims}} of {{&SELF}}

### SPDM Certificate Chain Claim

* Claim Name: spdm-certificate-chain
* Claim Description: SPDM Raw Certificate Chain
* JWT Claim Name: N/A
* Claim Key: CPA3810
* Claim Value Type(s): bytes
* Change Controller: IETF
* Specification Document(s): {{spdm-raw-claims}} of {{&SELF}}

### SPDM Raw Measurement Extension Log Claim

* Claim Name: spdm-raw-measurement-extension-log
* Claim Description: SPDM Raw Measurement Extension Log
* JWT Claim Name: N/A
* Claim Key: CPA3811
* Claim Value Type(s): bytes
* Change Controller: IETF
* Specification Document(s): {{spdm-raw-claims}} of {{&SELF}}

### TDISP Device Interface Report

* Claim Name: tdisp-device-interface-report
* Claim Description: TDISP Device Interface Report
* JWT Claim Name: N/A
* Claim Key: CPA3808
* Claim Value Type(s): map
* Change Controller: IETF
* Specification Document(s): {{interface-report}} of {{&SELF}}


### SPDM Raw Artefacts Claim

* Claim Name: spdm-raw-artefacts
* Claim Description: SPDM Raw Artefacts
* JWT Claim Name: N/A
* Claim Key: CPA3812
* Claim Value Type(s): map
* Change Controller: IETF
* Specification Document(s): {{spdm-raw-artefacts}} of {{&SELF}}

### SPDM Extracted Measurement Extension Log Claim

* Claim Name: spdm-extracted-measurement-extension-log
* Claim Description: SPDM Extracted Measurement Extension Log
* JWT Claim Name: N/A
* Claim Key: CPA3813
* Claim Value Type(s): bytes
* Change Controller: IETF
* Specification Document(s): {{spdm-extracted-claims}} of {{&SELF}}

--- back

# Examples

~~~ cbor-diag
{::include-fold cddl/examples/1.diag}
~~~

# Example Composite Device

{{fig-ratsd}} shows an example of the composite device described in {{Section 3.3 of -rats-arch}} within a confidential computing environment.
In this setup, a Trusted Virtual Machine (TVM) executes on a Confidential Platform, which provides the confidential computing environment.
One or more devices (e.g., a GPU) are assigned to the TVM.

Within the TVM, a Lead Attester agent, e.g., a userland daemon, can collect Evidence from the Confidential Platform, as well as from all the assigned devices, using the relevant ABI offered by the guest OS kernel.

When a challenger (i.e., a Verifier or a Relying Party) requests Evidence from the TVM, the Lead Attester broadcasts the received nonce to all the sub-Attesters, obtains Evidence from each of them and assembles the composite Evidence using a CMW Collection (Section 3.3 of {{-cmw}}).
It then signs the composite Evidence using its key material as shown in {{fig-ratsd-token}}.

The claims obtained by the assigned devices are repackaged into DAT submods, which are then signed as part of the CMW collection using the Lead Attester key.

~~~ aasvg
                signed-cbor-cmw
               .---------------------.
               |  .---------------.  |
         .------->| Lead Attester |  |
         |     |  | Evidence      |  |
         |     |  '---------------'  |
         |     |                     |
         |     |  .---------------.  |
 nonce --+------->| Platform      |  |
         |     |  | Evidence      |  |  .----------------------------.
         |     |  '---------------'  |  | eat_profile                |
         |     |                     |  | eat_nonce                  |
         |     |  .---------------.  |  | submod {                   |
         '------->| DAT           +- - -+   "spdm:A": { ... }        |
               |  '---------------'  |  |   "legacy-pcie:B": { ... } |
               '---------------------'  |   "spdm:C": { ... }        |
                                        | }                          |
                                        '----------------------------'
~~~
{: #fig-ratsd-token title="Composite Attestation Evidence" align="center" }

The Veraison project's `ratsd` daemon is an example of this behaviour.

# Acknowledgments
{:numbered="false"}

Thank you
Basma El Gaabouri,
Carl Wallace,
Giri Mandyam,
{{{Ionuț Mihalcea}}},
James Bottomley,
Jon Lange,
Lukas Wunner,
Michael Richardson,
Ned Smith,
Roksana Golizadeh Mojarad,
Simon Frost,
Yogesh Deshpande
and
Yousuf Sait
for your comments and suggestions.
