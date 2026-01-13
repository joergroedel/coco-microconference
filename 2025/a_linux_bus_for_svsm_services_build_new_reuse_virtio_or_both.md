# A Linux Bus for SVSM Services: Build New, Reuse VIRTIO, or Both? ([Video](https://www.youtube.com/watch?v=RdVyN6P4UTQ))

Presenter: Stefano Garzarella

In this presentation, Stefano Garzarella explores the implementation of a
Secure VM Service Module (SVSM) bus in Linux. SVSM provides security-related
services and devices to confidential guest VMs, such as AMD SEV-SNP guests.

Currently, Linux includes two SVSM drivers—one for a Trusted Platform Module
(TPM) and another for attestation reports—but both are platform-specific.
Stefano suggests creating a generic SVSM bus to unify these services and make
them platform-agnostic, with a core bus layer and platform-specific transports
for AMD, Intel, and ARM.

Key points from the discussion include:

- **Device Discovery:** There is a need for a reliable method to discover SVSM
  devices, possibly using platform-specific initialization and unique device IDs
  like UUIDs.
- **VIRTIO Considerations:** The group discussed whether to reuse VIRTIO. While
  VIRTIO is a common virtualization standard, using it for SVSM services presents
  challenges with device enumeration (potentially conflicting with host-provided
  VIRTIO buses) and MMIO emulation, which is difficult to implement securely
  without hypervisor involvement.
- **SVSM Bus Benefits:** A dedicated SVSM bus could provide a simplified,
  synchronous communication model better suited for services like TPM,
  while allowing for more complex VIRTIO-like transports in the future if
  needed.

The session concludes that while building a new SVSM bus seems promising for
unifying security services, further exploration is required to refine device
discovery and determine the extent to which VIRTIO can or should be integrated.
