# CHERI for Operating Systems

## Design Goals

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

## Required Changes

### To Kernel Space

Baseline CHERI support required the following general classes of kernel changes:

1. Initialize CHERI support in early architecture-dependent boot.
2. Implement capability-aware context switching for kernel and user threads, saving and restoring general-purpose and selected special-purpose capability registers.

One might also want to

- extend the kernel debugger to support debugging capability-related state;
- extend signal delivery to report CHERI-related exceptions, and also provide CHERI-related diagnostics including a capability-extended register frame;
- correct kernel memory-safety bugs discovered as a result of pure-capability testing.

??? note "Adjustments of User Space Running on Big, Monolithic Kernels"

    Moreover, with non-microkernels, i.e. bigger, monolithic kernels, we want to

    - maintain tags in user and kernel virtual memory, ensuring that when pages undergo copy-on-write, tags are swapped to disk, and so on, tags are retained;
    - ensure tags are not retained where to do so would negatively impact security (for memory mappings of files, when copying packet data, etc.);
    - implement both hybrid-capability and pure-capability versions of the kernel, supporting varying degrees of kernel memory safety and C-language models:
        - pure-capability kernel: retain tags across in-kernel memory copies, so that kernel pointers remain tagged (e.g., during structure data copies); better differentiate address and pointer types in virtual-memory subsystem; refine bounds and permissions on kernel pointers during kernel run-time linking and kernel memory allocation;
        - hybrid-capability kernel: annotate system-call argument pointer types as capabilities to maintain intentionality;
    - implement user space debugger extensions to support debugging capability-based applications, including extensions to `pointerace`(2), `truss`(1), and also to the core-dump format (e.g., to save capability register values and memory tags);
    - implement CheriABI, a pure-capability user space process environment in which all pointers (explicit or implied) are implemented using capabilities, including all system-call arguments;
    - implement support for (non-stack) temporal memory safety using sweeping revocation for user memory and user pointers stashed in the kernel on behalf of user processes.

### To User Space Programs

In full-fledged operating systems with a proper user space, the following further changes should be made to the user space code base:

- build two user space library and application environments, one for the hybrid-capability ABI, and the other for the pure-capability ABI;
- implement hybrid-capability and pure-capability C startup code to suitably set up user space execution environment following `execve`(2);
- extend the run-time linker to support execution of pure-capability binaries (includes parsing metadata emitted by the compiler/linker & using it to initialize bounded data and code pointers, as well as to provide isolation between shared objects) [...];
- modify `libc` to preserve tags across memory copies, as well as memory-copy-like operations such as `qsort`(3);
- dor the pure-capability ABI: Better differentiate address and pointer types, especially with respect to use of `uintpointer_t`. In `libc` and `libpthread`, refine bounds and permissions for memory allocations;
- implement `libcheri`, a user space compartmentalization runtime similar to a run-time linker;
- correct user space memory-safety bugs discovered as a result of pure-capability testing.
