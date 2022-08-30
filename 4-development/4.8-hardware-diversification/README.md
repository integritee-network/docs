# 4.8 Hardware Diversification

As of the time of writing, Intel SGX is the only TEE that allows remote attestation. This dependency on Intel is a single point of failure and as such undesirable for Integritee.

The Integritee team is investigating ways to provide remote attestation for ARM TrustZone and open source TEEs like Keystone.

Diversification would have a positive effect on TEE integrity because a vulnerability in one type of TEE would only affect a fraction of all TEEs. A pretty simple consensus algorithm could ensure integrity even in presence of large-scale attacks exploiting that vulnerability.

However, diversification could have a negative impact on confidentiality. If secrets are provisioned to several types of TEEs, it only takes a single TEE to leak the secret to compromise it for all.



### **Table of Content:**

* [4.8.1 Distributor-Level Remote Attestation](4.8.1.-distributor-level-remote-attestation.md)
* [4.8.2 Concept](4.8.2-concept.md)
* [4.8.3 Roles](4.8.3-roles/)
  * [4.8.3.1 Hardware Manufacturer](4.8.3-roles/4.8.3.1-hardware-manufacture.md)
  * [4.8.3.2 Trusted Software Source](4.8.3-roles/4.8.3.2-trusted-software-source.md)
  * [4.8.3.3 Provision Entity](4.8.3-roles/4.8.3.3-provision-entity.md)
  * [4.8.3.4 Member](4.8.3-roles/4.8.3.4-member.md)
  * [4.8.3.5 Verifier](4.8.3-roles/4.8.3.5-verifier.md)
