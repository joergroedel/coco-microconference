# Optimizing `guest_memfd` shared/private conversions ([Video](https://www.youtube.com/watch?v=4jodIc-t3vY))

Presenter: Ackerley Tng

The presentation and discussion at the Linux Plumbers Conference focused on
optimizing memory state conversions within the `guest_memfd` framework, a key
component for confidential computing (CoCo) VMs in KVM. Here is a summary of
the session:

## Core Problem: The High Cost of Folio Splitting

The current implementation of `guest_memfd` relies on splitting "folios"
(contiguous groups of memory pages) when converting memory between private and
shared states. This process is computationally and memory-intensive, involving:

- **Splitting filemap entries** in the XArray tree.
- **Undoing HugeTLB optimizations** like `vmemmap` optimization (HVO).
- **Allocating new metadata** for individual pages and copying flags.

## The Proposal: Shift to Mapping and User Management

Ackerley Tng from Google proposed a more efficient approach that avoids folio
restructuring altogether. The core idea is to treat `guest_memfd` purely as a
manager of memory mappings and a tracker of memory users.

### 1. Managing Mappings

Instead of physically splitting memory, `guest_memfd` would dynamically update
various page tables during conversions:

- **Stage 2 Page Tables:** Directing KVM on how to map guest memory.
- **Kernel Direct Map:** Removing or restoring host kernel access to pages.
- **Host Userspace Page Tables:** Managing visibility for the host process.
- **IOMMU Page Tables:** Ensuring devices cannot perform DMA (Direct Memory
  Access) to private pages.

### 2. Managing Users and Avoiding GUP

A major bottleneck is the use of "Get User Pages" (GUP), which pins pages in
memory and prevents certain optimizations. The proposal suggests:

- **Using `VM_PFNMAP` VMAs:** These specialized memory areas don't support GUP,
  forcing a transition to more efficient memory access methods.
- **Potentially dropping `struct page` tracking:** For memory managed by
  `guest_memfd`, removing standard page metadata could significantly reduce
  memory overhead and simplify state tracking.

## Discussion Highlights

The subsequent discussion with the community brought forward several critical
points:

- **Architectural Differences in Memory Protection:** Participants noted that
  different hardware architectures handle unauthorized memory access differently.
  For instance, Intel TDX can trigger a machine check exception—potentially
  crashing the host—if a device attempts to write to private memory, while AMD
  SEV-SNP may simply block the operation.
- **The Necessity of Unmapping:** There was debate over whether the host must
  always unmap memory from the IOMMU during conversions. While some argued that
  confidential hypervisors already provide isolation, the consensus favored
  unmapping as a vital "defense-in-depth" measure to protect the host from
  crashing due to malformed device requests.
- **The Role of GUP in Virtualization:** Discussion revolved around how much
  virtualization logic still relies on GUP and whether it's feasible to
  completely move away from it for `guest_memfd`. Some participants suggested
  that for shared pages, GUP might still be necessary unless specialized "bounce
  buffer" mechanisms are used.
- **User-space API and Orchestration:** Questions were raised about the best
  way for user-space tools to interact with these new mapping mechanisms,
  particularly as more I/O tasks are offloaded to specialized hardware.

## Conclusion

The session concluded with a call for further community input on how to best
integrate these mapping-based optimizations into the Linux kernel to make
confidential computing more performant and robust.
