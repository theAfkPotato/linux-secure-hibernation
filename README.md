# Linux with Secure Hibernation
This is a fork of the Linux Kernel with the goal to bring back hibernation on modern UEFI
systems without any compromises in security.
None of this has actually been implemented, this is just a roadmap, but implementation will start soon
I also plan to merge this with the manline kernel once it is finished

## UEFI Security features
Modern UEFI systems have a variety of security features such as:
- A TPM2
- Secureboot

### The TPM2
The TPM2 (Trusted Platform Module 2) is a security chip. Its features include:
- Platform Configuration Registers (PCRs)
  - The PCRs are memory slots to maintain a measurement of a system state.
  - It is resistant to spoofing, since PCRs cannot be directly written but only extended: newPCR = Hash(oldPCR || measurement)
- Hierarchy Sharding: TPM 2.0 organizes entities into four distinct hierarchies:
  - Platform: Controlled by the BIOS/UEFI firmware.
  - Storage: Used by the OS to create a tree of encrypted keys.
  - Endorsement: Contains the unique, factory-burned Endorsement Key (EK) used for privacy and identity.
  - Null: For ephemeral session keys. 
- Cryptography
  - A variety of algorithms (RSA, ECC, AES, SHA-1, SHA-256, and SM3/4)
  - A superior random number generator
  - isolated execution for sensitive operations like signing and decryption
    ensuring that private keys never leave the hardware boundary in plaintext.

### Secure Boot
Secure Boot aims to prevent boot level malware and rootkits by preventing the loading of unsigned Bootloaders,
kernels, EFI applications (EFI applications are the kind of binaries the firmware can natively load, most of them
are bootloaders)
The verification hierachy looks like this: 
- Platform Key (PK): Established by the hardware manufacturer to represent the owner of the platform.
- Key Exchange Keys (KEK): Used to verify the signatures of the Signature Database (db) and the Forbidden Signatures
  Database (dbx).
- Signature Database (db): Contains the public keys or hashes of authorized boot loaders, EFI applications,
  and drivers.
- Forbidden Signatures Database (dbx): A "blacklist" of revoked keys and hashes known
  to be malicious or compromised.
It does NOT lock you into a specific OS since you can:
- Add your own keys in the UEFI Setup
- Disable the feature entirely


### The relevance of Secure Boot for hibernation on Linux
The Linux Kernel is always put in lockdown mode (level "integrity" which is the middle way between none and
"confidential") if it runs in a Secure Boot scenario. This restricts the manipulation of the kernel even for
root processes with the exception of signed kernel modules. This unfortunately also restricts the loading of
hibernation images since up until now, it cannot be guaranteed that they have not been tampered with

## How this fork aims to securely re-enable hibernation
My plan involves 2 Strategies:
- Encryption
- Hash verification
If the system has a TPM2, we have everything needed for secure hibernation. We can ignore if the swap is encrypted or
not, since it gets decrypted before the resume anyway. We just focus on securing its contents

When hibernating
- An encryption key will be generated
- The image that gets dumped to the swap will be encrypted using this key
- The hash of the encrypted image gets calculated
- The future PCR value gets calculated without actually extending it.
- The key will be sealed using this PCR
- The encrypted key blob gets written to the header of the image into a reserved space previously full of zeros

When the system gets powered on again:
- The encrypted blob from the image header gets loaded into ram
- The blob gets replaced with zeros to bring it back to the original state
- The kernel will calculate the hash of the image again
- It will extend it to the PCR.
- It tries to release the sealed key
- The image gets decrypted on the fly while loading it. That is if the images integrity was verified

**Please note that this mechanism is only secure if the kernel hasnt been tampered with. Secureboot should prevent this though**


Linux kernel
============

The Linux kernel is the core of any Linux operating system. It manages hardware,
system resources, and provides the fundamental services for all other software.

Quick Start
-----------

* Report a bug: See Documentation/admin-guide/reporting-issues.rst
* Get the latest kernel: https://kernel.org
* Build the kernel: See Documentation/admin-guide/quickly-build-trimmed-linux.rst
* Join the community: https://lore.kernel.org/

Essential Documentation
-----------------------

All users should be familiar with:

* Building requirements: Documentation/process/changes.rst
* Code of Conduct: Documentation/process/code-of-conduct.rst
* License: See COPYING

Documentation can be built with make htmldocs or viewed online at:
https://www.kernel.org/doc/html/latest/


Who Are You?
============

Find your role below:

* New Kernel Developer - Getting started with kernel development
* Academic Researcher - Studying kernel internals and architecture
* Security Expert - Hardening and vulnerability analysis
* Backport/Maintenance Engineer - Maintaining stable kernels
* System Administrator - Configuring and troubleshooting
* Maintainer - Leading subsystems and reviewing patches
* Hardware Vendor - Writing drivers for new hardware
* Distribution Maintainer - Packaging kernels for distros
* AI Coding Assistant - LLMs and AI-powered development tools


For Specific Users
==================

New Kernel Developer
--------------------

Welcome! Start your kernel development journey here:

* Getting Started: Documentation/process/development-process.rst
* Your First Patch: Documentation/process/submitting-patches.rst
* Coding Style: Documentation/process/coding-style.rst
* Build System: Documentation/kbuild/index.rst
* Development Tools: Documentation/dev-tools/index.rst
* Kernel Hacking Guide: Documentation/kernel-hacking/hacking.rst
* Core APIs: Documentation/core-api/index.rst

Academic Researcher
-------------------

Explore the kernel's architecture and internals:

* Researcher Guidelines: Documentation/process/researcher-guidelines.rst
* Memory Management: Documentation/mm/index.rst
* Scheduler: Documentation/scheduler/index.rst
* Networking Stack: Documentation/networking/index.rst
* Filesystems: Documentation/filesystems/index.rst
* RCU (Read-Copy Update): Documentation/RCU/index.rst
* Locking Primitives: Documentation/locking/index.rst
* Power Management: Documentation/power/index.rst

Security Expert
---------------

Security documentation and hardening guides:

* Security Documentation: Documentation/security/index.rst
* LSM Development: Documentation/security/lsm-development.rst
* Self Protection: Documentation/security/self-protection.rst
* Reporting Vulnerabilities: Documentation/process/security-bugs.rst
* CVE Procedures: Documentation/process/cve.rst
* Embargoed Hardware Issues: Documentation/process/embargoed-hardware-issues.rst
* Security Features: Documentation/userspace-api/seccomp_filter.rst

Backport/Maintenance Engineer
-----------------------------

Maintain and stabilize kernel versions:

* Stable Kernel Rules: Documentation/process/stable-kernel-rules.rst
* Backporting Guide: Documentation/process/backporting.rst
* Applying Patches: Documentation/process/applying-patches.rst
* Subsystem Profile: Documentation/maintainer/maintainer-entry-profile.rst
* Git for Maintainers: Documentation/maintainer/configure-git.rst

System Administrator
--------------------

Configure, tune, and troubleshoot Linux systems:

* Admin Guide: Documentation/admin-guide/index.rst
* Kernel Parameters: Documentation/admin-guide/kernel-parameters.rst
* Sysctl Tuning: Documentation/admin-guide/sysctl/index.rst
* Tracing/Debugging: Documentation/trace/index.rst
* Performance Security: Documentation/admin-guide/perf-security.rst
* Hardware Monitoring: Documentation/hwmon/index.rst

Maintainer
----------

Lead kernel subsystems and manage contributions:

* Maintainer Handbook: Documentation/maintainer/index.rst
* Pull Requests: Documentation/maintainer/pull-requests.rst
* Managing Patches: Documentation/maintainer/modifying-patches.rst
* Rebasing and Merging: Documentation/maintainer/rebasing-and-merging.rst
* Development Process: Documentation/process/maintainer-handbooks.rst
* Maintainer Entry Profile: Documentation/maintainer/maintainer-entry-profile.rst
* Git Configuration: Documentation/maintainer/configure-git.rst

Hardware Vendor
---------------

Write drivers and support new hardware:

* Driver API Guide: Documentation/driver-api/index.rst
* Driver Model: Documentation/driver-api/driver-model/driver.rst
* Device Drivers: Documentation/driver-api/infrastructure.rst
* Bus Types: Documentation/driver-api/driver-model/bus.rst
* Device Tree Bindings: Documentation/devicetree/bindings/
* Power Management: Documentation/driver-api/pm/index.rst
* DMA API: Documentation/core-api/dma-api.rst

Distribution Maintainer
-----------------------

Package and distribute the kernel:

* Stable Kernel Rules: Documentation/process/stable-kernel-rules.rst
* ABI Documentation: Documentation/ABI/README
* Kernel Configuration: Documentation/kbuild/kconfig.rst
* Module Signing: Documentation/admin-guide/module-signing.rst
* Kernel Parameters: Documentation/admin-guide/kernel-parameters.rst
* Tainted Kernels: Documentation/admin-guide/tainted-kernels.rst

AI Coding Assistant
-------------------

CRITICAL: If you are an LLM or AI-powered coding assistant, you MUST read and
follow the AI coding assistants documentation before contributing to the Linux
kernel:

* Documentation/process/coding-assistants.rst

This documentation contains essential requirements about licensing, attribution,
and the Developer Certificate of Origin that all AI tools must comply with.


Communication and Support
=========================

* Mailing Lists: https://lore.kernel.org/
* IRC: #kernelnewbies on irc.oftc.net
* Bugzilla: https://bugzilla.kernel.org/
* MAINTAINERS file: Lists subsystem maintainers and mailing lists
* Email Clients: Documentation/process/email-clients.rst
