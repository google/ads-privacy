# Server trust model techniques

## Background
A part of Chrome’s proposed [FLEDGE API](https://github.com/WICG/turtledove/blob/master/FLEDGE.md) that is different to the [original TURTLEDOVE API](https://github.com/WICG/turtledove) is the addition of a call to a server to get additional signals for an Interest Group.  FLEDGE allows the server to see a limited amount of data: the domain and Interest Group keys.

One topic that hasn’t been discussed yet is what sort of trust model should be applied to this sort of server.  This explainer goes through some options for enforcing trust models in servers without advocating for any one of them.

These techniques have applicability beyond TURTLEDOVE and may also be useful for aggregated reporting, budgeting, pacing, and other use cases.

### What is a trust model?

Here a trust model is defined as:
1. What user information is sent to a server by the browser.
1. What a server is allowed to do with that user information.

## Trust model techniques

These techniques can be used individually to increase trust in a server or combined.

### Limited data

The simplest way to enforce that a server behaves as expected is to limit what is sent to it.

For example, if a server does not see contextual data about an ad request then it cannot possibly leak that data or learn anything from it. However, limited data availability to the server may limit the functionality and business value of the server.

### Open Source codebase and trusted deployment

One way to increase confidence that a server is doing what it is supposed to do is to have the code that it’s running be publicly available.  [See here for Open Source background.](https://opensource.com/resources/what-open-source)

By itself this does not provide much in the way of guarantees because whoever is running the server could easily modify the code.  To that end, the build and release process needs to be able to show that what running is, in fact, a known version of the Open Source code.

As Open Source code is public this also means that proprietary algorithms and information cannot be included; however the Open Source approach can be combined with sandboxing to allow for the execution of proprietary logic: an open-sourced server can run custom code in a sandboxed environment.

### Server governance

The entity that controls and runs the server will have considerable power over the servers.

If the entity has very strong incentives to uphold user privacy, for example, if it is a corporation whose public image is built on that, then the likelihood of the data being misused is lower.

### Auditing

Another way to ensure that a server is behaving as it claims is to have a trusted third-party entity verify the claims.

The effort involved in auditing will differ depending on the complexity of the server.  One way to make the auditing task far simpler is to pair this technique with an Open Source codebase - that way the auditors can simply audit the build and deployment process to ensure that a known Open Source code version is running.

The complexity here is in agreement on a trusted third-party and the logistics to set up auditing checks.  Proprietary information would not necessarily need to be	 made public.

Auditing increases in difficulty as the complexity of the server in question increases.  Therefore the simpler the server, and what it’s allowed to do, the easier it’ll be to audit.

### Policy-based sandboxes

Sandboxes are designed to put limits on what code under execution can do.  We could make use of them in these servers to control what can be done with user data, for example, by not allowing the code to connect to the network or have any other side effects.

Sandboxes rely on the owner of the server hardware being trustworthy because that entity will still be able to see what’s happening inside the sandbox, however they are less nascent techniques and have well understood overheads compared to the things below.  The APIs for what sandboxes allow, in terms of things like I/O and persistence, are needed to restrict side effects.

Examples of environments that sandboxes can be built on are:

* Javascript engines, e.g. [V8](https://v8.dev)
* [WebAssembly runtime](https://webassembly.org/)
* [gVisor](https://gvisor.dev/)
* [Firecracker](https://firecracker-microvm.github.io/)

By default, these environments do not provide any I/O, persistence or other side-effects to the sandboxed code, and any such side effects can only be provided explicitly by the embedding code.

### Trusted Execution Environments (TEEs)

Confidential computing is a set of new technologies that allows extra layer of protection of the user or customer data achieved by running code in hardware-rooted trusted execution environments (TEEs). It is done by utilizing hardware-based security innovations by the CPU vendors to isolate code from the host OS, the hypervisor, drivers, and other privileged applications and users.  In particular, the following protections are offered:

* Transparent, hardware-rooted encryption of in-flight data, i.e. in RAM.
* Protection of data that’s inside the TEE from the operating system or VM host.
* Memory integrity protection.

Remote attestation, to cryptographically guarantee to a caller (e.g. browser in our case) that certain, trusted code is running within the TEE, along with ability to seal input data to only be accessible to that TEE is also possible.

Code running within a TEE sees user data in cleartext and can do anything with the user data permissible by the operation parameters (e.g. it may have a network connection but may not be permitted to write data to local disk). In terms of constraints on the data flows, this is similar to keeping the user data and business logic in the browser. In both cases user data stays within the defined boundary, and processes that access it can do so only in a controlled fashion. TEEs need to be combined with sandboxing in order to place limits on what the code running inside the TEE is allowed to do with user data in terms of persisting it or sending it over the network.

In principle, modern hardware-supported TEEs can run arbitrary computations expressed in higher-level languages.

Confidential Computing shows significant promise but does add an overhead to code execution, is a relatively new technology, and there are ongoing improvements to strengthen security properties of TEEs, including mitigations of potential sophisticated side-channel attacks.

Links:

* [Intel SGX](https://www.intel.com/content/www/us/en/architecture-and-technology/software-guard-extensions.html)
* [AMD SEV-SNP](https://www.amd.com/system/files/TechDocs/SEV-SNP-strengthening-vm-isolation-with-integrity-protection-and-more.pdf)
* [Intel Trust Domain Extensions (TDX)](https://software.intel.com/content/www/us/en/develop/articles/intel-trust-domain-extensions.html)
* [Asylo](https://asylo.dev/): a framework that allows to build applications in C++ portable to different underlying enclave platforms, with remote attestation support.
* [Project Oak](https://github.com/project-oak/oak): a specification and a reference implementation for the secure transfer, storage and processing of data.

### Secure Multi-Party Computation (MPC)

[This technique](https://en.wikipedia.org/wiki/Secure_multi-party_computation) uses cryptography to create a system where the servers processing data can do so while the data remains encrypted and is opaque to the servers. User data remains secure even if some (but not all) of the servers collude with each other or with outside entities attempting to decrypt the data.

MPC can guarantee end-to-end user data encryption, and no party except the user’s browser ever sees user data in cleartext, assuming that at least one of the parties running the secure MPC servers remains honest.

In many MPC algorithms, if one or some parties in the secure MPC setting intentionally deviate from the crypto protocol, e.g. injecting erroneous/malicious intermediate results, the overall result will be detectably bad by other parties.

The same Secure MPC design protecting user data can also protect business confidential information needed during ads serving, e.g. Secure MPC algorithms can frequently update remaining budgets and shutdown ads delivery as soon as the budget runs out, without leaking business confidential budget information to any party, including the servers and server operators involved in the Secure MPC operation .
  
MPC can be divided into two categories:

1. Special purpose: the system is designed to support limited functionality at a lower overhead.  E.g. running an aggregation service or an auction.
1. General purpose: can run any program but imposes a higher computation overhead that may not be practical any time soon.  E.g. [garbled circuits](https://en.wikipedia.org/wiki/Garbled_circuit) and [secret sharing.](https://en.wikipedia.org/wiki/Secret_sharing)

Links:

* Special purpose MPC is part of the [Aggregation Service explainer.](https://github.com/WICG/conversion-measurement-api/blob/master/SERVICE.md)
