# Consequences

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

    \[Pure capability code\] has an easier deployment path through recompilation and – for selected pieces of software only minor source-code adaptations.

    Source: `[1]`

Referential protection is achieved by integrity and provenance validity enforcement.

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

With CHERI, you can utilize one address space, offering potentially greater compartmentalization scalability. CHERI also provides resilience against (unknown) vulnerabilities in known classes (such as buffer overflows) and against future as-yet undiscovered classes of vulnerability / exploit techniques.

Compartments are constructed utilizing closed graphs of capabilities:

- bounds & permissions ensure capabilities assigned to compartments grant access only to intended resources
- monotonicity ensures rights cannot be modified to include other resources
- temporal safety ensures data & capabilities do not improperly leak between compartments when memory is freed and reused

Switching can be achieved with two architectural mechanisms for controlled [non-monotonicity](./capabilities.md#reachable-capability-monotonicity)

## Software-Stack Prototypes

Several software stacks developed and/or adjusted:

1. CHERI Clang/LLVM/LLD (see 6.1 for more info & adjustments)
2. CHERI GDB
3. CHERI microkernel (interesting!)
4. CheriBSD kernel + CheriBSD hybrid/CheriABI user space + CheriBSD applications
