# Title: Extending ARM CCA Device Assignment to CXL Type-2 Accelerators

### **1. Overview and Objective**
* **Speaker:** Ankit Agrawal (Sr. System Software Engineer, NVIDIA) at the Linux Plumbers Conference (Confidential Computing Microconference).
* **Topic:** Extending **Arm CCA (Confidential Compute Architecture)** Device Assignment to support **CXL Type-2 Accelerators**.
* **Context:** While Arm CCA device assignment currently focuses on standard PCIe devices using PCIe IDE (Integrity and Data Encryption), CXL Type-2 devices incorporate coherent host/device memory caching and access protocols that require architectural and Linux kernel extensions across the Host, VMM (QEMU), and Guest/Realm kernels.

---

### **2. Technical Background**
* **CXL Sub-Protocols:** 
  * `CXL.io`: PCIe-style discovery, MMIO, and DMA.
  * `CXL.cache`: Allows the device to coherently cache host system memory.
  * `CXL.mem`: Allows the CPU/host to access device-attached memory coherently.
  * *Type-2 Devices* utilize all three protocols simultaneously to provide a unified, coherent view of system and device memory.
* **Key Configuration Constructs:**
  * **HDM (Host-managed Device Memory) Decoders:** Hardware registers mapping Host/VM Physical Addresses (HPA/IPA) to Device Physical Addresses (DPA).
  * **Decoder Allocation Models:**
    * *Precommitted:* Programmed and locked by firmware/BIOS prior to OS boot.
    * *Non-precommitted:* Programmed and committed dynamically at runtime by the CXL driver.

---

### **3. Host-Side Design & Sequence**
* **TSM & RMM Registration:**
  * The Trusted Subsystem Manager (TSM) must discover coherent CXL ranges and register them with Arm’s Realm Management Monitor (RMM).
  * *Proposal:* Treat precommitted CXL coherent ranges similar to PCIe BAR resources during bus enumeration, storing them in `pci_dev->coh_resource[]`.
* **Proposed Execution Flow (Precommitted Decoders):**
  1. Firmware programs and commits the HDM decoders.
  2. Host kernel fetches the committed HPA ranges during enumeration.
  3. `sysfs` triggers a stream connect call (`RMI_PDEV_STREAM_CONNECT`) specifying coherent ranges.
  4. Device binds to VFIO and is passed through to the VM/Realm.
* **Challenge with Non-Precommitted Decoders (Probe vs. Connect Ordering):**
  * Typically, CXL region creation and HDM decoder commitment occur inside the driver probe, but the TSM needs region geometry beforehand to register the stream during connect.
  * *Discussion:* Audience members (including Jonathan Cameron) highlighted that in secure virtualization, allowing dynamic decoder reprogramming after initialization breaks security boundaries. The RMM requires decoder configuration to be locked down before devices are passed to Realms. Dynamic runtime sizing is further complicated because VFIO does not support resizable BARs.

---

### **4. VM / Guest Realm Design Choices**
* **Circular Dependency Issue:**
  * In a confidential guest (Realm), validating memory ranges with the RMM requires knowing the CXL region, but the driver probing process cannot safely run until the memory region is already accepted and trusted.
* **Resolution / Consensus:**
  * **Restrict Realms to Precommitted Decoders Only:** Firmware/VMM sets up and locks the HDM decoder window in the guest physical address space (via CFMWS) before guest OS boot.
  * Audience members strongly agreed that guest kernels should expect fixed, pre-locked decoder configurations, rejecting any non-precommitted dynamic reprogramming inside a confidential Realm.

---

### **5. Multiple Coherent Regions vs. Specification Constraints**
* **Problem:** 
  * The Arm RME/CCA specification currently defines only a single coherent region identifier (reserved at the top index, `0xF...F`), whereas CXL Type-2 hardware can contain multiple HDM decoders.
* **Proposed Workaround:** Index multiple regions downward from the highest index (leaving standard BAR indices 0–5 intact).
* **Discussion Feedback:**
  * Attendees questioned whether exposing multiple distinct coherent regions to a single guest is practically required. 
  * The consensus favored presenting a single unified contiguous window at the VMM/firmware level, failing validation if an unexpected multi-region topology is passed to a guest without specification updates.

---

### **6. Conclusion & Memorial**
* Patches are underway across the Linux CXL subsystem, VFIO-CXL support, and QEMU to implement Type-2 passthrough for precommitted flows.
* The session concluded with a memorial slide honoring the late **Dan Williams**, acknowledging his foundational guidance and contributions to Linux CXL subsystem architecture and confidential computing.
