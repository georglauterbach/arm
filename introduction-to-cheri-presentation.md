# CHERI

- Implemented by microarchitectural extensions to the CPU and SoC
- tagging is micro-arch
- CHERI exposes software permission bits uninterpreted by architecture

## protection semantics for pointers

Integrity and provenance validity ensure that valid pointers are derived from other valid pointers via valid transformations; invalid pointers cannot be used
• Valid pointers, once removed, cannot be reintroduced solely unless re-derived from other valid pointers
• E.g., Received network data cannot be interpreted as a code/data pointer – even previously leaked pointers
• Bounds prevent pointers from being manipulated to access the wrong object
• Bounds can be minimized by software – e.g., stack allocator, heap allocator, linker
• Monotonicity prevents pointer privilege escalation – e. g. , broadening bounds
• Permissions limit unintended use of pointers; e.g., W^X for pointers
• These primitives not only allow us to implement strong spatial and temporal memory protection, but also higher-level policies such as scalable software compartmentalization

## Lang & Runtime

- lang-level mem safety: ptrs to heap allocs, functions, globals, memory mappings, sub-objects, stack allocations
- sub-lang mem safety: return address, GOT, PLT, vararg array otrs, stack ptrs, vtable ptrs, ELF arg arg ptrs

Capabilities are refined by the kernel, run-time linker, compiler-generated code, heap allocator, ...

## compartmentalization

Usefully thought about as a graph of interconnected components, where the attacker’s goal is to compromise nodes of the graph providing a route from a point of entry to a specific target

Unlike memory protection, software compartmentalization requires careful software refactoring to support strong encapsulation, and affects the software operational model

Software compartmentalization decomposes software into isolated compartments that are delegated limited rights

Key insight: With CheriABI, we can safely colocate multiple UNIX processes within the same virtual address space using CHERI capabilities

## ARM

- since 2014
- ARMv8-A
