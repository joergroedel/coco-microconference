# Title: Deferring SWIOTLB Bounce to Reduce Heavy CPU Utilization

### **1. Presentation Overview**
* **Speaker:** Ryan Afranji (Google, Confidential VM Team)
* **Topic:** *Deferring SWIOTLB Bounce to Reduce Heavy CPU Utilization* / *CoCo VM's Soft Lockups During High-Bandwidth Storage Reads*

---

### **2. The Problem: Guest Soft Lockups during Storage Reads**
* **Confidential Computing (CoCo) DMA Constraint:** In CoCo architectures (such as AMD SEV-SNP, Intel TDX, and Arm CCA), I/O devices lack access to guest memory encryption keys and cannot directly DMA into private guest memory.
* **SWIOTLB Bounce Buffers:** Storage operations use bounce buffers located in shared (unencrypted) memory via SWIOTLB. The CPU must explicitly copy and encrypt/decrypt data transferred between the unencrypted bounce buffer and the guest’s private memory.
* **IRQ Context Bottleneck:** 
  * In the NVMe driver, completing storage reads and performing this memory copy happens synchronously inside the hard IRQ context (`nvme_pci_complete_batch`).
  * NVMe pins specific completion queues and interrupt vectors to single, dedicated vCPUs to minimize lock contention.
* **Failure Mode:** Under heavy read workloads and high IOPS, completion batches become very large. Copying the data via CPU inside the interrupt handler takes so long that new completion entries arrive before the handler finishes, re-triggering the interrupt immediately. This monopolizes the pinned vCPU, starves other kernel threads, and triggers **guest soft lockups**.

---

### **3. Short-Term Mitigation vs. Long-Term Solution**
* **Short-Term Mitigation:** 
  * Increase the number of storage I/O completion queues (e.g., from 4 to 16) and reduce the queue depth (e.g., from 512 to 128 entries).
  * This spreads the interrupt load across more vCPUs (4x CPU distribution), which successfully eliminates soft lockups under current speeds.
  * *Why it fails long-term:* Hardware queues are limited, the CPU copy still occurs inside an interrupt context, and future increases in storage bandwidth will re-introduce soft lockups.
* **Proposed Long-Term Solution (Deferred Workqueue):**
  * Decouple the bounce copy from the hard IRQ handler by offloading it to a kernel workqueue.
  * Use an **unbound workqueue**, allowing the Linux scheduler to distribute the heavy memcpy workload across any available vCPU rather than pinning it to the interrupt vCPU.
  * Keeps the hard IRQ handler fast and lightweight, preventing CPU starvation.

---

### **4. Benchmark Results & Trade-Offs**
* **Throughput / IOPS:** Benchmarks across 4-vCPU and 12-vCPU configurations showed a **5% to 10% increase** in both IOPS and bandwidth.
* **Latency Regression:** Because processing is deferred rather than executed immediately in the interrupt path, read latency suffered a **~10% degradation** across percentiles (p50, p90, p99).

---

### **5. Audience Discussion and Community Feedback**
* **Conditional Enabling via Existing APIs:**
  * Attendees pointed out that there is already a DMA API that detects whether SWIOTLB is active. This can be used to register the deferred handler only when running in a bounce-buffered/untrusted environment, preventing unnecessary latency regressions in standard (non-CoCo) environments.
* **Parallels to Network Stack:**
  * An attendee noted that networking previously faced the exact same issue with high-throughput NICs overwhelming interrupt handlers, leading to the development of NAPI, softirqs, and deferred polling/workqueues. Something similar (or a dim-like mechanism) may be warranted for NVMe under heavy software-bound loads.
* **Real-World Workloads & Encryption Layers:**
  * **dm-crypt / Shared Memory:** A discussion arose regarding whether real-world applications would read raw, unencrypted data from disk into confidential memory, or if they would use full-disk encryption (`dm-crypt`) or pre-allocated shared memory (e.g., `dma-buf` / `io_uring` shared buffers).
  * If the disk is encrypted with `dm-crypt`, the page cache copy might be unified with decryption, potentially optimizing out extra copies.
  * **AI/ML Use Cases:** Another attendee countered that some major confidential computing use cases (such as proprietary AI model weights running on public datasets) do stream huge quantities of unencrypted data from disk into the guest, making fast, unencrypted-to-private memory bouncing a critical real-world bottleneck.
