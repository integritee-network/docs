# 2.7 Privacy Technology: Trusted Execution Environments

A Trusted Execution Environment (TEE) is an isolated environment that uses both special-purpose hardware and software to protect data. In general, TEEs provide a “trusted environment” inside which computations and analysis can be run while remaining invisible to any other process on the processor, the operating system, or any other privileged access.

Moreover, the manufacturer can authenticate each TEE and provide remote attestation to a user to confirm that her untampered program is actually running on a genuine TEE, even if the machine is physically located in an off-site data center.

Assuming we trust TEE manufacturers’ integrity and design competence, TEEs allow us to execute any state update without sharing our data with the blockchain validator or other users. Private token transfers, private smart contracts, and private state channels thus become possible, with little computational effort.



### **Table of Content:**

* [2.7.1 What is a TEE?](2.7.1-what-is-a-tee.md)
* [2.7.2 What is remote attestation?](2.7.2-what-is-remote-attestation.md)
* [2.7.3 How we use the technology to make confidential computation publicly verifiable](2.7.3-how-we-use-the-technology-to-make-confidential-computation-publicly-verifiable.md)
* [2.7.4 Intel SGX](2.7.4.-intel-sgx/)
  * [2.7.4.1 Why SGX and not other technologies](2.7.4.-intel-sgx/2.7.4.1-why-sgx-and-not-other-technologies.md)
  * [2.7.4.2 Intel SGX Security Design](2.7.4.-intel-sgx/2.7.4.2-intel-sgx-security-design.md)
