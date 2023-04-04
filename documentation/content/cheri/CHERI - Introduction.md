# An Introduction to CHERI

This report is a high-level introduction to CHERI. The report describes our architectural approach, CHERI’s key microarchitectural implications, our approach to formal modeling and proof, the CHERI software model, our software-stack prototypes, further reading, and potential areas of future research.

---

## Abstract

- capabilities = unforgeable tokens of authority, used to implement
  - explicit pointers (declared in the language)
  - implied pointers (used by runtime and generated code) in C and C++ (for C/C++ memory protection: directly mitigates broad range of known vulnerability types & exploit techniques)
- support for more scalable software compartmentalization facilitates software mitigation techniques such as sandboxing (also defends against future (currently unknown) vulnerability classes and exploit techniques)
- developed, evaluated, and demonstrated approach (through hardware-software prototypes, including multiple CPU prototypes, and) a full software stack; including
  - adapted version of the Clang/LLVM compiler suite with support for capability-based C/C++
  - full UNIX-style OS (CheriBSD, based on FreeBSD) implementing spatial, referential, and (currently for user space) non-stack temporal memory safety
- formal modeling and verification allow making strong claims about security properties of CHERI-enabled architectures

---

## Introduction

!!! definition "Memory Safety"

    Memory safety is the property of a program where memory pointers used always point to valid memory, i.e. allocated and of the correct type/size. Memory safety is a correctness issue – a memory unsafe program may crash or produce nondeterministic output depending on the bug.

    <https://stanford-cs242.github.io/f18/lectures/05-1-rust-memory-safety.html>

- new memory protection features allow memory-unsafe programming languages (C/C++) to support strong, compatible, and efficient protection against currently widely exploited vulnerabilities
- **scalable compartmentalization enables fine-grained decomposition of OS** (and application) **code** to limit the effects of security vulnerabilities to a degree **unsupportable by current architectures**

- **hybrid capability architecture extension**
  - able to blend architectural capabilities with conventional MMU-based architectures and microarchitectures
  - and with conventional CPU stacks based on virtual memory and C/C++
- **approach allows incremental deployment within existing ecosystems**

Already developed:

1. ISA changes: introduce architectural capabilities (= hardware-supported descriptions of permissions that can be used, in place of integer addresses, to refer to data, code, and objects in protected ways) (see [Chapter 2](#the-cheri-architecture))
2. New microarchitecture (demonstrating that capabilities can be implemented efficiently in hardware, including support for efficient tagged memory to protect capabilities in memory, and compressed capabilities to reduce memory overhead) (see [Chapter 3](#cheri-microarchitecture))
3. Formal models of CHERI-extended ISAs (used as the architecture definition, as readable documentation, for architecture design exploration, for automatic construction of executable ISA-level simulators, for automatic test generation, and for mechanized verification of formal statements and proofs of the architecture’s security properties) (see [Chapter 4](#formal-modeling-and-verification))

Enables new cap-based software constructs (incrementally deployable):

1. New software construction models that use capabilities to provide fine-grained memory protection and scalable software compartmentalization (see [Chapter 5](#cheri-software-models))
2. Language and compiler extensions to use capabilities in implementing memory-safe C and C++, and Foreign Function Interfaces (FFIs) for higher-level managed languages (see [Section 6.1])
3. OS extensions to use (and support application use of) fine-grained memory protection (spatial, referential, and (non-stack) temporal memory safety) and abstraction extensions to support scalable software compartmentalization (see [Section 6.2])
4. Application-level adaptations to operate correctly with CHERI memory protection and software compartmentalization (see [Section 6.3])

**CHERI is a hardware/software/semantics co-design project, combining architecture design, hardware implementation, adaption of mainstream software stacks, and formal semantics and proof.**

---

## The CHERI Architecture

- extends conventional ISAs (which use machine words to represent language-level integers and pointers) with a new type of hardware-supported data; the **_architectural capability_**
- capabilities protect (virtual) addresses (code or data pointers)
  - source-language pointers
  - used in the underlying implementations of language features such as local and global variables, thread-local storage, return addresses, vtable pointers, inter-library linkage
- **all memory accesses (loads, stores, instruction fetch) must be authorized by a capability**
- capabilities are held in registers and in memory (as with existing kinds of hardware-supported data (integers, floats, vectors))
  - loaded, stored, and manipulated using new **_capability-aware instructions_**
  - compatibility with current software = essential
  - composes well with current RISC architectures, microarchitectures, compiler implementations, operating-system designs & application structure

Authoritative architecture reference = **CHERI ISA specification** (SOURCE: Capability Hardware Enhanced RISC Instructions: CHERI Instruction-Set Architecture (Version 7). Technical Report UCAM-CL-TR-927, University of Cambridge, Computer Laboratory, June 2019): describes overall research approach, architecture-neutral protection model, mappings into \[...\] 32/64-bit RISC-V architectures + provides detailed design rationale for number of key CHERI design choices.

### Portability

CHERI-aware code = portable across underlying architectures, except for

- architecture-specific compiler backend code
- machine-dependent aspects of OS kernel (e.g., early boot, context switching, exception handling)
- user space runtime (e.g., run-time linker)

### Capabilities

- capabilities = twice the width of native integer pointer type of the baseline architecture
- capability consists of
  - integer address of the natural size for the architecture
  - metadata, compressed to fit in the remaining bits
- cap also associated with 1-bit _validity “tag”_: value maintained in registers and memory by the architecture

Each element of cap contributes to protection model + enforced by hardware:

1. **validity tag**: tracks validity of cap
   - if invalid, capability cannot be used for load, store, instruction fetch, or other operations
   - still possible to extract fields from an invalid capability, including address
   - cap-aware instructions maintain tag (if desired) as caps are loaded / stored, + as cap fields are accessed, manipulated, and used – as long as the [rules](#architectural-rules-for-capability-use) are followed
2. **bounds**: (lower/upper) describe portion of AS to which cap authorizes loads, stores, and/or instruction fetches
3. **permissions**: (mask) controls how cap can be used (example: restricting loading/storing of data and/or capabilities; prohibiting instruction fetch)
4. **object type**: != −1 && cap == “sealed” (with this object type), i.e. cannot be modified or dereferenced
   - sealed caps: used to implement opaque pointer types
   - foundation of controlled non-monotonicity used to support fine-grained, in-address-space compartmentalization

- when in memory: valid caps must be naturally aligned as that = granularity at which in-memory tags are maintained (Section 3.1)
- partial or complete overwrites with data, rather than a complete overwrite with a valid capability, lead to the in-memory tag being cleared, preventing corrupted capabilities from later being dereferenced.
- cap bounds compression reduces mem footprint

### Architectural Rules for Capability Use

Several important security properties on changes to cap metadata!

Execution of single instructions:

1. **provenance validity**: valid caps can only be constructed by instructions that do so explicitly (from other valid caps) (applies to: in mem + reg)
2. **cap monotonicity**: when instruction constructs new cap (except in sealed cap manipulation/exception raising), it cannot exceed permissions + bounds of cap it was derived from

About executions in general:

1. **reachable capability monotonicity**: in any execution of arbitrary code, until execution is yielded to another domain, set of reachable capabilities (those accessible to the current program state via registers, memory, sealing, unsealing, and constructing sub-capabilities) cannot increase

Boot time: architecture provides initial caps to firmware (allowing: data access + instruction fetch across full AS) + all tags cleared in mem. Further caps: derived in accordance with monotonicity property (arch -> firmware -> bootloader -> hypervisor -> OS -> application). At each stage in derivation chain, bounds + permissions may be restricted to further limit access.

Caps passed in syscall: kernel may only use cap bounds:  prevents “confused deputy” problems (a more privileged party uses an excess of privilege when acting on behalf of a less privileged party, performing operations that were not intended to be authorized).

### General-Purpose Capability Registers

Caps can be held in

1. **architectural registers** extended to hold cap tag + full cap data width
2. tagged memory

When in reg: cap can be used as operands to **cap-aware instructions** that inspect, manipulate, dereference, otherwise operate on caps.

~ can be implemented in two ways with respect to the general-purpose register file:

1. **split cap reg file**: introduces new general-purpose cap register file (in style of a floating-point register file, that complements existing general-purpose integer register files)
2. **merged cap reg file**: extends existing general-purpose integer registers to include tag + additional width required to hold caps

- both work; different: architectural, microarchitectural, memory-footprint, software-stack tradeoffs.
- caps move between regs & mem: tags track flow of valid (uncorrupted) caps through system (controlling future use of cap values) (tagging cap registers themselves, not just mem locations: allows implementation of cap-oblivious code)

### Special Capability Registers

- some special-purpose regs require extensions (to cap width) (`%pc` -> `%pcc` program counter cap, `%epc` -> `epcc`)
- some entirely new cap-width special-purpose regs required too (default data cap `%ddc`:  automatically indirects and controls all integer-relative loads and stores, allowing non-cap-aware code to be constrained using a cap)

### Capability-Aware Instructions

1. **retrieve capability fields**: retrieve integer values for various cap fields (including its tag, address, permissions, object type) (generally includes conditional move and comparison instructions for certain fields to improve the density of generated code)
2. **manipulate capability fields**: set or modify, subject to monotonicity, various cap fields (including address, permissions, object type) (includes cap pointer arithmetic instructions)
3. **load or store via capabilities**: load integer, cap, or other values via suitably authorized cap (may include instructions to access data relative to the program counter cap)
4. **Control flow**: perform jump or jump-and-link-register to cap destination.
5. **Special capability registers**: retrieve and set values of special cap registers – e.g., of the exception program-counter cap (`%epcc`) during exception handling
6. **Compartmentalization**: support fast protection-domain transitions (see [section 2.7](#controlled-non-monotonicity))

Important aspect: **instructions expect either cap OR integer, NEVER dynamically select one** interpretation or another based on tag value.

### Controlled Non-Monotonicity

Cap monotonicity: prevents new capability values with greater rights from being derived from prior capability values with fewer rights. = essential foundation for software compartmentalization. Some legitimate use cases where monotonicity might prevent current design patterns: mem allocation. Re-derivation = preferred solution: keep more privileged cap and derive one more.

But 3 legit use cases:

1. **exception handling**: when exception is thrown, existing architectural mechanism performs ring transition and transfers control to well-defined (+ protected) vector; suitably privileged code also gains access to additional capability registers providing additional rights to exception handler, which may be distinct from those held by the interrupted code; typically used to grant exception handler access to data and further kernel capabilities
2. **CCall to sealed caps**: new control flow instruction `CCall`; compare two sealed operand regs; if same obj type -> unseal & install first in `%pcc`; transfers control to well-defined & protected vector + grants access to additional data capability
3. **Jump to sentry cap**: similar to 2. (allows domain transition to be implemented without requiring the use of exceptions or ring transitions)

### Hybrid Capability Architecture

- key design goal: support continued use of C/C++-language and virtual-memory-based hypervisors, operating systems, and applications
- ~ = hybrid cap architecture ( designed to integrate a capability model with a conventional MMU-based architecture in non-disruptive and incrementally adoptable manner for software stacks)

Several design choices stem from these goals:

1. caps on MMU-enabled systems describe virtual addresses
2. default data capability (DDC) constrains integer-relative memory accesses
3. program-counter capability (PCC) constrains instruction fetches

~ extensions generally conform to architectural expectations

### CHERI in Specific Architectures

Talks about implementation in MIPS & ARMv8 & RISC-V

## CHERI Microarchitecture

> A principal design goal of the CHERI architecture has been to add new architectural primitives with only limited impact on the overall microarchitecture of contemporary processor and memory-subsystem designs.

Key challenges:

1. **tagged memory**: conventional DRAM does not support capability tagging (protection model does not require particular implementation of tagging, just that tags be suitably protected & properly coherent with the data they protect) -> non-uniform distribution of caps, use hierarchical tag table
2. **cap compression**: caps would be 4x (not 2x) (3 extra virt addr: (bottom bound, capability address, upper bound) -> exploit redundancy between these 4 addr
3. others: increase in bus/data path width; DDC extra ADD impact on critical path;

Essential elements (pipeline structure, memory subsystem designs including caches, MMUs) retain current structure.

### Tag Controllers and Tag Caches

Hierarchical page table = best fit for most work (minimal overhead for DRAM).

### Capability Compression

Architecturally, capabilities consist of a series of fields of natural integer register size for the architecture. CHERI Concentrate = current compression scheme.

## Formal Modeling and Verification

Not important here.

## CHERI Software Models

Use caps for

1. **fine-grained memory protection**: spatial, referential, and temporal memory (for memory-unsafe programming languages) by utilizing caps instead of integers + modest OS extensions
2. **scalable software compartmentalization**: alternative means to construct software isolation + controlled communication (not strictly MMU-based (note: MMU is used because virt mem is used))

### Fine-Grained Memory Protection (for Programming Languages)

- adjust language runtime: using caps (instead of ints) to implement pointers
- 2 new compilation modes (with corresponding C-lang interpretations, calling conventions, ABIs, etc.):
  - pure-cap code: all ptrs = caps; ABI-disruptiveas ptr size increases (changing in-memory layout of data structures);  additional care must be used so that pointer values retain tags where intended
  - hybrid-cap code: ptr types = ints by default (interpreted with respect to DDC); special lang annotations for using caps; used for FFI

#### Spatial Memory Safety (Pure Cap C/C++)

> [pure cap code] has an easier deployment path through recompilation and – for selected pieces of software only minor source-code adaptations

= bounds and permission checks!

referential protection = integrity and provenance validity enforcement

Spatial protection properties associated with caps

1. **integrity and provenance validity**: arch enforces that all valid code/data ptrs be derived from other valid ptrs through valid transformations.
2. **bounds**: when suitable narrowed, prevent erroneous manipulation & use of ptrs from accessing unintended objects (bounds be narrowed by software stack + capture intent of program; typically automatically inserted by compiler (stack alloc or taking refs to sub-objects) or runtime library (heap alloc) or kernel syscall interface (for `mmap`))
3. **permissions**: when suitable masked, prevent ptrs from being used for purposes other than for what they were intended; done by software stack (runtime linker & kernel)
4. **monotonicity**: prevents broadening of bounds or increase in permissions on pointers

#### Temporal Memory Safety (Pure Cap C/C++)

Capability metadata and protection properties directly support reliable temporal memory safety:

1. **cap tags**: mechanism to safely & precisely search for ptrs in sys regs and mem
2. **integrity & provenance validity**: ensures ptrs cannot be improperly (re-)introduced to unallocated/reallocated memory
3. **bonds**: ptrs point to specific mem (sub-)allocation removing source of ambiguity when analyzing heap arenas
4. **permissions**: software-defined permission used to distinguish different notions of ptr ownership (including by allocator)
5. **monotonicity**: prevents ptrs from having their bounds / permissions modified to include other mem allocations (or to improperly mark them as owned by mem allocator)

### Scalable Software Compartmentalization

- conventionally MMU-based via address spaces (isolated processes) (comm via IPC (note: shared memory?))
- provides resilience against unknown vulnerabilities in known classes (such as buffer overflows) AND against future as-yet undiscovered classes of vulnerability / exploit techniques
- suffer from scalability issues (number of compartments / their communication is severely limited)
- with CHERI: 1 AS (offering potentially greater compartmentalization scalability)
- compartments constructed utilizing closed graphs of caps
  - bounds + permissions ensure caps assigned to compartments grant access only to intended resources
  - monotonicity ensures rights cannot be modified to include other resources
  - temporal safety ensures data & caps do not improperly leak between compartments when memory is freed and reused
- switching can be achieved with two architectural mechanisms for controlled [non-monotonicity](#controlled-non-monotonicity)

## CHERI Software-Stack Prototypes

Several software stacks developed/adjusted:

1. CHERI Clang/LLVM/LLD (see 6.1 for more info & adjustments)
2. CHERI GDB
3. CHERI microkernel (interesting!)
4. CheriBSD kernel + CheriBSD hybrid/CheriABI user space + CheriBSD applications

## Operating System

Design goals:

1. support strong spatial, referential, and (non-stack) temporal memory safety
2. implement the concept of an _abstract capability_ throughout kernel and user space
3. utilize capability intentionality to limit confused-deputy attacks
4. illustrate the potential adoption benefits of CHERI’s hybrid capability model by showing how differing degrees of CHERI integration can co-exist within a single system
5. utilize the intra-address-space protection properties of CHERI to support single-address space software compartmentalization
6. ... while simultaneously avoiding substantial disruption to key OS design choices, APIs, file formats, management models, etc.

> A key goal has been to utilize CHERI’s hybrid model by continuing to support MMU-based software structures, such as the conventional UNIX supervisor protection and process models, while enabling varying degrees of capability use in the operating system and application

Baseline CHERI support required the following general classes of kernel changes:

- Initialize CHERI support in early architecture-dependent boot
- Implement capability-aware context switching for kernel and user threads, saving and restoring general-purpose and selected special-purpose capability registers
- maintain tags in user and kernel virtual memory, ensuring that when pages undergo copy-on-write, are swapped to disk, and so on, tags are retained
- ensure tags are not retained where to do so would negatively impact security (for memory mappings of files, when copying packet data, etc.)
- implement both hybrid-capability and pure-capability versions of the kernel, supporting varying degrees of kernel memory safety and C-language models
  - pure-capability kernel: retain tags across in-kernel memory copies, so that kernel pointers remain tagged (e.g., during structure data copies); better differentiate address and pointer types in virtual-memory subsystem; refine bounds and permissions on kernel pointers during kernel run-time linking and kernel memory allocation
  - hybrid-capability kernel: annotate system-call argument pointer types as capabilities to maintain intentionality
- extend kernel debugger to support debugging capability-related state
- extend signal delivery to report CHERI-related exceptions, and also provide CHERI-related diagnostics including a capability-extended register frame.
- implement user space debugger extensions to support debugging capability-based applications, including extensions to `ptrace`(2), `truss`(1), and also to the core-dump format (e.g., to save capability register values and memory tags).
- implement CheriABI, a pure-capability user space process environment in which all pointers (explicit or implied) are implemented using capabilities, including all system-call arguments.
- implement support for (non-stack) temporal memory safety using sweeping revocation for user memory and user pointers stashed in the kernel on behalf of user processes
- correct kernel memory-safety bugs discovered as a result of pure-capability testing

The following further changes were made to the user space code base:

- build two user space library and application environments, one for the hybrid-capability ABI, and the other for the pure-capability ABI.
- implement hybrid-capability and pure-capability C startup code to suitably set up user space execution environment following `execve`(2).
- extend the run-time linker to support execution of pure-capability binaries (includes parsing metadata emitted by the compiler/linker & using it to initialize bounded data and code pointers, as well as to provide isolation between shared objects) [...]
- modify `libc` to preserve tags across memory copies, as well as memory-copy-like operations such as `qsort`(3).
- For the pure-capability ABI: Better differentiate address and pointer types, especially with respect to use of `uintptr_t`. In `libc` and `libpthread`, refine bounds and permissions for memory allocations .
- implement `libcheri`, a user space compartmentalization runtime similar to a run-time linker.
- correct user space memory-safety bugs discovered as a result of pure-capability testing.

---

In addition to imposing spatial protection (bounds and permission checks) and referential protection (integrity and provenance validity enforcement), CHERI can also be used to implement strong C/C++-language temporal memory safety.
