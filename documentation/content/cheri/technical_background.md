# Technical Background

!!! quote
    CHERI is a hardware/software/semantics co-design project, combining architecture design, hardware implementation, adaption of mainstream software stacks, and formal semantics and proof.

## Memory Safety

!!! abstract "Definition: Memory Safety"
    Memory safety is the property of a program where memory pointers used always point to valid memory, i.e. allocated and of the correct type/size. Memory safety is a correctness issue – a memory unsafe program may crash or produce nondeterministic output depending on the bug.

    [Source](https://stanford-cs242.github.io/f18/lectures/05-1-rust-memory-safety.html)

Traditional programming languages, such as C, are inherently memory unsafe. Modern programming languages such as Rust are memory safe as long as you are not using its `unsafe` features. Is there a way to eliminate these shortcomings at an even lower level? Introduce: CHERI.

CHERI's new memory protection features allow memory-unsafe programming languages (C/C++) to support strong, compatible, and efficient protection against currently widely exploited vulnerabilities and porgramming errors.

## The CHERI Architecture

CHERI is a - **hybrid capability architecture extension** able to blend architectural capabilities with conventional MMU-based architectures and microarchitectures and with conventional CPU stacks based on virtual memory and C/C++.

!!! note
    This approach allows incremental deployment within existing ecosystems.

CHERI **extends conventional ISAs** which use machine words to represent language-level integers and pointers with a new type of hardware-supported data: the [**architectural capability**][docs-capabilities]!

### Already Developed

The following work has already been done or partly finished:

1. **ISA Changes**: introduce [architectural capabilities][docs-capabilities]
2. **New microarchitecture**: demonstrating that capabilities can be implemented efficiently in hardware, including support for efficient tagged memory to protect capabilities in memory, and compressed capabilities to reduce memory overhead
3. **Formal models**: of CHERI-extended ISAs (used as the architecture definition, as readable documentation, for architecture design exploration, for automatic construction of executable ISA-level simulators, for automatic test generation, and for mechanized verification of formal statements and proofs of the architecture’s security properties)

This will enable:

1. **New software construction models** that use capabilities to provide fine-grained memory protection and scalable software compartmentalization
2. **Language and compiler extensions** to use capabilities in implementing memory-safe C, and Foreign Function Interfaces (FFIs) for higher-level managed languages
3. **OS extensions** to use (and support application use of) fine-grained memory protection (spatial, referential, and (non-stack) temporal memory safety) and abstraction extensions to support scalable software compartmentalization
4. Application-level adaptations to operate correctly with CHERI memory protection and software compartmentalization

[docs-capabilities]: ./capabilities.md

### Portability

CHERI-aware code is portable across underlying architectures, except for

- architecture-specific compiler backend code
- machine-dependent aspects of OS kernel (e.g., early boot, context switching, exception handling)
- user space runtime (e.g., run-time linker)

### Hybrid Capability Architecture

!!! info

    CHERI is a hybrid capability architecture, designed to integrate a capability model with a conventional MMU-based architecture in non-disruptive and incrementally adoptable manner for software stacks.

One of the key design goals of CHERI is to support the continued use of C/C++ (or alike languages) and virtual-memory-based hypervisors, operating systems, and applications.

Several design choices stem from these goals, and extensions generally conform to architectural expectations:

1. capabilities on MMU-enabled systems describe virtual addresses
2. default data capability (DDC) constrains integer-relative memory accesses
3. program-counter capability (PCC) constrains instruction fetches

## Microarchitecture

!!! quote

    A principal design goal of the CHERI architecture has been to add new architectural primitives with only limited impact on the overall microarchitecture of contemporary processor and memory-subsystem designs.

    Source: `[1]`

Key challenges:

1. **Tagged Memory**: conventional DRAM does not support capability tagging
    - protection model does not require particular implementation of tagging, just that tags be suitably protected & properly coherent with the data they protect
    - -> non-uniform distribution of capabilities -> use hierarchical tag table
    - tag controllers and tag caches: a hierarchical page table is the best fit for most work (due to minimal overhead for DRAM)
2. **Capability Compression**: capabilities would be 4x (not 2x) bigger
    - capabilities consist of a series of fields of natural integer register size for the architecture
    - 3 extra virtual addresses: bottom bound, capability address, upper bound
    - exploit redundancy between these 4 addresses
    - CHERI Concentrate is the current compression scheme
3. others
    1. increase in bus/data path width
    2. DDC extra ADD impact on critical path

Essential elements - like pipeline structure, memory subsystem designs including caches, MMUs - retain their current structure.

## Software Models

CHERI uses capabilities for

1. **Fine-Grained Memory Protection**: of spatial, referential, and temporal memory (for memory-unsafe programming languages) by utilizing capabilities instead of integers & modest OS extensions
2. **Scalable Software Compartmentalization**: alternative means to construct software isolation & controlled communication (not strictly MMU-based (note: MMU is used because virtual memory is used))
