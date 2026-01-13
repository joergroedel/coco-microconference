# PCI device authentication & encryption ([Video](https://www.youtube.com/watch?v=h4W6wrTrigY))

Presenter: Mathieu Poirier

The provided video features a session from the Linux Plumbers Conference 2025
titled **"PCI Device Authentication & Encryption,"** presented by Mathieu
Poirier. The session focuses on integrating Security Protocol and Data Model
(SPDM) and Trusted Device Interface Security Protocol (T-DISP) into the Linux
kernel for PCI device authentication.

Here is a summary of the presentation and the subsequent discussion:

## **Presentation Highlights**

- **The Core Problem:** The speaker addressed the challenge of authenticating
  SPDM/T-DISP-compliant devices without creating an overly complex architecture.
  He highlighted two primary contexts:
  - **CMA/SPDM:** Component measurement and authentication.
  - **PCI/TSM:** Trusted execution environments and Trusted VMs.
- **Existing Patchsets:**
  - **Lukas Wunner’s Patch:** Focused on CMA/SPDM, proposing a framework for
    device authentication using the kernel keyring, sysfs for artifacts, and
    netlink for signature communication.
  - **Dan Williams’ Patch:** Focused on PCI/TSM, specifically enacting the
    T-DISP state machine for devices assigned to Trusted VMs.
- **Proposed Holistic Solution:** Mathieu Poirier advocated for merging these
  approaches into a unified framework. This would involve a consistent
  representation of SPDM artifacts in sysfs, identical netlink notification
  systems for measurements, and a shared probing policy and kernel authentication
  process for both CMA and TSM use cases.

## **Key Discussion Points**

The presentation was followed by an active technical debate among attendees,
including Dan Williams and others, regarding implementation details:

- **Mechanism vs. Policy:** A major point of discussion was whether the kernel
  should handle authentication "policy" or if that should be strictly relegated
  to user space. There was general consensus that the kernel should provide the
  "mechanism," while the user space (potentially within the initrd) should handle
  the decision-making policy.
- **Probing Policy:** Participants discussed "auto-probing." It was suggested
  that for high-security environments, the kernel should be configured to **not**
  auto-probe PCI devices. Instead, user space would verify the device and then
  manually trigger the probe once trust is established.
- **Sysfs vs. Netlink:** The group debated the best interface for various
  tasks. While sysfs is useful for displaying device status and artifacts,
  attendees argued that **Netlink** is far superior for request-response
  transactions, such as handling cryptographic nonces and signature
  artifacts.
- **Boot Timing and Keyrings:** There were concerns about when certificates are
  installed in the kernel keyring. Some suggested leveraging the **MOK (Machine
  Owner Key)** mechanism from Secure Boot to preload device certificates before
  the root filesystem is even mounted, solving the "chicken and egg" problem of
  early boot authentication.
- **Coexistence of Sessions:** There was a clarification that while the CMA and
  CVM (Confidential VM) features can coexist in the kernel, an active SPDM
  session cannot be shared simultaneously between a non-trusted host and a
  trusted VM due to hardware/protocol limitations.

### **Conclusion**

The session concluded with a "violent agreement" on the general direction:
moving away from complex sysfs implementations in favor of using Netlink for
complex communication and ensuring that user space retains control over the
final authentication policy before drivers are allowed to probe sensitive
devices.

