# Title: Improving Intel TDX Linux Integration

### **1. Presentation Overview**
* **Speaker:** Elena Reshetova (Intel)
* **Session:** Confidential Computing Microconference at Linux Plumbers Conference (LPC)
* **Topic:** *Improving TDX Linux Integration: Changes in TDX Migration and Attestation*
* **Core Objective:** Share major architectural changes Intel made to Intel Trust Domain Extensions (TDX) to address Linux maintainer and cloud service provider (CSP) feedback, improve maintainability, and align confidential computing approaches.

---

### **2. Background: The Problem with "Service TDs"**
* **Original Issue:** Complex, long-running operations inside the TDX Module (such as live migration setup, cryptographic operations, and attestation) must be interruptible so the host hypervisor (KVM) can service interrupts. Breaking these into tiny, state-preserving calls inside firmware adds significant complexity to the TDX module.
* **Initial Solution (Service TDs):** Intel initially proposed "Service TDs"—dedicated, specialized confidential VMs running these complex tasks alongside regular user TDs.
* **Why it Failed in Linux:** Service TDs required Linux/KVM and host orchestrators to explicitly manage their full lifecycle (allocating memory/vCPUs, scheduling, destruction, and binding/rebinding them to tenant TDs). Linux developers and CSPs pushed back, considering this too complex, costly, and invasive for host management.

---

### **3. The Architectural Shift: TDX Module Extensions**
* **The New Design:** Rather than exposing Service TDs to the host OS, Intel introduced **TDX Module Extensions**.
* **Host Perspective:** To KVM/QEMU, extensions behave like standard interruptible and resumable `SEAMCALL`s with minimal host management.
* **Under the Hood:** Currently implemented internally as "hidden TDs" fully orchestrated by the TDX module itself, keeping the complex lifecycle management within the module's boundary and out of the host kernel.

---

### **4. Key Use Cases**
1. **TEE-I/O Provisioning Agent (TPA):** Manages SPDM (Security Protocol and Data Model) connections between the host and trusted devices.
2. **Quoting Service:** Migrates TDX attestation away from SGX-based attestation toward standard, DICE-based attestation.
3. **Migration Setup Service:** Establishes secure sessions, performs policy negotiation, and exchanges migration keys between source and target hosts before live migration begins.
4. **Ephemeral vTPM:** A potential future use case under exploration.

---

### **5. Concurrency & Scheduling Constraints**
* **Configurable Thread Pool:** During TDX module initialization, the host configures a fixed pool size of concurrent processing threads for each extension.
* **Interrupt Resumption Rule:** If an extension `SEAMCALL` is interrupted, the assigned thread slot remains busy until resumed on the same thread context. KVM/QEMU must track and properly resume interrupted calls to prevent thread starvation.
* **Memory Trade-Off:** Allocating larger thread pools consumes memory (e.g., tens of megabytes reserved upfront for module structures), so CSPs must balance concurrency needs against memory footprint.

---

### **6. Discussion & Q&A Highlights**
* **Networking & Policy Negotiation During Migration:**
  * *Question:* How does policy negotiation and key establishment work if extensions lack a network stack?
  * *Elena's Answer:* The extension handles all cryptographic logic, handshake negotiation, and policy enforcement within the secure boundary. The untrusted host hypervisor merely serves as the network transport ("middleman"), passing encrypted handshake packets across the standard network/QEMU control channel. Once approved, the negotiated keys are installed directly into the TDX Module.
* **Dynamic Thread Allocation & Concurrency:**
  * *Question:* Can the number of concurrent processing threads be resized dynamically at runtime, or why not simply allocate one thread per physical CPU?
  * *Elena's Answer:* Dynamic resizing is not currently supported to keep the implementation simple. Sizing threads equal to total physical CPUs is possible, but incurs memory overhead (roughly 55–90 MB depending on the extension), so fixed upfront sizing based on expected concurrent TD operations is preferred.

---

### **7. Lessons Learned**
Elena concluded with key takeaways for integrating confidential computing hardware into Linux:
* **Expose primitives, not policies:** Keep security enforcement inside the trusted computing base (TCB), but leave resource management and scheduling policies to the host operating system.
* **Respect existing Linux abstractions:** Avoid introducing bespoke architectural patterns when standard kernel primitives can be used.
* **Simplicity vs. Security:** Balance hardware security features against software operational complexity.
* **Cross-vendor collaboration:** Align designs across hardware vendors (e.g., AMD SEV, ARM CCA, Intel TDX) to ensure consistent kernel abstractions and avoid fragmentation.
