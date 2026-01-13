# Discussion: TDISP, VM migration, and paravisors ([Video](https://www.youtube.com/watch?v=GNe6YEY-6Lo))

Presenters: Chris Oo (Microsoft), John Starks (Microsoft), Jon Lange (Microsoft)

This session from the Linux Plumbers Conference 2025, titled **"Paravisor
Integration with Confidential Services,"** was led by Jon Lange and John
Starks, virtualization architects at Microsoft. The presentation and subsequent
discussion focused on the architectural challenges of bridging the gap between
transparent paravisors and modern, "enlightened" guest operating systems that
need awareness of their confidential computing environment.

## **Presentation Overview**

The core of the presentation centered on the evolving role of the **paravisor**
(specifically mentioning Microsoft's **OpenHCL**) in a confidential computing
context.

- **Paravisor vs. SVSM Models:** Lange contrasted two main approaches:
  - **The Paravisor Model:** The paravisor takes full responsibility for all
    virtualization services (e.g., CPUID, MSRs, device emulation). It aims for
    high compatibility, often being transparent to legacy or unmodified guests.
  - **The SVSM (Secure VM Service Manager) Model:** More common in AMD SEV-SNP
    environments, where the guest OS handles more of its own service management
    and directly interacts with higher-privilege layers for specific services
    like virtual TPM (vTPM).
- **The Problem Statement:** Modern guests increasingly need to be
  "enlightened"—aware of their confidential environment to utilize specific
  services like **guest-driven attestation** or **TDISP (Trusted Device Interface
  Security Protocol)** for secure device binding. 
- **The Conflict:** How can a guest OS be informed that it is running in a
  confidential environment with these services available without forcing it to
  abandon paravisor-managed virtualization and take on complex, low-level tasks
  like handling its own virtualization exceptions?

## **Key Discussion Points**

The session transitioned into an open discussion with the audience,
highlighting several critical architectural and practical concerns:

- **Trust and Cloud Provider Influence:** An early question touched on the
  trust model when a cloud provider (like Microsoft) controls the paravisor.
  Lange pivoted this to an architectural discussion, suggesting a model where a
  customer could potentially bring their own paravisor implementation to maintain
  a strict trust boundary.
- **Kernel-Level Awareness:** There was consensus that even if services are
  exposed to user space (Ring 3), the guest kernel (Ring 0) must first be aware
  of the confidential environment to initialize the necessary drivers.
- **Feature Discovery Mechanisms:** David Woodhouse and others characterized
  the issue as a "feature discovery" problem. 
  - One suggestion was to use **CPUID leaves** to signal specific
    paravisor-supported features. 
  - However, participants noted that this approach needs careful
    standardization to be cross-vendor (Intel TDX vs. AMD SEV-SNP) and
    cross-architecture (x86 vs. ARM, where CPUID doesn't exist).
- **Standardization vs. Custom ABIs:** The current paravisor architecture often
  relies on hypervisor-specific ABIs (like Hyper-V's). The discussion explored
  the possibility of creating a more generic, paravisor-aware ABI that different
  hypervisors (KVM, Hyper-V, etc.) could implement.
- **A "Blended" Approach:** The presenters argued that confidential computing
  shouldn't be an "all or nothing" choice between full guest management and total
  transparency. The goal is to allow a blend where a paravisor handles complex
  virtualization tasks while still exposing modular, optional confidential
  services to an enlightened guest.

## **Conclusion**

The session concluded with a call for better **plumbing in the Linux kernel**
to support a common concept of "paravisor-aware" confidential computing. This
would allow the guest OS to discover and utilize high-level security features
while continuing to benefit from the simplified virtualization management
provided by a paravisor.
