![](img/image1.png)

Tessera – Integrated HSM

Project Definition

0.5.0

Integrated HSM represents an evolution in how the industry can scale hardware security modules (HSMs) and modernize key management. It addresses several well-known limitations of centralized HSM architectures, including deployment agility, lack of interoperable and auditable mechanisms for securely transferring keys, and long-standing key management antipatterns.

The Integrated HSM approach introduces workload-attached companion HSMs, which perform cryptographic operations closer to the workload while keeping keys continuously protected within HSM boundaries. The result is stronger security assurances, improved performance for cryptographic operations, and a simpler operational model.

In data centers, traditional HSMs are typically deployed separately from compute resources, requiring cryptographic operations to be performed over the network. This architecture introduces additional latency from network round trips and increases operational dependencies. To mitigate these challenges, enterprises may adopt workarounds that reduce latency or operational dependencies but can inadvertently weaken key protection. Common examples include the following key management antipatterns:

##### 1)      Critical keys generated outside the HSM

HSM-backed keys may be used alongside externally generated keys. For example, when an HSM key acts as a certificate authority and endorses keys created outside the HSM. Those externally generated keys can then operate on behalf of the HSM-backed key while residing outside the HSM security boundary.

##### 2)      Key Export (KE)

Keys stored in HSMs may be marked extractable and released into workload environments. These environments may not maintain the same security-domain properties as the HSM and may not meet the same assurance level (for example, FIPS 140-3 Level 3). This increases the risk of unintended use (due to loss of HSM-enforced key attributes) and potential key exposure (due to weaker protections).

##### 3)      Key Resilience (KR)

To improve resilience and availability, services may minimize operational dependencies by caching keys locally using sealing techniques rather than relying on the availability of centralized HSMs. This can avoid service failures tied to network-based services such as attestation, authentication, or remote HSM access. However, it can result in key-management outside the HSM environment.

##### 4)      Cryptographic latency

To reduce network round-trip latency for centralized HSM operations (for example, sign/verify or encrypt/decrypt), workloads may adopt approaches like externally generated keys or key release (see #1 and #2). These approaches can move keys into weaker isolation mechanisms, reduce assurance of posture, and increase the risk of accidental or malicious misuse.

##### 5)      Key Governance Policy

Operating centrally managed HSMs with application unique partitions and security domains can introduce additional operational overhead. For small and medium-sized services, that overhead can be a barrier to adopting strong key isolation practices within HSMs. Service designs may over-rely on single security domains supporting multiple services, keys used across, multiple services, or falling back to keys managed outside of the HSM during use.

##### 6) Regulatory Challenges

Cloud HSMs today protect cryptographic keys within hardware-rooted boundaries, but keys cannot be securely moved between independently operated HSM environments from different vendors, or across clouds, without either re-generating the key, relying on proprietary transport mechanisms, or accepting a risk of plaintext exposure. This creates compounding problems for enterprises:

* Multi-cloud and disaster recovery architectures that span providers cannot maintain a single, auditable cryptographic lineage for the same key material.
* Regulated customers face difficulty demonstrating "exit-ready" cryptographic controls to regulators under frameworks such as DORA, OSFI, PCI-HSM, and sovereign-cloud requirements.
* Key security attributes, usage policies, and provenance metadata are not preserved in a standardized way when crossing vendor boundaries.

To work around these challenges, enterprises opt to either create keys outside HSMs or export from the HSM environment and into an intermediatory software environment.

These antipatterns are widely observed across the industry and can undermine the fundamental security objectives that HSMs are intended to provide. The Integrated HSM key‑management model addresses these challenges by extending the HSM security domain to system attached HSMs, enabling consistent enforcement of hardware‑rooted protections across distributed environments.

In this model, cryptographic keys do not necessarily need to persist within local HSMs. As in centralized HSM models, encrypted key blobs can be stored in durable storage accessible to the systems.

### Transparent & Integrated HSM

The ecosystem can benefit from an open and transparent key-management model that supports interoperability between multi-vendor centralized HSMs and workload-attached system-integrated HSMs, without requiring extracting keys (CKA\_EXTRACTABLE) from the bounds of a HSM.

Tessera is a fully open-source, software, firmware, and silicon integrated HSM design that provides transparency, consistency and establishes a common foundation for interoperability across HSM solutions. Like Caliptra, Tessera is available for industry-wide collaboration.

##### Tessera Work Products:

* Published technical specifications through OCP for open architecture and protocols, the Integrated HSM design, and design requirements.
* Advance HSM protocols for PQC resilience.
* A reference implementation (software, firmware, silicon and hardware) contributed by participating members and maintained in a public github repository
* Initial hardware availability will be made available through OCP and manufacturing channel partners. Enabling multi-vendor availability through OCP and manufacturing channel partners.
* Chip capabilities and architecture, fully open-source silicon implementation of Integrated HSM as demonstrated with Caliptra.