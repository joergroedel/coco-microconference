# Secure Interrupt delivery: Lessons Learned from Alternate Injection Enablement ([Video](https://www.youtube.com/watch?v=cfiAn5HNBuU))

Presenter: Melody Wang

Melody Wang, representing the AMD Server OS group, presented a session on
secure interrupt delivery, specifically focusing on lessons learned from
alternate injection enablement. The presentation addressed security concerns
where a hypervisor, while untrusted, remains responsible for interrupt
injection, potentially compromising guest confidentiality and integrity.

Key highlights from the presentation and the subsequent discussion include:

- **Need for Secure Interrupt Delivery:** Wang highlighted vulnerabilities
  where an untrusted hypervisor could inject malicious interrupts, citing
  specific CVEs. Solutions like restricted injection, alternate injection, and
  secure AVIC aim to give the guest more control.
- **Alternate Injection Mechanism:** This approach involves the hypervisor
  injecting interrupts into the Secure VM Service Module (SVSM), which then uses
  alternate injection to deliver them to the guest OS at a lower Virtual Machine
  Privilege Level (VMPL).
- **KVM Design Considerations:** Wang proposed a separate path in KVM for
  secure interrupt delivery to ensure compatibility and isolation from other
  guest types. This involves using the SVSM APIC for certain VMPL levels and
  bypassing the KVM local APIC via a doorbell page.
- **OVMF Challenges:** A significant portion of the session was dedicated to a
  protocol mismatch in OVMF, which starts in xAPIC mode, while alternate
  injection only supports x2APIC. 
- **Proposed Solutions and Feedback:** Two potential solutions for the OVMF
  issue were discussed: moving x2APIC enablement to an earlier phase or enabling
  it unconditionally for confidential guests. The discussion also touched on the
  nuances of hardware acceleration for different APIC modes across various AMD
  processor generations (Milan vs. Genoa) and the potential use of hint bits to
  inform the guest of the preferred mode.
- **Current Status:** Wang concluded by stating that the Proof of Concept (POC)
  is complete, with patches for OVMF and the guest currently being sent upstream.
  Future work involves basing KVM upstream patches on broader VMPL support.
