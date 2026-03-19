# The 2025 Confidential Computing Microconference at LPC

The Linux Plumbers Conference 2025 in Tokyo featured the fifth edition of the
Confidential Computing microconference.  It again brought together kernel
developers, virtualization experts, and researchers which discussed how to
integrate Confidential Computing into the Linux and open source ecosystem.

The microconference had seven sessions on a broad range of subject matter
topics, which are summarized below.

## Optimizing `guest_memfd` Shared/Private Conversions

The first session Ackerley Tng lead the discussion on optimizing memory state
conversions within the `guest_memfd` framework. The primary technical hurdle
discussed was the high overhead of Folio splitting, where contiguous memory
pages must be broken down and restructured during transitions between private
and shared states. This current process is resource-intensive because it
requires re-allocating metadata, updating XArray entries, and undoing HugeTLB
optimizations. To solve this, Ackerley proposed shifting from physical memory
restructuring to a mapping-based approach, where `guest_memfd` acts as a
manager of page tables (Stage 2, IOMMU, and Host Userspace) rather than a
physical memory organizer.

The discussion highlighted several architectural nuances and potential
implementation paths. Participants debated the necessity of unmapping memory
from the IOMMU, noting that hardware protections vary: Intel TDX might trigger
a host-crashing machine check on unauthorized access, while AMD SEV-SNP and ARM
CCA provide hardware-level checks that prevent such writes. There was also a
strong push to move away from `get_user_pages()` (GUP) in favor of specialized
`VM_PFNMAP` VMAs to improve efficiency. The session concluded with a focus on
refining the userspace API to allow for better orchestration of IOMMU unmapping
and a call for further collaboration to integrate these mapping-based
optimizations into the Linux kernel.

[Session notes](https://github.com/joergroedel/coco-microconference/blob/main/2025/optimizing_guest_memfd_shared_private_conversions.md) | [Video](https://www.youtube.com/watch?v=4jodIc-t3vY)

## Toward a standard device attestation token for device assignment

This session lead by Mathieu Poirier addressed the security challenges of
assigning hardware devices, such as GPUs or network cards, to Trusted Virtual
Machines without compromising the integrity of the Confidential Computing
environment.  Current workflows for device assignment lack a unified standard,
leading to fragmented implementations across different hardware vendors. To
solve this, the presenters proposed a standardized **Attestation Token** based
on the IETF's **Entity Attestation Token (EAT)** format. This token would act
as a cryptographically signed identity card for the device, containing
essential evidence claims—such as firmware measurements and configuration
states—that the TVM can verify before granting the device access to its memory
space.

The discussion also focused on the plumbing required within the Linux kernel to
support this standard. A major point of consensus was the need for a
vendor-neutral interface that allows the kernel to fetch these tokens from the
device (or a Trusted Device Manager) and pass them to the TVM. This is
particularly vital for supporting live migration, where a TVM must re-establish
trust with a new physical device on a different host without interrupting the
workload. By standardizing the token format and the communication protocol
(potentially via netlink sockets or a dedicated TSM/CMA framework), the
community aims to ensure that hardware from different manufacturers can be
seamlessly and securely integrated into any Linux-based confidential computing
stack.

[Session notes](https://github.com/joergroedel/coco-microconference/blob/main/2025/towards_a_standard_device_attestation_token_for_device_assignment.md) | [Video](https://www.youtube.com/watch?v=-zZrDuvgMgQ)

## PCI device authentication & encryption

Another session lead by Mathieu Poirier focused on standardizing how the Linux
kernel handles security for high-performance peripherals. The primary goal is
to integrate two previously separate security frameworks: **CMA/SPDM**
(Component Measurement and Authentication), which verifies a device's identity
and firmware integrity, and **PCI/TSM** (Trusted Security Module), which
manages device assignment for Confidential Virtual Machines (CVMs). By merging
these, the community aims to create a unified system where device certificates
and measurements are exposed consistently through `sysfs`, notifications are
handled via `netlink` sockets.

In the discussion a consensus emerged to separate kernel-level mechanisms from
user-space authentication policies. To enhance security, the group favored a
manual probing model where user space verifies PCI devices before triggering a
probe, rather than allowing the kernel to auto-probe. Technically, the
discussion shifted toward using **Netlink** over sysfs for complex
request-response transactions involving cryptographic artifacts. It was
suggested that **Machine Owner Keys (MOK)** could resolve early-boot
certificate timing issues. Finally, the session concluded with an agreement to
prioritize Netlink for communications and ensure user space retains ultimate
control over device trust, while noting that active SPDM sessions cannot be
shared between a host and a confidential VM.

[Session notes](https://github.com/joergroedel/coco-microconference/blob/main/2025/pci_device_authentication_and_encryption.md) | [Video](https://www.youtube.com/watch?v=h4W6wrTrigY)

## Standardization of Attested TLS Protocols

This session lead by Muhammad Usama Sardar explored how remote attestation can
be integrated into TLS to support Confidential Computing use cases. The
presentation introduced a system model in which a TLS endpoint (typically the
server) acts as a RATS attester, interacting with a broader infrastructure that
includes the confidential computing platform, quoting agents, and virtual
machines. The central security goals are ensuring the **integrity and
freshness** of attestation evidence and **cryptographically binding** that
evidence to a specific TLS session, thereby preventing relay and replay-style
attacks. Within the broader standardization landscape—particularly in the
Internet Engineering Task Force (IETF)—multiple working groups such as TLS and
SEAT are actively discussing how to achieve this integration, with a clear
focus on TLS 1.3 and forward-looking requirements like post-quantum security.

A major part of the session examined three architectural approaches:
**pre-handshake**, **intra-handshake**, and **post-handshake attestation**.
Pre-handshake designs (e.g., RA-TLS) perform attestation before establishing a
secure channel, while intra-handshake approaches embed evidence directly into
TLS messages. Post-handshake attestation, by contrast, performs verification
after the secure channel is established and allows for continuous or on-demand
validation. The discussion highlighted that pre- and intra-handshake approaches
face notable challenges, particularly around insufficient binding between
attestation evidence and the full TLS session context, as well as ensuring
freshness guarantees. These limitations can expose systems to relay or
diversion attacks. In contrast, post-handshake approaches were seen as less
intrusive and more flexible, since they avoid modifying the core TLS protocol
and can leverage existing mechanisms.

The discussion further expanded on key design considerations.  Participants
explored **mutual attestation (mTLS)** scenarios where both client and server
provide attestation evidence, which is important for many confidential
computing deployments. There was also debate about how to anchor trust, whether
in hardware-backed evidence (e.g., platform reports) or more abstracted
mechanisms like vTPMs. The consensus was leaning toward stronger guarantees
from hardware roots of trust. Another important theme was the reuse of existing
standards, particularly **channel binding mechanisms** (such as RFC 9261), to
securely link attestation evidence to TLS sessions instead of introducing
entirely new constructs.

[Session notes](https://github.com/joergroedel/coco-microconference/blob/main/2025/standardization_of_attested_tls_protocols.md) | [Video](https://www.youtube.com/watch?v=MF9AwkMJOlw)

## Discussion: TDISP, VM migration, and paravisors

The session lead by a group of Microsoft engineers explored how paravisors fit
into modern confidential computing architectures, particularly as guest
operating systems become more enlightened. The presenters contrasted two
models: the **paravisor model**, where a layer like Microsoft’s OpenHCL
abstracts all virtualization services for compatibility and simplicity, and the
**SVSM model**, where the guest OS takes on more responsibility and directly
interacts with privileged services.  The core challenge is enabling guests to
access advanced confidential computing features, such as attestation or
TDISP, without forcing them to abandon the simplicity and compatibility benefits
provided by a paravisor.

A key tension discussed was how to expose confidential computing capabilities
to the guest in a standardized and portable way. Participants highlighted this
as a **feature discovery problem**, with proposals such as using CPUID leaves
to advertise available services. However, this raises cross-vendor and
cross-architecture concerns, requiring careful standardization to work across
Intel TDX, AMD SEV-SNP, and even non-x86 platforms. The discussion also touched
on trust boundaries, especially when cloud providers control the paravisor, and
suggested that enabling customers to supply their own paravisor implementations
could strengthen trust models.

The session concluded with broad agreement that a **“blended” approach** is
needed, one that combines the convenience of paravisor-managed virtualization
with the flexibility of guest-aware confidential services. This implies the
need for improved Linux kernel support to recognize and interact with
paravisor-provided features in a generic way, moving beyond hypervisor-specific
ABIs. Such paravisor-aware plumbing would allow guest kernels to initialize
necessary drivers and securely consume higher-level services while maintaining
compatibility and reducing complexity.

[Session notes](https://github.com/joergroedel/coco-microconference/blob/main/2025/discussion_tdisp_vm_migration_and_paravisors.md) | [Video](https://www.youtube.com/watch?v=GNe6YEY-6Lo)

## A Linux Bus for SVSM Services: Build New, Reuse VIRTIO, or Both?

In this session Stefano Garzarella examined how Linux can better expose Secure
VM Service Module (SVSM) functionality to confidential guest VMs. Today,
SVSM-related support in Linux is fragmented, with separate, platform-specific
drivers for services like TPM and attestation. Garzarella proposed introducing
a **generic SVSM bus** that would unify these services behind a common
abstraction, while still allowing platform-specific transports for technologies
such as AMD SEV-SNP, Intel TDX, or future ARM-based solutions.

A central topic of discussion was **device discovery and standardization**.
Participants highlighted the need for a robust mechanism to enumerate
SVSM-provided devices, potentially combining platform-specific initialization
with globally unique identifiers such as UUIDs. The group also debated whether
to reuse **VIRTIO**, the widely adopted virtualization standard. While VIRTIO
offers an existing ecosystem, it presents challenges in this
context particularly around device enumeration conflicts with host-provided
VIRTIO devices and the difficulty of securely emulating MMIO without relying on
the hypervisor.

The discussion ultimately leaned toward the value of a **dedicated SVSM bus**,
especially for security-sensitive services that benefit from a simpler, more
controlled communication model. At the same time, there was openness to a
hybrid approach, where VIRTIO-like mechanisms could be leveraged for more
complex use cases in the future. The session concluded that while a new SVSM
bus is a promising direction for unifying confidential computing services in
Linux, further work is needed to refine discovery mechanisms and determine how,
or if, VIRTIO should be incorporated.

[Session notes](https://github.com/joergroedel/coco-microconference/blob/main/2025/a_linux_bus_for_svsm_services_build_new_reuse_virtio_or_both.md) | [Video](https://www.youtube.com/watch?v=RdVyN6P4UTQ) 

## Secure Interrupt delivery: Lessons Learned from Alternate Injection Enablement

In this session Melody Wang from AMD’s Server OS group explored how interrupt
delivery can be secured in confidential computing environments where the
hypervisor is not trusted. A central concern is that the hypervisor, which
traditionally injects interrupts into guest VMs, could act maliciously and
compromise guest integrity. Wang highlighted known vulnerabilities and CVEs in
this area, motivating the need for mechanisms such as restricted injection,
alternate injection, and secure AVIC that shift more control over interrupt
handling into trusted guest-side components.

The proposed **alternate injection mechanism** routes interrupts through the
Secure VM Service Module (SVSM), which acts as a proxy and delivers them to the
guest at a lower VMPL. This design reduces reliance on the hypervisor and
introduces a secure interrupt path. On the KVM side, Wang outlined the need for
a dedicated secure interrupt delivery path to maintain isolation from
traditional guests. This includes leveraging the SVSM APIC for certain
privilege levels and bypassing the standard KVM local APIC using a doorbell
page, ensuring compatibility with confidential VM requirements.

A key challenge discussed was a mismatch with OVMF, which initializes in xAPIC
mode while alternate injection requires x2APIC. Proposed solutions included
enabling x2APIC earlier in the boot process or enforcing it unconditionally for
confidential guests. The discussion also covered hardware-specific
considerations across AMD generations (e.g., Milan vs. Genoa) and the possible
use of hint bits to guide guest configuration. Wang concluded that a proof of
concept has been completed, with patches for OVMF and guest support already
submitted upstream.

[Session notes](https://github.com/joergroedel/coco-microconference/blob/main/2025/secure_interrupt_delivery_lessons_learned_from_alternate_injection_enablement.md) | [Video](https://www.youtube.com/watch?v=cfiAn5HNBuU)
