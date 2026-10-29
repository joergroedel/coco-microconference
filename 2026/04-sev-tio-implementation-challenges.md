# Title: SEV-TIO Implementation Challenges

### **1. Presentation Overview**
* **Speaker:** Alexey Kardashevskiy (AMD)
* **Title:** *AMD SEV-TIO: Implementation Challenges – Trusted Device Assignment for Confidential VMs*
* **Session:** Confidential Computing Microconference at Linux Plumbers Conference (LPC)
* **Goal:** Present the technical hurdles encountered while implementing and preparing upstream patches (~50 patches) for **AMD SEV-TIO (Trusted I/O)**, which enables secure, encrypted direct device assignment (PCIe/VFIO) into AMD SEV-SNP confidential virtual machines.

---

### **2. Core Architectural Challenge: Synchronizing Three Page Tables**
For trusted device assignment under SEV-TIO, three distinct page tables on the host must remain strictly synchronized on every access:
1. **NPT (Nested Page Table):** Translates Guest Physical Addresses (GPA) to System Physical Addresses (SPA) for CPU access; untrusted, controlled by the hypervisor (KVM).
2. **IOMMU Page Table:** Translates device DMA addresses to SPA; supports multiple page sizes (4KB, 2MB, etc.).
3. **RMP (Reverse Map Table):** Hardware-enforced, security-critical table managed by the AMD Secure Processor (PSP) that tracks page ownership, encryption state, and validation status (supports 4KB and 2MB entries via `RMPUPDATE` and `PSMASH`).

Because these structures use different formats, levels, and granularities, coordinating page-size changes and permission transitions across all three is complex.

---

### **3. Challenge 1: Distinguishing Private vs. Shared DMA (vTOM vs. C-bit)**
* **The Problem:** Trusted devices inside the guest’s trusted computing base (TCB) must perform DMA not only to private encrypted memory, but also to shared (unencrypted) memory (e.g., for virtio queues, peer-to-peer DMA with legacy devices, or network buffers).
* **Why the C-bit Fails for Devices:** SEV-SNP uses bit 51 (the C-bit) in physical addresses to indicate encryption. However, many PCIe devices only support 32-bit to 48-bit DMA addressing and cannot set bit 51.
* **AMD’s Workaround (vTOM - Virtual Top of Memory):**
  * Divides the guest address space: memory below vTOM is private/encrypted; memory above vTOM is shared/unencrypted.
  * QEMU mirrors guest RAM above vTOM, and devices perform shared DMA by adding the vTOM offset to DMA addresses.
* **Community Discussion (Jason Gunthorpe & James Bottomley):**
  * Jason Gunthorpe noted that this matches Arm CCA’s approach (where the top bit of the IPA indicates shared memory).
  * Jason emphasized that **vTOM should be treated as a global per-VM property in the architecture code**, not embedded inside device-specific or PCI-level structures.
  * The VM builder must enforce that the guest physical address space plus the shared bit does not exceed the hardware DMA addressing capability of assigned devices (e.g., a 40-bit device limits the guest address space to ~39 bits).
  * Regarding **ATS (Address Translation Services)**: If no guest vIOMMU is present, identity translation is used, and the hypervisor handles full Address Translation Cache (ATC) invalidations when mappings change.

---

### **4. Challenge 2: Page Smashing (`PSMASH`) and IOMMU Synchronization**
* **The Problem:**
  * To maximize performance, guests typically boot with 2MB huge pages mapped in both NPT and the RMP.
  * When a guest performs dynamic 4KB conversions (e.g., allocating a 4KB shared buffer for `/dev/sev-guest` attestation or SWIOTLB), hardware executes a `PSMASH` to split the 2MB RMP entry into 512 × 4KB entries.
  * With an assigned device actively performing DMA, the IOMMU’s 2MB mapping must also be shattered into 4KB entries synchronously to match the RMP; otherwise, RMP hardware security checks fail.
* **AMD’s Hardware Solution (TMPM):** AMD uses the Trusted Memory Protection Module (TMPM) on the EPYC processor to momentarily stall DMA traffic (for microseconds), synchronize the page tables, and smash both the RMP and IOMMU entries.
* **Community Consensus & Recommendation:**
  * **Jason Gunthorpe and Michael Roth (AMD)** strongly recommended **avoiding dynamic 4KB page smashing altogether during active DMA**.
  * Instead, the guest kernel should **pre-allocate shared DMA bounce pools (SWIOTLB) at boot time in 2MB-aligned huge page chunks**.
  * By restricting shared conversions to 2MB boundaries, the IOMMU, NPT, and RMP maintain 2MB mappings throughout runtime, eliminating fragmentation, the need for complex runtime smashing logic, and the dependency on TMPM stalling.
  * The audience noted that Arm CCA adopted a similar approach to simplify hypervisor/IOMMU management.

---

### **5. Challenge 3: IOMMU TLB Invalidation & VMPL**
* Changing page states (private ↔ shared, read-write ↔ read-only, or VMPL access levels) requires proper invalidation of IOMMU TLB caches, which store not only translation addresses but also RMP check results and page sizes.
* While specific operations like vCPU hotplug require 4KB private page allocations (VMSA pages), attendees agreed these can be handled separately from device DMA paths without requiring general runtime 4KB smashing.
