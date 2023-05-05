# CHERI Programming Guide

## Definitions

CHERI Clang/LLVM and LLD implement the following new language, code-generation, and linkage models:

1. CHERI C/C++
    - language dialects tuned to requirements arising from implementing all pointers using CHERI capabilities
    - includes all explicit & implicit pointers
    - diverges: prevents pointers passed through integer type other than `uintptr_t` and `intptr_t` from being dereferenced
2. Hybrid C/C++
    - only selected pointers are implemented using capabilities, remainder implemented using integers
    - primarily used in systems software that bridges between environments executing pure-capability machine code and non-CHERI aware machine code
3. Pure-capability machine code
    - compiled code / hand-written assembly that utilizes CHERI capabilities for all memory accesses (rather than integers)
    - not binary compatible with cap-unaware code

## Background

- caps must be naturally aligned as that is the granularity at which in-memory tags are maintained
- the compression scheme uses a floating-point representation, allowing high-precision bounds for small objects, but requiring stronger alignment and padding for larger allocations
- architecture provides initial capabilities to the firmware, allowing data access and instruction fetch across the full address space (+ all tags are cleared in memory)
- hardware guarantees that cap tags and data is written atomically

### Rules

1. **Provenance validity** ensures that capabilities can be used – for load, store, instruction fetch, etc. – only if they are derived via valid transformations of valid capabilities. This property holds for capabilities in both registers and memory.
2. **Monotonicity** requires that any capability derived from another cannot exceed the permissions and bounds of the capability from which it was derived (leaving aside sealed capabilities, used for domain transition, whose mechanism is not detailed in this report).

### Special Memory Safety

1. **Referential safety**
    - protects pointers (references) themselves
    - includes integrity (corrupted pointers cannot be dereferenced) and provenance validity (only pointers derived from valid pointers via valid manipulations can be dereferenced)
    - capability tags and provenance validity naturally provide this protection
2. **Spatial safety**
    - pointers may be used only to access memory within bounds of their associated allocation (manipulating an out-of-bounds pointer will not grant access to another allocation)
    - accomplished by adapting various memory allocators
3. **Temporal safety**
    - prevents a pointer retained after the release of its underlying allocation from being used to access its memory if that memory has been reused for a fresh allocation
