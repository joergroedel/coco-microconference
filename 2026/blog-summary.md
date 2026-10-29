# The Confidential Computing Microconference at Linux Plumbers 2026

The Confidential Computing Microconference at Linux Plumbers Conference 2026
focused on what it takes to make protected virtual machines fit into Linux.
Across seven sessions, presenters examined migration, device assignment, memory
sharing, storage performance, and profiling. A recurring concern was how to
keep security-sensitive operations inside a trusted boundary while giving Linux
interfaces it can manage efficiently.

The discussions also showed where architectures face common problems. Migration
needs flexible dirty-page reporting. Trusted devices complicate memory
ownership and page-size changes. Memory copying for Guest I/O can turn into a
lockups. Several sessions produced clear recommendations, while others
identified questions that need further specification work or subsystem review.

## Improving Intel TDX Linux Integration

Elena Reshetova (Intel) presented a change in how Intel Trust Domain Extensions
(TDX) handles complex services such as migration setup and attestation. Intel's
earlier Service TD design put those operations in dedicated confidential VMs,
leaving KVM and host orchestration software responsible for allocating,
scheduling, binding, and destroying them. Linux maintainers and cloud providers
considered that lifecycle management too invasive.

The replacement is **TDX Module Extensions**. To KVM and QEMU, these look like
interruptible, resumable `SEAMCALL`s. Internally, the current implementation
uses hidden TDs whose lifecycle the TDX module manages. Intended uses include a
TEE-I/O provisioning agent for secure device connections, a quoting service
moving toward DICE-based attestation, and a migration setup service. An
ephemeral vTPM remains a possible future use.

Discussion concentrated on migration transport and concurrency. Extensions do
not need their own network stack: the untrusted host transports encrypted
handshake messages, while the extension performs cryptographic negotiation,
checks policy, and installs migration keys within the trusted boundary.

Concurrency comes with an operational constraint. The host chooses a fixed
thread-pool size for each extension during module initialization. An
interrupted call keeps its slot occupied until it resumes on the same thread
context, so KVM and QEMU must track unfinished calls carefully. Runtime
resizing is currently unsupported, and larger pools consume more reserved
memory.

The architectural change already addresses the main feedback on Service TDs;
the remaining discussion clarified its resource-management trade-offs.
Reshetova's broader recommendation was to expose usable primitives, respect
existing Linux abstractions, and coordinate across vendors to avoid separate
kernel mechanisms for similar confidential-computing functions.

## Arm CCA Realm Live Migration

Mathias Brossard (Arm) examined how Realm migration under Arm Confidential
Compute Architecture (CCA) fits the proposed generic KVM live-migration API.
The initial scope is migration of architectural state—including memory, CPU
state, stage-2 page tables, and virtual SMMU state—with 4 KB page transfers.
Physical device state and larger page mappings are outside that initial scope.

CCA currently uses Service Realms at the source and destination to authenticate
one another and negotiate migration keys and policy. That creates a setup-flow
mismatch with the proposed KVM API: session establishment belongs to the
Service Realms, while the host handles initialization, tracking resources, and
transport where needed.

Dirty tracking drew the most concrete API discussion. The Realm Management
Monitor (RMM) scans protected stage-2 page tables and returns lists of dirty
guest frame numbers. Converting those lists into the bitmap expected by
`KVM_GET_DIRTY_LOG`, only for userspace to convert them back, adds unnecessary
work. Scanning inside the RMM also has a latency cost.

**The discussion favored a range-based query that returns dirty frame numbers
directly and lets userspace scan separate address ranges in parallel.**
Migration must also preserve permissions and metadata for address ranges
outside KVM memory slots, because a Realm can assign attributes to memory that
has no backing yet. Export-buffer sizes need runtime discovery to accommodate
different implementations.

Post-copy migration remains architecturally possible, but provider concerns
about reliability and network failures keep pre-copy and stop-and-copy as the
immediate focus. Brossard also clarified that exhausting the permitted
migration epochs requires aborting the migration; restarting means repeating
setup and key negotiation. The session established requirements for adapting
the API rather than reporting a finalized interface.

## CoVE in Action: Confidential VMs on RISC-V

Ruoqing He (LingCage), presenting joint work with Xiaoxia Cui (Damo Academy,
		Alibaba), described an end-to-end proof of concept for the
RISC-V Confidential VM Extension (CoVE). It uses Deployment Mode 1, where host
KVM and the Trusted Security Manager (TSM) run as peers in HS mode, separated
by hardware memory protection. An OpenSBI-based Root Domain Security Manager
handles transitions between them.

The Rust TSM, `xvm2`, manages confidential guest memory and vCPU state. The
demonstration boots one confidential VM with one vCPU, shows host accesses to
protected memory faulting, and scrubs memory before returning it to the host.
At the time of the presentation, the TSM was open source; the Linux, OpenSBI,
and QEMU changes remained out of tree.

Four integration questions shaped the discussion. First, the planned
upstreaming sequence starts with guest support before host support and
interrupt virtualization. KVM feedback treated `guest_memfd` as a prerequisite
for confidential guest memory support. Pinning vCPUs to physical harts was
considered acceptable for early RFCs, with dynamic scheduling required for a
production merge.

Second, CoVE's region-based memory sharing conflicts with Linux's page-oriented
conversions. A limit of 16 tracked regions can be exhausted by ordinary
boot-time allocations unless adjacent regions are merged. Attendees recommended
considering page-granularity sharing while the specification is still open,
aligning the architecture with existing Linux behavior.

Third, attestation is intended to reuse `/sys/kernel/config/tsm/report`, with
DICE-based Entity Attestation Tokens. Root endorsement and distribution of
reference measurements remain unresolved across RISC-V vendors. Discussion also
emphasized unique instance identifiers to protect against redirection attacks.

Finally, the original nested-virtualization-oriented NACL layout does not
naturally fit Deployment Mode 1. A proposed shared run page would exchange exit
reasons and register state between KVM and the TSM; lighter message-passing
designs are also being explored. These were directions for further design work,
not settled ABI decisions.

## SEV-TIO Implementation Challenges

Alexey Kardashevskiy (AMD) presented implementation challenges for SEV-TIO,
which enables trusted device assignment to SEV-SNP guests. The central
difficulty is coordinating CPU nested page tables, IOMMU page tables, and the
hardware-enforced Reverse Map Table (RMP). Their mappings, ownership
information, and page sizes must remain compatible as memory changes state.

One issue is distinguishing private and shared DMA. Devices with limited DMA
address widths cannot necessarily express the high encryption C-bit used by
SEV-SNP. The presented solution uses Virtual Top of Memory (vTOM): addresses
below the boundary are private, and an alias above it provides shared access.
Devices reach shared memory by adding the vTOM offset.

Discussion recommended treating vTOM as a VM-wide architectural property. VM
configuration must also account for assigned devices' DMA address widths when
choosing the guest address space and shared-memory offset. Address Translation
Services add cache-invalidation requirements when mappings change.

Page splitting was the other major concern. A guest converting a 4 KB page
inside a 2 MB mapping can force the RMP entry to split through `PSMASH`; an
active device's IOMMU mapping must change in step. AMD's Trusted Memory
Protection Module can briefly stall DMA while the structures are synchronized.

**The preferred direction was to avoid routine 4 KB splitting during active DMA
by allocating shared SWIOTLB bounce pools at boot in 2 MB-aligned chunks.**
Keeping conversions on those boundaries preserves large mappings and reduces
dependence on runtime DMA stalls. Attendees considered special allocations,
such as vCPU state pages, separately from the general DMA path.

The session also emphasized that IOMMU cache invalidation must cover changes in
permissions, page sizes, and RMP-related access checks. The recommendations
simplify the implementation, but the summary does not record an upstream
acceptance decision.

## Extending Arm CCA Device Assignment to CXL Type-2 Accelerators

Ankit Agrawal (NVIDIA) explored trusted assignment of CXL Type-2 accelerators
to Arm CCA Realms. These devices combine PCIe-style I/O with coherent caching
of host memory and coherent CPU access to device memory. That makes the
configuration of coherent address windows part of the trust problem.

The proposed host flow discovers committed Host-managed Device Memory (HDM)
decoder ranges, registers them with the RMM through the Trusted Subsystem
Manager, and then assigns the device through VFIO. One proposal stores those
coherent resources alongside PCI device resources for discovery during
enumeration.

Dynamic decoder configuration creates ordering problems. The driver may
normally create regions during probe, but secure stream registration needs the
region geometry beforehand. Inside the Realm, the guest must know and validate
the memory window before safely probing the device. Reprogramming decoders
after establishing trust would also undermine the validated configuration.

**The discussion converged on precommitted, locked decoder windows for
Realms.** Firmware and the VMM should establish the guest-visible window before
the guest boots, with no dynamic decoder reprogramming inside the Realm.

Multiple coherent regions remain a specification issue: the described CCA
interface provides a single coherent-region identifier, while devices can have
several HDM decoders. Rather than adopting the proposed additional indexing
scheme immediately, discussion favored presenting one contiguous guest-visible
window and rejecting unsupported multi-region configurations until the
specification can accommodate them.

Work is underway across Linux CXL, VFIO-CXL, and QEMU for the precommitted
flow. The session closed with a memorial honoring Dan Williams and his
contributions to Linux CXL and confidential computing.

## Deferring SWIOTLB Bounce to Reduce Heavy CPU Utilization

Ryan Afranji (Google) described guest soft lockups during high-bandwidth
storage reads. For I/O paths using shared SWIOTLB bounce buffers, data must be
copied into private guest memory. In the NVMe path presented, that copying
happens during completion processing in hard interrupt context.

Large completion batches can keep an interrupt's assigned vCPU busy long enough
for more completions to arrive immediately. The resulting interrupt load
starves other work and can trigger soft lockups. Increasing completion queues
from four to sixteen and reducing queue depth from 512 to 128 mitigated the
problem in the reported configuration, but queue availability and rising
storage bandwidth limit that approach.

The proposed longer-term change moves bounce copying to an **unbound
workqueue**, allowing the scheduler to distribute the work across available
vCPUs. Reported benchmarks on four- and twelve-vCPU guests showed 5–10% gains
in IOPS and bandwidth, alongside roughly 10% worse read latency across the
measured percentiles. The trade-off is therefore better throughput and
scheduling behavior at the cost of additional completion latency.

Attendees suggested using existing DMA API detection to enable deferral only
where bounce buffering is active. They also drew parallels with networking's
use of deferred processing under heavy interrupt load.

Workload discussion covered whether disk decryption could be combined with
copying to eliminate extra work. Others pointed to confidential AI workloads
reading public datasets as a concrete need for fast transfers from shared
buffers into private memory. The session identified useful implementation
directions, but did not record a final choice of deferral mechanism or policy.

## PMU Event Filtering for Confidential Guests

Manali Shukla and Sandipan Das (AMD) presented hardware-assisted Performance
Monitoring Unit (PMU) event filtering for SEV-ES and SEV-SNP guests. KVM's
software filtering checks guest performance-counter programming through VM
exits. Hardware filtering can let permitted writes complete directly, reducing
the cost of multiplexing events across limited counters.

In the reported benchmark, hardware assistance reduced VM exits from more than
101,000 to approximately 7,500. The confidential-guest complication is handling
rejected events: KVM cannot inspect the encrypted register state in the guest's
Virtual Machine Save Area.

The proposal uses a guest `#VC` handler and the shared Guest-Host Communication
Block (GHCB). The guest reports attempted event programming, KVM checks it
against policy and updates the hardware's allowed-event slots, and the guest
handles the response. A dynamic exchange is needed because the limited hardware
slots cannot hold every event that a guest may multiplex.

The presenters proposed fixing confidential guests' event-filter policy at
creation time. Changing policy during execution is difficult when KVM cannot
inspect active guest state. That fixed policy would still allow the hardware
slots to be updated as the guest selects different permitted events.

The main unresolved issue is guest `perf` compatibility. The proposed response
to a denied event clears its enable bit, but attendees warned that changing the
value a guest expects to read back can break profiling assumptions. **The key
recommendation was to review that behavior with upstream Linux `perf`
developers before settling the protocol.** RFC patches have been submitted,
with guest-visible MSR semantics still needing agreement.
