# Title: PMU Event Filtering for Confidential Guests

### **1. Overview & Context**
* **Title:** PMU Event Filtering for Confidential Guests
* **Presenters:** Manali Shukla and Sandipan Das (AMD) at the Linux Plumbers Conference (Confidential Computing Microconference).
* **Topic:** Extending AMD’s hardware-assisted Performance Monitoring Unit (PMU) event filtering to confidential virtual machines (SEV-ES and SEV-SNP).

---

### **2. Background: What is PMU Event Filtering?**
* **Motivation:** In multi-tenant and virtualized environments, the hypervisor/host does not fully trust the guest. Allowing unconstrained PMU profiling could leak sensitive architectural or microarchitectural details (e.g., side-channel information, cache occupancy, or speculative execution behaviors).
* **PMU Event Filtering:** Allows the VMM/host to define an *allow* or *deny* list of events. Safe events (e.g., branch instructions, instructions retired) are permitted, while sensitive events are blocked.
* **Software-Based Filtering in KVM:**
  * Uses `KVM_SET_PMU_EVENT_FILTER` to register an allow/deny policy.
  * Every guest write to performance counter control MSRs (`PERF_CTL`) triggers a VM-exit.
  * KVM checks the policy: if allowed, KVM programs the hardware counter; if denied, the counter remains disabled.
  * **Drawback:** High VM-exit overhead, especially when profiling workloads multiplex many events across limited hardware counters.

---

### **3. Hardware-Assisted Guest Event Filtering (AMD)**
* **Mechanism:**
  * Newer AMD processors provide hardware-level filtering via the VMCB (Virtual Machine Control Block).
  * The host programs allowed event selectors directly into a dedicated VMCB control area (`ALLOWED_PERF_CTR[0..5]`).
  * When the guest writes to `PERF_CTL`, hardware checks the event against the VMCB allowed list:
    * **Allowed:** The write completes directly in hardware with **zero VM-exits**.
    * **Denied:** Standard SVM guests exit via `#VMEXIT_WRMSR`; SEV-ES/SNP guests trigger a `#VC` (VMM Communication) exception inside the guest.
* **Performance Impact:**
  * In benchmarks with counter multiplexing, standard mediated PMU approaches generated over **101,000 VM-exits**, whereas hardware-assisted filtering reduced exits to only **~7,500**.

---

### **4. The Challenge in Confidential Computing (SEV-ES / SEV-SNP)**
* In standard VMs, the VMCB save area is plaintext; KVM can inspect the attempted MSR write and update state.
* In SEV-ES and SEV-SNP, guest register states reside in the **encrypted VMSA** (Virtual Machine Save Area), which the hypervisor cannot read or modify.
* When hardware denies an event in an encrypted guest, KVM cannot directly inspect which event the guest attempted to program.

---

### **5. Proposed Solution: GHCB Protocol Protocol**
* **Workflow:**
  1. An unallowed `PERF_CTL` write triggers a `#VC` exception inside the confidential guest.
  2. The guest's `#VC` handler packages a request via the **GHCB** (Guest-Host Communication Block) shared buffer containing the attempted MSR events.
  3. KVM inspects the GHCB request, consults the VMM policy, updates the VMCB allowed list, and responds via the GHCB.
  4. The guest `#VC` handler clears the enable bit (`EN = 0`) for any denied counter in the MSR, preventing future profiling of that denied event without repetitive `#VC` traps.
* **Mid-Run Policy Changes:**
  * Changing filtering policies dynamically during guest execution is straightforward in standard VMs, but nearly impossible in confidential guests because KVM cannot read the guest’s active VMSA state to reconcile counters.
  * **Proposal:** Enforce that PMU event filters for confidential guests are **fixed at guest creation time** and disable dynamic mid-run filter modifications.

---

### **6. Key Discussion Points & Community Feedback**
* **Need for GHCB Exchange:**
  * *Question:* Why is a dynamic GHCB exchange needed rather than programming the allowed list upfront?
  * *Answer:* The hardware VMCB allowed list has a limited number of counter slots (e.g., 6 slots). When guests profile more events than available physical slots, event multiplexing requires dynamically updating which subset of allowed events is currently loaded in the VMCB.
* **Interaction with the Linux `perf` Subsystem:**
  * Perf maintainers/attendees urged caution regarding how denied MSR values are handled:
    * Silently clearing bits or altering the perceived value of `PERF_CTL` MSRs can break internal `perf` assumptions, corrupt profiling state, or cause issues during NMI/PEBS handling.
    * In standard KVM, perf writes to the virtual MSR and expects to read back what it wrote, even if the underlying hardware counter does not advance.
    * Attendees strongly recommended vetting the GHCB response behavior with upstream Linux `perf` developers to ensure compatibility across guest kernel profiling frameworks.

---

### **7. Current Status**
* RFC/patch series implementing hardware-assisted PMC filtering for both standard SVM and confidential guests have been submitted upstream to the Linux Kernel Mailing List (`lore.kernel.org`).
