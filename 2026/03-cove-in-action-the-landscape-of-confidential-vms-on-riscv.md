# Title: CoVE in Action: The Landscape of Confidential VMs on RISC-V

### **1. Executive Summary & Overview**
* **Title:** *CoVE in Action: The Landscape of Confidential VMs on RISC-V*
* **Speaker:** Ruoqing He (LingCage), presenting joint work with Xiaoxia Cui (Damo Academy, Alibaba).
* **Session:** Confidential Computing Microconference at Linux Plumbers Conference (LPC).
* **Core Topic:** Demonstrating an end-to-end open-source proof-of-concept (PoC) of **CoVE (Confidential VM Extension)** running in **Deployment Mode 1 (DM1)** on RISC-V, outlining the upstreaming roadmap, and discussing major architectural hurdles encountered in Linux integration.

---

### **2. Architecture: CoVE Deployment Mode 1 (DM1)**
Unlike earlier PoCs (such as RivOS’s Salus, which used **DM2** where the host hypervisor is nested under a secure hypervisor in VS-mode), this implementation focuses on **DM1**:
* **HS-Mode Peers:** Both the Host OS/KVM and the **TSM (Trusted Security Manager)** run side-by-side in Hypervisor-Supervisor (HS) mode as peers, isolated into distinct supervisor domains by hardware via **sPMP / Smpt** (Memory Protection Table).
* **TSM Implementation:** Built in Rust (`xvm2`, `no_std`, heapless), open-sourced by the team. TSM manages confidential VM (TVM) page tables, vCPU contexts, and memory state.
* **M-Mode RDSM:** The Root Domain Security Manager (implemented in extended **OpenSBI**) handles domain switching between KVM and the TSM via `TEECALL` and `TEERET`.
* **Host & Guest Software:** Host runs Linux 6.6 + KVM with `kvmtool` as the VMM. Guest runs Linux 6.6 with CoVE support.
* **Hardware/Emulation:** Tested on both real silicon/FPGA (T-Head Xuantie C910) and internal QEMU with Smpt extensions.
* **Demonstrated PoC Capabilities:**
  * Boots a confidential guest (currently 1 TVM with 1 vCPU).
  * Validates memory confidentiality: attempts by the host root user to read TSM memory or guest private memory trigger hardware load-access faults. Once the TVM is destroyed, memory is scrubbed and safely reclaimed by the host.

---

### **3. Upstreaming Status & Four Core Challenges**

Currently, only the TSM (`xvm2`) is open source; kernel, OpenSBI, and QEMU patches remain out-of-tree. The presenter highlighted four major integration challenges:

#### **Problem 1: KVM Host Support & Upstreaming Order**
* **Plan:** Freeze specifications one extension at a time, upstreaming guest-side patches first (`CoVE-G`, following the path taken by Intel TDX, AMD SEV, and Arm CCA), followed by host-side support and interrupt virtualization (`CoVE-I`).
* **Open Questions:** Is `guest_memfd` strictly required for initial upstreaming? Is hard-pinning each vCPU to a physical hart acceptable in v1?
* **Discussion/Feedback:** 
  * Audience members (including KVM developers) confirmed that **`guest_memfd` is now a de facto prerequisite** for confidential guest memory upstreaming across all architectures.
  * Pinning vCPUs to harts is acceptable for early RFCs and code sharing, but dynamic scheduling across harts is required for production merge.

#### **Problem 2: Guest I/O & Memory Sharing Semantics (Page vs. Region)**
* **The Issue:** The Linux kernel's standard confidential computing I/O mechanisms (`swiotlb`, `set_memory_decrypted()`) convert and share memory **one allocation/page at a time**. However, the current CoVE specification mandates sharing memory by **regions** (with TSM only tracking up to 16 regions). During boot, bursts of adjacent page conversions quickly exhausted region tracking slots unless TSM proactively merged them.
* **Discussion/Feedback:**
  * Attendees compared this to TDX and Arm CCA, where conversion requests are tracked per-page (idempotent state transitions) rather than static ranges.
  * Audience feedback noted that because the CoVE specification is not yet finalized or ratified, **RISC-V should consider dropping the region-based sharing model** entirely in favor of page-granularity sharing to align directly with Linux abstractions.

#### **Problem 3: Attestation Architecture & Root of Trust**
* **The Plan:** Reuse the standard Linux user/kernel ABI (`/sys/kernel/config/tsm/report`) rather than introducing a RISC-V–specific interface. Produce evidence formatted as Entity Attestation Tokens (EAT) based on the DICE standard, verifying via Trustee/Enarx plugins.
* **Open Questions:** Who endorses the root of trust across fragmented RISC-V SoC vendors, and how are reference measurements distributed?
* **Discussion/Feedback:**
  * Attendees pointed out that RISC-V's openness allows for multiple endorsement models, such as user-provided signing keys or vendor CAs.
  * Another attendee stressed the importance of tracking unique instance identifiers (e.g., Realm/Platform Instance IDs) to defend against redirection/man-in-the-middle attacks, pointing out ongoing work in the IETF RATS (Remote Attestation Procedures) working group.

#### **Problem 4: vCPU State & NACL Layout**
* **The Issue:** The original 2023 CoVE RFC assumed nested virtualization (DM2) and utilized NACL (Nested Acceleration) shared memory between L0 and L1. In DM1, the host is not nested.
* **Proposed Solution:** Introduce a new call (`COVE_H_SET_RUN_PAGE`) where the host passes a shared run-page to exchange exit reasons and CPU register state with the TSM upon VM exits.
* **Discussion/Feedback:** Attendees noted that future iterations of the specification are exploring lighter message-passing mechanisms between KVM and the TSM to minimize unnecessary register sharing and avoid complex shared memory layouts.
