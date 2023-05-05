# CHERRI Cap Model

What exactly is `C0`?

### TODO

- cleanly separate and define the protection mechanism down below
- define address validation vs. pointer safety

---

- CHERI complements, rather than replaces the ubiquitous page-based protection mechanism
- deconflating data-structure protection and OS memory management

- _Mondrian_ in particular identified the conflation of protection with translation as a flaw in existing approaches:
    - paging is useful for operating systems that provide coarse-grained separation and virtualization (translation)
    - segmentation is more useful to enforce intra-program protection

- program safety and security depends on
    - enforcing pointer safety (the size and permission aspects of dynamic type safety)
    - isolation (for sandboxing or application compartmentalization)

- CHERI = hybrid capability model that blends conventional ISA and MMU design choices with a capability-system model
- key features
    - capability coprocessor (defining a set of compiler-managed capability registers holding capabilities similar to unforgeable segment descriptors)
    - tagged memory (protecting in-memory capabilities)
    - capability addressing occurs before virtual-address translation such that each process is a self-contained virtual capability system

- scalable and secure intra-address-space protection & high level of source-code and binary compatibility

## Practical memory protection requirements

Practical user space protection has several desirable properties:

1. **Unprivileged use**: Protection should be the common case and therefore should not require frequent system calls.
2. **Fine granularity**: Granularity should accommodate data structures that are small and densely packed (e.g., on-stack), or with odd numbers of bytes or words.
3. **Unforgeability**: Software should not be able to increase its permissions, accidentally or maliciously.
4. **Access control**: Hardware should enforce region permissions such as store and execute.
5. **Segment scalability**: Performance and memory storage overhead should scale gracefully with the number of protected memory regions.
6. **Domain scalability** Performance and memory storage overhead should scale gracefully with the number of protection domains and frequency of their communication
7. **Incremental deployment** Extant user space software should run without recompilation even as selected components, such as shared libraries, make use of fine-grained protection

- user space protection should exploit program knowledge to offer _pointer safety_, not just _address validity_
    - address validity models associate protection properties with regions of address space -> paged virtual memory is an address validity mechanism
    - pointer safety models associate protection properties with object references -> fat pointers are a pointer safety mechanism
    - pointer safety is more precise than address validity
    - pointer safety can distinguish between a buffer overflow and a reference to an adjacent object in memory
    - address validity, however, makes supervision more convenient due to a centralized protection table
    - address validity enables features such as efficient revocation

> We observe that pointer safety implies a segmented view of memory rather than the common flat view. It should be possible to blend these two views on memory to gain safety without unnecessarily breaking compatibility.

- most efficient unforgeable pointer safety implementations use a memory capability model
    - memory capability = unforgeable pointer that grants access to a linear range of address space
    - all memory accesses must occur through a memory capability
    - current protection domain defined by capabilities stored in registers along with all capabilities in memory reachable through those capabilities
    - allows protection to scale with memory space rather than with a fixed resource like the TLB

To meet our memory-protection requirement in a RISC memory capability architecture, we must ensure that:

1. Capability manipulation instructions are unprivileged.
2. Capabilities can span any range in the virtual address space.
3. Legacy references are supported, but are constrained by the capability memory model.

## Implementation

- CHERI capability extensions are implemented as a MIPS coprocessor, CP2
- similar to the MIPS floating-point coprocessor, CP1, the capability coprocessor holds a new register file and logic to access and update it

> The greatest challenge for a protection model is to protect memory capabilities from arbitrary manipulation (unforgeability) without appealing to the kernel (unprivileged use). This is important, as system calls remain a relatively expensive operation

- we have implemented tagged memory rather than supporting only regional separation (problematic since most contemporary programming languages allow arbitrary intermixing of pointers and data.)
- instructions that change fields in a capability must strictly reduce privilege, that is, disclaim per- missions or reduce the extent

> These restrictions allow CHERI to ensure capabilities are unforgeable. With the software unable to fabricate arbitrary memory references, a protection domain is defined by the transitive closure of memory capabilities reachable from its capability register set.

- CHERI tags physical memory, not virtual memory, and therefore maintains a single table for the entire system
- one tag bit for each 256-bit line in memory, or 4MB of tag space per gigabyte of memory
- a tag manager below the last level cache presents a 257-bit, tagged-memory interface to the CHERI cache hierarchy
- the manager associates each memory transaction with a tag from the table and ensures consistency between memory and tags
- decision to use physical – not virtual – memory for tags eliminates translation for the tag table + allows tags to accompany physical cache lines through the cache hierarchy
- prototype maintains the tag table in DRAM

- CHERI allows capability registers to contain general-purpose data, which preserves the cleared tag to prevent use as a capability

- extended version of FreeBSD enables the capability coprocessor on boot
- when first user process is created (or execve() is invoked), entire user virtual address space is delegated to the user register file
- kernel saves and restores per-thread capability-register state on context switches
- user process then manages capabilities within that space, thus restricting access
-
- Capability-aware allocators can manage memory and return capabilities in much the same way as con- ventional memory allocators. Revocation can be accomplished via zero-address-space-reuse allocators, TLB unmapping, or by a simplified version of garbage collection (made reliable by capability tags). New TLB permissions authorize capability loads and stores. The OS virtual-memory system is being extended to preserve tags for swapped pages.

- CHERI capabilities used as fat pointers avoid race conditions by updating capability fields and tags atomically

- MMUs implement only address validation, not pointer safety
- A key virtue of the MMU as an address validation approach is centralized management, which simpli- fies address-space revocation, a classic weakness of capabil- ity machines.
- the operating system can manipulate mappings of the underlying pages to enforce revocation. To facilitate this, CHERI extends page ta- ble entries with bits to to authorize capability loads and stores -> This also allows the OS to implement shared memory between processes that cannot act as a channel for passing capabilities

## Comparison

### Protection Mechanisms

- unprivileged use
- fine-grained
- unforgeable
- access control
- pointer safety
- segment scalability
- domain scalability
- incremental deployment

### Approaches

#### MMU

- MMUs map (potentially sparse) vAS into physical memory via page table
- permissions assigned at page-granularity
- kernels use MMUs to isolate processes (implementing protection domain)
- sharing by mapping same physical frame into multiple ASs

- fails most requirements for in-AS isolation
- coarse grained
- small allocations waste physical memory
- only address validation, not pointer safety

- MMU process isolation possible, but expensive due to TLB capacity limits
- thus, apps limit use of sandboxes, reducing isolation in favor of performance

- CHERI inherits all benefits of MMU approach
  - adds cap protection for principled fine-grained protection within an AS
  - **key virtue** of MMU as an address validation approach = centralized management -> simplified AS revocation (a classic weakness of cap systems)

#### _Mondrian_

- allows fine-grained mem protection to be layered on top of page-based virtual memory, facilitating multiple protection domains
- page table supplemented by word-granular in-mem protection tables containing permissions managed by a supervisor
- no user space ISA changes required
- relies on supervisor mode to maintain protection table: incurs a domain switch for each allocation and free event
- modern user space allocators now aggressively cache memory though -> impairs segmentation scalability

- provides address validation, no pointer validation
- CHERI does not require system calls to create new segments
- CHERI provides pointer safety suitable for bounds checking on densely packed (and divisible) memory locations

- protection domain scalability is limited: each domain requires its own complete protection table
- user space protection in CHERI does not involve tables; protection info embedded in pointers & protection domains very scalable + defined only by set of cap ptrs reachable by current thread

#### _Hardbound_

- HW-assisted fat-pointer model grounded in SW bounds-checking
- provides pointer safety, not just address validation
- able to enforce sub-allocations and stack variables
- maintains a shadow table of base and bounds values for each pointer-aligned virtual memory location + another table of tag bits to identify pointers (+ ideally also by a modified compiler for stack and global objects)

- Hardbound, the M-Machine, and CHERI rely on tags to robustly distinguish pointers from other data in memory
- Hardbound ptrs are forgeable: `setbounds` instruction allows arbitrary bounds & tables are accessible via virtual memory -> ptrs do not constitute protection domain
- CISC design with microcode implementation
- requires transactional memory for writing to (three) tables atomically
- retains native PTR size; executables are compatible -> incremental adoption possible

- performance is limited by TLB (sparse table access) and memory (inflated pointers) overhead
- ptr compression not always possible within ptr itself

#### iMPX

- HW assisted bounds checking similar to _Hardbound_ but with important differences
  - bounds not automatically propagated (needs instruction & new reg)
  - no compression
  - hierarchical protection table
- not used today anymore (too many flaws discovered in the design for it to be useful): memory overhead, no permission bits; require transactional memory to preserve atomicity

#### _M-Machine_

- 64-bit tagged-memory capability system design using guarded pointers to implement fine-grained memory protection for pointer safety
- PTRs unforgeable
- defines protection domain within AS + switching is supported
- depends on supervisor mode for ptr creation & manipulation
- almost 0 compatibility

- MM use MMU only for paging support, CHERI for multiple ASs & caps for protection and domain switching with AS
- MM compression to 64bit at cost of granularity
- CHERI has explicit PTR/cap conversions for legacy code
- CHERI allows cap manipulation in user mode

## Perf

- CHERI will always enforce bounds dynamically in hardware
- MIPS and CHERI execution times remain very close when data is cached, but performance degrades where pointer-size is dominant
- CHERI will benefit from capability compression
- perhaps also elision techniques, although performance is acceptable even in pointer-heavy benchmarks
- more space of processor, more log elements (32% on FPGA)

## Conclusions

- Fine-grained memory protection is vital for increasing security and robustness in contemporary software. To date, only coarse- grained MMU-based protection models have had wide impact
- Our feature comparison and limit study illustrate how protection-model design choices made by several published schemes can trade off among protection, performance, and compatibility

