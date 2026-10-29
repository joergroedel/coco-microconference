# Title: ARM CCA Realm Live Migration

### **1. Overview & Speaker Introduction**
* **Speaker:** Mathias Brossard (Arm)
* **Session:** Confidential Computing Microconference at Linux Plumbers Conference (LPC)
* **Topic:** *Arm CCA Realm Live Migration*
* **Core Objective:** Present the architectural design and ABIs for Arm Confidential Compute Architecture (CCA) Realm live migration, compare it against the proposed upstream KVM Live Migration API (RFC by Intel/SUSE authors), and discuss open challenges and areas needing adaptation for Arm CCA.

---

### **2. Arm CCA Architecture & Core Principles**
* **Architectural Boundaries:**
  * **Normal World (Non-secure):** Runs host Linux, KVM (at NS-EL2), and VMM/QEMU (at NS-EL0).
  * **Realm World:** Runs the confidential guest ("Realm") at R-EL1/R-EL0, governed by the **Realm Management Monitor (RMM)** at R-EL2.
* **Migration Scope:**
  * Only migrates **architectural state** (CPU registers, memory contents, stage-2 page tables, virtual SMMU state). Non-architectural physical device states (e.g., PCIe) are excluded.
  * Focuses initially on 4KB page-by-page migration (block/huge page mappings are deferred for future consideration).
* **Dirty Page Tracking Mechanisms:**
  * **Stage-2 Page Table Dirty-Bit Tracking (Current):** CPU MMU and IOMMU update hardware dirty bits in the stage-2 page tables. The RMM scans these tables on demand and reports dirty pages to KVM.
  * **Dirty-Bit Logging (Future / Architectural):** Hardware appends dirty GPAs directly into dedicated logging buffers (similar to Intel PML), which RMM exposes to KVM.

---

### **3. Alignment with the Proposed KVM Live Migration API**
Mathias walked through the phases of the proposed generic KVM Live Migration ioctl RFC to evaluate how well it matches Arm CCA requirements:

#### **A. Setup Phase**
* **Live Migration Service Realm:** Unlike Intel’s recent pivot to internal TDX extensions, Arm CCA currently uses a dedicated **Service Realm** on both the source and target hosts.
* **Authenticated Channel:** The Service Realms establish a secure end-to-end channel via attestation to negotiate session keys and migration policies.
* **Semantic Mismatch:** In the RFC KVM flow, setup and handshake messages pass directly through the host hypervisor. For CCA, session establishment is handled between the Service Realms (out-of-band or tunneled opaquely), so the host only manages initialization, tracking structure allocation, and dirty tracking activation.

#### **B. Pre-Copy Phase**
* **Streams and Epochs:** Exported memory pages are assigned to logical "streams" tracked by the RMM to ensure ordering and cryptographic integrity across iterations/epochs.
* **Metadata vs. Content:** Exports can consist of page content plus metadata (GPA, integrity protection values) or metadata-only (for unbacked/zero pages).
* **Limitations of `KVM_GET_DIRTY_LOG`:**
  * The traditional `KVM_GET_DIRTY_LOG` expects a monolithic per-memslot bitmap. 
  * In CCA, RMM must scan page tables dynamically and outputs a list of Guest Frame Numbers (GFNs). Converting GFN lists into bitmaps in the kernel, only for userspace/VMM to convert them back into lists, is inefficient.
  * Arm CCA requires the ability to scan page tables **concurrently across multiple threads/vCPUs** by querying specific GPA ranges.

#### **C. Stop-and-Copy (Downtime) Phase**
* **Unmapped GPA Range Scanning:** Guest page tables can store permissions/state for memory ranges outside standard KVM memory slots (e.g., unbacked or reserved ranges). RMM must scan and export metadata for these regions.
* **Dynamic Sizing:** Buffer sizes for exporting vCPU states and memory metadata vary across hardware implementations and must be discoverable at runtime rather than fixed.

---

### **4. Discussion & Q&A Highlights**

* **Scanning Outside Memory Slots:**
  * *Question (Sean Christopherson):* Why is scanning GPA ranges outside KVM memory slots security-critical?
  * *Mathias' Answer:* A Realm guest can set permissions or attributes on unmapped address ranges. If memory is later populated or attributes persist, they must be preserved across migration to guarantee security and architectural invariants.

* **Post-Copy Live Migration:**
  * *Question (Audience):* Why focus only on pre-copy and stop-and-copy? Is post-copy (faulting pages in on demand) planned?
  * *Mathias' Answer:* Post-copy is architecturally possible (using an invalidation call to transfer metadata first and demand-paging contents later). However, cloud providers currently express hesitation about deploying post-copy in production confidential computing environments due to reliability and network fault handling.

* **SMMU Hardware Dirty Logging:**
  * *Question (Online attendee):* Does the SMMU support dirty logging in hardware today?
  * *Mathias' Answer:* While current SMMU hardware primarily supports Hardware Translation Table Updates (HTTU/dirty-bit updating), the SMMUv3.5 specification introduces hardware dirty-bit logging buffers.

* **Proposed Alternative to `KVM_GET_DIRTY_LOG`:**
  * *Question / Audience Discussion:* What should replace `KVM_GET_DIRTY_LOG` for confidential guests?
  * *Consensus:* A range-based query ioctl that returns dirty GFNs directly to userspace. Because traversing secure stage-2 page tables inside firmware (RMM) carries a latency cost, the API must allow userspace to divide the guest address space into chunks and scan them in parallel across multiple threads.

* **Migration Epoch Limits:**
  * *Question (Elena Reshetova):* What happens if migration does not converge before exhausting the permitted number of iteration epochs?
  * *Mathias' Answer:* The migration must be aborted. Tearing down the session and restarting requires a full setup sequence and key re-negotiation, making convergence tuning critical.
