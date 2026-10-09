![](img/image1.png)

Tessera – Masking Key Transfer Protocol

0.5.0









# Acknowledgements

The Contributors of this Specification:

|  |  |
| --- | --- |
| **Name** | **Company** |
| Bryan Kelly | Microsoft |
| Craig Barner | Marvell |
| Mohan Kumar | Oracle |
| Vishal Soni | Microsoft |









# Summary

This architectural specification combines two related Tessera Integrated HSM security operations involving authenticated Hybrid Public Key Encryption (HPKE) contexts. The first operation transfers a Masking Key between authenticated HSM partitions so that both partitions can consume the same Masked Key Objects. The second operation transports one selected key from an HSM partition to a conformant secure hardware device, or secure device interface without transferring the source Masking Key. Both operations layer certificate validation, runtime attestation, independently issued audit evidence, policy evaluation, key attributes, explicit bidirectional key confirmation and auditable operations.

Masking Key transfer deliberately expands a Security Domain (SD) across HSM devices by cloning HSM portions across devices. After successful installation, the source and destination partitions become members of the same authorized HSM partition ring and share access to objects protected by the transferred Masking Key. Individual key transport has a narrower scope but transfers designated keys from HSM partitions to conformant devices. This operation does not expand the source Masking Key domain and is distinct from traditional export or key release.

The source documents establish the ownership hierarchy, partition cloning model, Masking Key purpose, multi-vendor transfer policy, interface binding, attestation expectations, and audit report. This draft preserves those foundations. It introduces an explicit three-level certificate validation model, authenticated HPKE context binding, algorithm-neutral key-confirmation constructions, a unified Protected Transfer Object, CWT and COSE authorization for MSI tokens, detailed state machines, and consolidated failure and threat analysis as proposed architectural extensions.

# Purpose

The specification explains how HSM authorizes and protects two forms of key movement: Masking Key transfer among authorized HSM partitions, and the transport of selected keys to authorized secure hardware devices, or secure device interfaces. It describes the certification authentication, evidence, policies, session bindings and token bindings, state transitions, and audit records that allow the participating security boundaries to agree on what can be moved and where it can be moved.

# Scope

The architecture covers device Root certifications, device owner certificates, partition ownership, runtime evidence, and authenticated HPKE establishment with key confirmation. Masking Key version transfer, individual key transport, KOTA authorization, optional CWT MSI authorization, key-attribute preservation, destination installation, cleanup, rotation, revocation, recovery, and interoperability. A coordinating trusted execution environment or service and an external attestation verifier can participate when deployment architecture uses them, but neither replaces endpoint validation.

# Architectural Principles

Authorization is an intersection, not a single credential check. Device authenticity, device ownership, partition ownership, measured state, audit reports, key attributes, token authorization when present, and transfer policy all contribute to how keys are moved across between HSM partitions and to designated devices. A successful check at one layer does not compensate for a failed or mismatched check at another layer.

HPKE provides confidentiality and context-derived secrets, while the Tessera Transport protocol provides 3 level certificate validation, attestation, authorization, explicit key confirmation, audit verification, policy enforcement, anti-rollback checks, and attribute verification. The protocol transfers key material only after both endpoints verify that they derived the same authenticated context for the same identities, and met all conditions to satisfy the transfer policy.

The destination preserves the security-relevant attributes received from the source. It can impose tighter local restrictions, but it does not weaken sensitivity, usage, extractability, destination, interface, lifecycle, token, or policy controls carried in the Protected Transfer Object.

# Terminology and Acronyms

|  |  |  |
| --- | --- | --- |
| Term | Meaning | Mapping |
| HPKE | Hybrid Public Key Encryption used to establish an authenticated transfer context and session-derived secrets. | Authentication keys and evidence bind to HSM identities. |
| Explicit key confirmation | Bidirectional proof that both endpoints derived the same context for the intended transcript, identities, roles, policy, and purpose. | Uses context derived from validated HSM evidence. |
| Masking Key | Partition-held key that protects Masked Key Objects stored outside the HSM boundary. | Partition attestation reports version and state. |
| Transfer Masking Key | Temporary session key that protects a Masking Key during final partition cloning phase. | Derived from HPKE and distinct from the Masking Key. |
| Transport Key | Temporary session key that protects an individual key and attributes during kety transport. | Derived from HPKE and distinct from the Transfer Masking Key. Targets individual key, with transfer attributes. |
| Masked Key Object | Externally stored object containing a key protected by a partition Masking Key and associated metadata. | Consumed inside a partition holding the applicable Masking Key. |
| DRAC | Device Root Authentication Certificate. | LDevID typically operator-endorsed |
| AOAC | Tessera Owner Authentication Certificate; the end-entity owner endorsement is distinct from its trust anchor. | Target level is FMC Alias using OCP DOT |
| POTA | Partition Owner Trust Anchor, the partition ownership authority or certificate authority. | Partition ownership authority. |
| Partition Identity Certificate | POTA-issued certificate over a partition identity key; replaces source use of POAC for the end-entity certificate. | Partition-level owner endorsement. |
| KOTA | Key Owner Trust Anchor governing authorization to transport a key to a device or interface. | Policy trust anchor bound to the requesting interface identity. |
| Interface identity | Cryptographic identity for a participating interface, including binding to the parent device and KOTA. | Issued from Caliptra device evidence and presented in TDISP TDI Interface Report. Authorized (signed) by bound TEE with KOTA. |
| Runtime attestation | Fresh evidence describing current measured device, partition, and interface state. | Caliptra measurements, SVN, Caliptra CWT audit, device security state, owner and partition info |
| Caliptra-endorsed audit evidence | Independent audit or conformance result of Caliptra Subsystem integration, correlated with make/model runtime state. | Binds an approved state to the current measured state. |
| CWT | CBOR Web Token used for MSI authorization. | Compact token suitable for firmware processing. MSI token offers key use grant. |
| COSE | CBOR Object Signing and Encryption structures used to protect the CWT. | Carries the MSI token signature and optional confirmation material. |
| MSI token | CWT authorization token issued for a managed identity or service identity and bound to the request or transcript for key use authorization. | No JWT alternative is defined. |
| Key-transfer policy | Policy combining destination, evidence, ownership, KOTA, token, lifecycle, and cryptographic constraints. | References accepted roots, measurements, verifiers, and signers. |
| Key attributes | Identity, type, usage, security, transfer, destination, token, and lifecycle properties protected with the key. | Reflected in key attestation and the Protected Transfer Object. |

# Partition Masking Key Transfer

Partition Masking Key Transfer is the protocol that clones like HSM partitions such that they share the same Masking Key. The HPKE based protocol performs 3 levels of certificate verification to establish hardware authenticity, Tessera owner, and partition owner.

# 3 Tier Certificate Authentication

## Level 1: DRAC

Caliptra IDEV establishes device manufacturer authenticity, but partition cloning across HSMs from different manufactures requires a root identity rooted to the physical device operators trust anchor. The operator-endorsed Caliptra LDevID identifies the device under an operator-controlled domain and forms the basis for DRAC. In the cross manufacturer HSM partition cloning protocol, the source and target HSM retrieves the Caliptra certificate chain and validates the LDev certificate was issued (signed) from the DRAC Trust Anchor (TA).

The DRAC is typically provisioned during HSM onboarding by a data center operator. DRAC is provisioned after the LDEV field entropy is programmed. The LDEV CSR is signed by the DRAC Trusted Anchor and issued certificate issued to the device.

IDevID and LDevID remain distinct. IDevID provides the manufacturer-rooted device identity. LDevID expresses operator endorsement and can become the basis of DRAC after validation and registration. The architecture does not treat the two certificates as interchangeable. IDevID remains the manufacturer authentication certificate.

## Level 2: AOAC

AOAC establishes the owner domain for an HSM. This domain covers the entire server system and can be a dedicated service or consumer. OCP Device Ownership Transfer creates an owner-controlled device. The AOAC trust anchor endorsed the FMC Alias. In device identity chaining, the FMC Alias chains to IDEV in SPDM slot 1, SPDM slot 2 binds DRAC, SPDM lot 3 binds AOAC. The Caliptra, the FMC Alias, reflects the AOAC ownership both key derivation of FMC Alias and device measurement.

In the certificate verification stage of the protocol handshake, AOAC verification permits only AOAC device transfers. The AOAC trust anchor is the issuing authority; AOAC is the end-entity owner certificate or endorsement mapped to the FMC Alias.

## Level 3: POAC

The partition generates a unique partition identity key inside its security boundary. The device provides attestation or a device-level endorsement for the public key. The partition creates a certificate signing request, and the provisioning authority compares the CSR public key with the partition-attested public key.

The POTA then issues the Partition Owner Authentication Certificate. The partition installs the POTA chain and its certificate. Partition attestation binds configuration measurements, sealing service, ownership, and the partition authentication pub key.

In the Masking Key Transfer protocol, POAC must be endorsed by one of the same POTAs.

|  |  |  |  |  |  |  |
| --- | --- | --- | --- | --- | --- | --- |
| Validation level | Credential | Trust anchor | Identity established | State binding | Authorization enabled | Failure outcome |
| Level 1 | DRAC | Manufacturer root or accepted operator device root | Authentic physical device | Hardware instance identity.  Caliptra conformance endorsement. | Device authenticity. | Reject before owner or partition evaluation |
| Level 2 | AOAC | Service or system Operator trust anchor | Device-owner domain and FMC Alias | Ownership-transfer state and measured firmware | Same-owner or accepted-owner transfer | Reject cross-owner transfer |
| Level 3 | Partition Identity Certificate | Partition Owner Trust Anchor | Partition and partition owner | Configuration, measurements, Masking Key state | Cloned partition ring membership | Reject Masking Key sharing or partition-scoped access |

# Attestation

Runtime attestation describes the device's current measured state. This attestation depends on the security of the HSM Root of Trust. In Tessera, the root of trust is Caliptra Subsystem.

The attestation evidence set in session establishment for the Masking Key Transfer Protocol includes POAC, AOAC, DRAC, firmware measurements, SVN, partition configuration state, and sealing service.

The protocol rejects missing, expired, stale, untrusted, replayed, cross-device, cross-interface, attestation-inconsistent, or policy-incompatible evidence.

# HPKE Protocol

The HPKE design includes post-quantum and hybrid post-quantum key establishment. A PQC-only profile uses an approved post-quantum KEM within the selected HPKE construction. A hybrid profile combines a classical KEM contribution with a post-quantum KEM contribution so that compromise of either component alone does not reveal the derived session secrets. Negotiation authenticates the complete suite choice and binds it into the transcript, preventing an intermediary from silently removing the PQC component or selecting a weaker profile. Both PQC-only and hybrid PQC sessions use explicit bidirectional key confirmation.

## HPKE Context Binding

The HPKE information and transcript context bind the protocol; negotiated HPKE suite; source and destination 3 level certificate chain validation (DRAC, AOAC, and POAC).

# Key Confirmation

HPKE encryption alone does not demonstrate that both participants derived the same context for the intended identities, roles, policy, evidence, suite, and transfer purpose. Explicit confirmation occurs in both directions before source transfers the Masking Key. The confirmation detects unknown-key-share behavior, endpoint confusion, role confusion, transcript substitution, stale session state, and mismatched certificate or attestation evidence.

# Masking Key Transfer

The workflow involves DRAC, AOAC, POAC chains are validated independently; retrieves fresh partition attestations; compares configuration and policy; verifies POTA compatibility, firmware measurements, SVN, and Masking Key version; establishes authenticated PQC or hybrid PQC HPKE; and completes bidirectional key confirmation. The source protects the Masking Key under the temporary Transfer Masking Key together with version, and associated state.

The destination and source do mutual authentication and confirm committed versions. After the Mask Key Transfer, both endpoints erase HPKE and Transfer Masking Key metadata, and both record the event. The transfer expands the Masking Key security domain: both partitions become members of the same authorized ring and can consume the same Masked Key Objects.

DRAC, AOAC, or POTA mismatch; partition-configuration mismatch; attestation failure; SVN rollback; Masking Key version rollback; confirmation failure; prevents commit. Any partially staged context remain inaccessible and are removed during cleanup. Retry begins with fresh nonces, evidence, policy evaluation, and a new HPKE context.

# Key Transfer

key transport allows a key to be transferred to a pre-defined device or device interface. Although the Key Transfer uses HPKE, the flow is distinct from Masking Key Transfer. Key transfer operates at the secure device and key level, whereas the Masking Key Transfer protocol operates at the HSM and Partition level. The Key Transfer HPKE allows for 3, 2 and single level certificate verification flows depending on the key policy.

After authorization, the source and target establish an ephemeral Transport Key from a PQC or hybrid PQC authenticated HPKE context and complete bidirectional confirmation. The source HSM decrypts the selected key internally from its Masked Key Object, creates a Protected Transfer Object containing the key and security attributes, and encrypts that object under the Transport Key.

The target device decrypts and validates the object inside its Caliptra security boundary, applies all received restrictions, rewraps the key with its local Masking Key or equivalent protection key, commits the protected key atomically, and returns an authenticated installation acknowledgement. Both endpoints erase temporary plaintext and session secrets. This operation does not expand the source Masking Key domain.

Traditional export or key release commonly exposes an extractable representation outside the originating HSM and can lose hardware-enforced attributes. This architecture instead transports a protected object directly between authenticated hardware boundaries, verifies destination state and authorization before source disclosure, and preserves key attributes through installation.

# KOTA Interface Authorization

The authorization decision combines DRAC, AOAC, and KOTA, runtime attestation, Caliptra conformance audit evidence, key attributes, and transfer policy. The destination device interface must prove position of a KOAC, which is a KOTA endorsed certificate.

The source HSM transferring the target key verifies interface authenticity, parent-device binding, expected DRAC, AOAC domains, the expected KOTA binding, runtime measurements, freshness, key-attribute permission, and destination security properties.

While the default KOAC is a KOTA endorsed certificate, Tessera also supports KOAC as a KOTA signed CWT MSI Token. This flow supports token binding for keys.

The key transfer policy in the source HSM partition determines the certificate validation requirements, and whether three, two or single certificate validation is required. Effectively restricting the target key to single security domain or cross domain transfer.

Keys by default are non-transferable. A transfer policy with specified KOTA, validation depth (requiring DRAC, AOAC) must be assigned to the key by the Crypto Officer before the key can be considered transferable.

# Key Attributes and Transfer Policy

The attribute model classifies properties as identity, cryptographic type, usage, security, transfer policy, destination binding, token authorization, and lifecycle. The Protected Transfer Object carries every security-relevant property. The destination preserves restrictions and can add tighter local controls.

|  |  |  |
| --- | --- | --- |
| Class | Attributes | Architectural effect |
| Identity | Key identifier, source identity, ownership references | Correlates the key with its origin and policy domain. |
| Cryptographic type | Key type and algorithm family | Constrains supported installation and use. |
| Usage | Encrypt, decrypt, sign, verify, wrap, unwrap, token-authorized use | Limits operations after installation. |
| Security | Sensitivity, extractability, non-extractability enforcement | Prevents attribute downgrade and plaintext export. |
| Transfer policy | Transfer classification and type; destination types; DRAC, AOAC, POTA, KOTA, verifier, audit signer, SVN, and measurement constraints | Defines eligible destinations and evidence. |
| Destination binding | Destination bindings (DRAT, AOTA, KOTA) | Prevents reassignment. |
| Token authorization | Accepted CWT signers and KOTA associations | Constrains MSI token authorization. |
| Lifecycle | Validity, expiration, refresh, policy version, revocation, anti-rollback | Controls time and update behavior. |

# Audit Authorization

The measurements used in the Key Transfer Protocol are endorsed by the Caliptra Subsystem Root of Trust. Protection of DRAC, AOAC, KOAC and endorsement of measurements depend on the Caliptra. To ensure proper integration of Caliptra into the target device, the Caliptra Subsystem root of trust mush have undergone conformance testing by an OCP S.A.F.E auditor. In the endorsed audit evidence describes an independently evaluated approval of conformance result

Through the OCP SAFE conformance audit, and review of RTL integration, the auditor issues a CWT audit endorsement. This endorsement is per product and must patch the model and device of the IDEV.

# KOAC or KOA-Token Authorization

The KOTA architecture allows the choice between CWT MSI tokens or x509 certificates. The HSM decodes the CWT, validates the COSE signature, identifies the issuer and signer, interprets the subject, checks audience, issued-at, not-before, expiration, token identifier, replay state, and any proof-of-possession or confirmation claims, and binds the token to the request or HPKE transcript. Processing uses a minimal CBOR and COSE subset suitable for firmware.

# Protocol Messages

|  |  |  |  |
| --- | --- | --- | --- |
| Message / producer -> consumer | Purpose and sensitive fields | Transcript and protection | Replay and validation |
| Transfer Initiation / requester -> source | Purpose, key ID, destination and policy reference | Full message; authenticated channel | Caller identity, eligibility, request nonce |
| Capability Negotiation / both -> peer | Select PQC or hybrid PQC suite | Full message; integrity protected | Version, policy, anti-downgrade |
| Challenge / both -> peer | Fresh nonce and session reference | Full message; integrity protected | Nonce uniqueness and active state |
| Evidence Bundle / endpoint or verifier -> peer | Chains, attestation, audit artifact, token | Evidence digests bound; token confidentiality by policy | Syntax, trust, freshness, endpoint correlation |
| HPKE Encapsulation / source -> target | Creates authenticated context | Full message; HPKE processing | Suite, role, and endpoint acceptance |
| Key Confirmation / both -> peer | Confirmation value | Full value and role; ExporterSecret-derived MAC | Expected identity, role, transcript, purpose |
| Protected Transfer Object / source -> target | Key and attributes | Object digest; encrypted and authenticated | Confirmation and authorization complete |
| Installation Acknowledgement / target -> source | Object digest, installed version, result | Full message; context-derived authentication | Atomic commit complete |
| Error Response / detector -> peer | Bounded failure category | Authenticated after context exists | Caller authentication and session stage |
| Session Closure / both -> peer | Session reference, counters, cleanup result | Full message; authenticated | Terminal state and replay record |

Security-sensitive fields are disclosed only at the stage where the consumer has enough authenticated context to process them. Error responses to unauthenticated callers use coarse categories and omit certificate, evidence, policy, and key diagnostics.

# Error Handling

|  |  |  |  |
| --- | --- | --- | --- |
| Failure | Detection and transition | Caller and audit | Cleanup and retry |
| Invalid DRAC | Device validation; Rejection | Coarse device-authentication failure; certificate-validation event | Erase context; retry after credential change |
| Invalid AOAC | Owner validation; Rejection | Coarse owner-domain failure; certificate-validation event | Erase context; retry after owner-chain correction |
| Invalid partition certificate or POTA mismatch | Partition validation; Rejection | Coarse partition-authorization failure; chain digest event | Erase context; reprovision or select compatible partition |
| Missing or mismatched KOTA | Interface policy; Rejection | Coarse destination-authorization failure; KOTA use event | Erase context; retry after policy or identity change |
| Invalid interface identity | Interface validation; Rejection | Coarse interface failure; identity digest event | Erase context; re-attest interface |
| Invalid CWT or unauthorized signer | Token validation; Rejection | Coarse token failure; signer and token-ID digest event | Erase token copy and context; obtain new token |
| Signer-to-POTA mismatch | Token-policy correlation; Rejection | Coarse authorization failure; association event | Erase context; correct administrative association |
| Expired or replayed token | Time or replay check; Rejection | Coarse token freshness failure; token-ID event | Mark replay state; obtain fresh token |
| Invalid audit evidence | Audit validation; Rejection | Coarse evidence failure; artifact digest event | Erase context; obtain accepted artifact |
| Invalid runtime attestation or stale evidence | Attestation validation; Rejection | Coarse state failure; evidence digest event | Erase context; collect fresh evidence |
| SVN rollback | Attestation or policy; Rejection | Coarse rollback failure; SVN event | Erase context; update destination |
| Policy mismatch | Policy evaluation; Rejection | Coarse policy failure; policy-version event | Erase context; retry after authorized policy change |
| Attribute downgrade | Object validation; Rejection | Coarse object failure; attribute digest event | Erase plaintext and staged object; no automatic retry |
| Key-confirmation failure | Confirmation; Rejection | Coarse session failure; transcript digest event | Erase all HPKE secrets; restart full session |
| Masking Key version rollback | Installation check; Rejection | Coarse rollback failure; version event | Erase staged key; select newer version |
| Destination installation failure | Installation; Rejection | Coarse installation failure; object digest event | Rollback staged state; retry with new session |
| Interrupted transfer | Any active state; Rejection | Coarse interruption; last-state event | Timeout, erase session, remove staged state |
| Cleanup failure | Cleanup; quarantined rejection | Coarse internal failure; high-priority cleanup event | Block partition or interface until verified cleanup |
| DRAC mismatch | Peer correlation; Rejection | Coarse identity mismatch; both digests recorded | Erase context; new peer discovery |
| AOAC mismatch | Peer correlation; Rejection | Coarse owner mismatch; both digests recorded | Erase context; use accepted owner domain |
| Partition-configuration mismatch | Policy comparison; Rejection | Coarse configuration failure; config digests recorded | Erase context; align authorized configuration |
| Attestation/audit replay | Freshness correlation; Rejection | Coarse replay failure; nonce and digest event | Invalidate evidence and session; collect fresh state |

# Audit Model

Audit events cover DRAC provisioning, rotation, replacement, and removal; AOAC provisioning and rotation; POTA registration and removal; partition identity provisioning; KOTA policy creation, use, update, and removal; CWT signer registration; signer-to-POTA association changes; audit-signer policy changes; HPKE initiation; certificate, attestation, audit-evidence, CWT, and key-confirmation results; Masking Key transfer; individual key transport; destination installation; rejection; cleanup; and session completion.

Records contain identifiers, certificate and evidence digests, policy versions, event counters, timestamps, result categories, session purpose, suite profile, and endpoint roles. Records exclude plaintext keys, Masking Keys, Transfer Masking Keys, Transport Keys, HPKE exporter secrets, complete bearer tokens, sensitive seed material, and unprotected Masked Key Objects. Caliptra endorsement or cryptographic anchoring ties relevant audit digests and monotonic state to the device identity and measured firmware state.

# Security Analysis

|  |  |
| --- | --- |
| Threat | Controls |
| Endpoint impersonation and certificate substitution | Three-level validation, key correlation, attestation, transcript certificate digests |
| Unknown-key-share and endpoint or role confusion | Authenticated HPKE, bidirectional confirmation, role and purpose binding |
| Replay and stale state | Nonces, counters, evidence freshness, token-ID tracking, transcript binding |
| Downgrade and PQC stripping | Authenticated negotiation, policy-bound PQC or hybrid PQC profile, suite in transcript |
| Cross-owner and cross-partition disclosure | AOAC, POTA, Partition Identity Certificate, configuration and policy comparison |
| Interface reassignment and confused deputy | Parent-device binding, interface identity, KOTA, request and audience binding |
| Attribute stripping | Authenticated object, destination preservation, atomic validation |
| Token theft and unauthorized signer | COSE, accepted-signer attributes, confirmation claim when present, POTA correlation |
| Stale audit or substituted attestation | Independent validation, digest correlation, freshness and nonce checks |
| Firmware or Masking Key rollback | Minimum SVN, measured-state policy, version lineage, anti-rollback state |
| Secret-erasure failure and domain expansion | Cleanup state, quarantine, explicit purpose, ring-membership audit |