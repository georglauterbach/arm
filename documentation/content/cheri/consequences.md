# Consequences

This page present some of the new abilities we gain when using CHERI. It also presents work that has already been done.

## Fine-Grained Memory Protection (for Programming Languages)

With CHERI, we can adjust language runtimes using capabilities (instead of integers) to implement pointers. We can create 2 new compilation modes (with corresponding C-lang interpretations, calling conventions, ABIs, etc.):

1. **Pure-Capability Code**: all pointers are capabilities
    - ABI-disruptive as pointer size increases (changing in-memory layout of data structures)
    - additional care must be taken so that pointer values retain tags where intended
2. **Hybrid-Capability Code**: pointer types are integers by default
    - integer pointers are interpreted with respect to DDC
    - special language annotations are used for capabilities
    - mainly used for FFI

We are now going to look at pure-capability code.

!!! quote

    \[Pure capability code\] has an easier deployment path through recompilation and – for selected pieces of software \[we require\] only minor source-code adaptations.

    Source: `[1]`

### Spatial Memory Safety for Pure-Capability Code

Spatial protection properties are associated with capabilities:

1. **Integrity and provenance validity**: architecture enforces that all valid code/data pointers are derived from other valid pointers through valid transformations
2. **Bounds**: when suitably narrowed, prevent erroneous manipulation & use of pointers from accessing unintended objects
    - bounds are narrowed by software stack & capture the intent of program
    - typically automatically inserted by compiler (stack allocations or taking references to sub-objects) or runtime library (heap alloc) or kernel syscall interface (for `mmap`)
3. **Permissions**: when suitably masked, prevent pointers from being used for purposes other than for what they were intended
    - done by software stack (runtime linker & kernel)
4. **Monotonicity**: prevents broadening of bounds or increase in permissions on pointers

### Temporal Memory Safety for Pure-Capability Code

Capability metadata and protection properties directly support reliable temporal memory safety:

1. **Capability tags**: mechanism to safely and precisely search for pointers in system registers and memory
2. **Integrity & Provenance Validity**: ensures pointers cannot be improperly (re-)introduced to unallocated/reallocated memory
3. **Bonds**: pointers point to specific memory (sub-)allocation removing any source of ambiguity when analyzing heap arenas
4. **Permissions**: software-defined permission are used to distinguish different notions of pointer ownership (including by allocator)
5. **Monotonicity**: prevents pointers from having their bounds or permissions modified to include other memory allocations (or to improperly mark them as owned by mem allocator)

## Scalable Software Compartmentalization

Software compartmentalization is conventionally achieved view MMU-based address spaces (i.e. isolated processes) and communication via IPC. This suffers from scalability issues though, and the number of compartments and their communication is severly limited.

With CHERI, you can utilize one address space, offering potentially greater compartmentalization scalability. CHERI also provides resilience against (unknown) vulnerabilities in known classes (such as buffer overflows) and against future as-yet undiscovered classes of vulnerability / exploit techniques (this is achieved by a technique called a´sandboxing).

Compartments are constructed utilizing closed graphs of capabilities:

- bounds & permissions ensure capabilities assigned to compartments grant access only to intended resources
- monotonicity ensures rights cannot be modified to include other resources
- temporal safety ensures data & capabilities do not improperly leak between compartments when memory is freed and reused

Switching can be achieved with two architectural mechanisms for controlled [non-monotonicity](./capabilities.md#reachable-capability-monotonicity)

## Software-Stack Prototypes

Several software stacks have been developed and/or adjusted with CHERI:

1. CHERI Clang/LLVM/LLD (see 6.1 for more info & adjustments)
2. CHERI GDB
3. CHERI microkernel (interesting!)
4. CheriBSD kernel + CheriBSD hybrid/CheriABI user space + CheriBSD applications

!!! quote "CAP-VMs: Capability-Based Isolation and Sharing in the Cloud"

    We ask the question “if the hardware supported dynamic, low-overhead sharing of arbitrary-sized memory regions between otherwise isolated regions, how would this impact the cloud stack design?” We exploit hardware support for memory capabilities [23, 70], which impose flexible bounds on all memory accesses, allowing components to be isolated without page table modifications or adherence to page boundaries.

## Operating Systems

### Design Goals

Several design goals have been proposed:

1. Support strong spatial, referential, and (non-stack) temporal memory safety.
2. Implement the concept of an _abstract capability_ throughout kernel and user space.
3. Utilize capability intentionality to limit confused-deputy attacks.
4. Illustrate the potential adoption benefits of CHERI’s hybrid capability model by showing how differing degrees of CHERI integration can co-exist within a single system.
5. Utilize the intra-address-space protection properties of CHERI to support single-address space software compartmentalization.
6. ... while simultaneously avoiding substantial disruption to key OS design choices, APIs, file formats, management models, etc.

!!! quote

    A key goal has been to utilize CHERI’s hybrid model by continuing to support MMU-based software structures, such as the conventional UNIX supervisor protection and process models, while enabling varying degrees of capability use in the operating system and application.

    Source: `[1]`

### Required Changes

Baseline CHERI support required the following general classes of kernel changes:

1. Initialize CHERI support in early architecture-dependent boot.
2. Implement capability-aware context switching for kernel and user threads, saving and restoring general-purpose and selected special-purpose capability registers.

One might also want to

- extend the kernel debugger to support debugging capability-related state;
- extend signal delivery to report CHERI-related exceptions, and also provide CHERI-related diagnostics including a capability-extended register frame;
- correct kernel memory-safety bugs discovered as a result of pure-capability testing.

??? note "Adjustments of User and Kernel Space Running on Big, Monolithic Kernels"

    Moreover, with non-microkernels, i.e. bigger, monolithic kernels, we want to

    - maintain tags in user and kernel virtual memory, ensuring that when pages undergo copy-on-write, tags are swapped to disk, and so on, tags are retained;
    - ensure tags are not retained where to do so would negatively impact security (for memory mappings of files, when copying packet data, etc.);
    - implement both hybrid-capability and pure-capability versions of the kernel, supporting varying degrees of kernel memory safety and C-language models:
        - pure-capability kernel: retain tags across in-kernel memory copies, so that kernel pointers remain tagged (e.g., during structure data copies); better differentiate address and pointer types in virtual-memory subsystem; refine bounds and permissions on kernel pointers during kernel run-time linking and kernel memory allocation;
        - hybrid-capability kernel: annotate system-call argument pointer types as capabilities to maintain intentionality;
    - implement user space debugger extensions to support debugging capability-based applications, including extensions to `pointerace`(2), `truss`(1), and also to the core-dump format (e.g., to save capability register values and memory tags);
    - implement CheriABI, a pure-capability user space process environment in which all pointers (explicit or implied) are implemented using capabilities, including all system-call arguments;
    - implement support for (non-stack) temporal memory safety using sweeping revocation for user memory and user pointers stashed in the kernel on behalf of user processes.

    In full-fledged operating systems with a proper user space, the following further changes should be made to the user space code base:

    - build two user space library and application environments, one for the hybrid-capability ABI, and the other for the pure-capability ABI;
    - implement hybrid-capability and pure-capability C startup code to suitably set up user space execution environment following `execve`(2);
    - extend the run-time linker to support execution of pure-capability binaries (includes parsing metadata emitted by the compiler/linker & using it to initialize bounded data and code pointers, as well as to provide isolation between shared objects) [...];
    - modify `libc` to preserve tags across memory copies, as well as memory-copy-like operations such as `qsort`(3);
    - dor the pure-capability ABI: Better differentiate address and pointer types, especially with respect to use of `uintpointer_t`. In `libc` and `libpthread`, refine bounds and permissions for memory allocations;
    - implement `libcheri`, a user space compartmentalization runtime similar to a run-time linker;
    - correct user space memory-safety bugs discovered as a result of pure-capability testing.

## ARM _Morello_

!!! warning "This Section is TODO"

- extends the ARMv8.2-A architecture (execution state `aarch64` only)
- implements 129 Bit CHERI capabilities
