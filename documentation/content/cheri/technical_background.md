# Technical Background

!!! quote

    CHERI is a hardware/software/semantics co-design project, combining architecture design, hardware implementation, adaption of mainstream software stacks, and formal semantics and proof. \[...\] Formal modeling and verification allow us to make strong claims about security properties of CHERI-enabled architectures.

    Source: `[1]`

CHERI extends conventional processor ISAs with architectural capabilities to enable fine-grained memory protection and highly scalable software compartmentalization.

The authoritative architecture reference is the [**CHERI ISA Specification**][cheri-isa-specification]. It describes the overall research approach, architecture-neutral protection model, mappings into \[...\] 32/64-bit RISC-V architectures and it provides a detailed design rationale for a number of key CHERI design choices.

[cheri-isa-specification]: https://www.cl.cam.ac.uk/techreports/UCAM-CL-TR-951.pdf

## The CHERI Architecture

CHERI is a **_hybrid_ capability architecture extension** (based on the [_Capsicum_ security model][wikipedia-capsicum]) able to cleanly integrate architectural capabilities with conventional MMU-based RISC (micro-)architectures.

A hybrid capability architecture extensions

1. allows an incremental adoption (hybrid);
2. utilizes capabilities (capability architecture);
3. is built on top of an existing computer architecture (architecture extension).

---

CHERI extends conventional ISAs that use machine words to represent language-level integers **and** pointers with a new type of hardware-supported data type: the [**architectural capability**][docs-capabilities]!

!!! note "Incremental Adoptibility"

    This approach allows incremental deployment within existing ecosystems.

By providing strong, non-probabilistic, efficient mechanisms to support the principles of least privilege (POLA) and intentional use in the execution of software at multiple levels of abstraction, one can

1. prevent and mitigate vulnerabilities;
2. enable software to efficiently implement fine-grained memory protection and scalable software compartmentalization;
3. addresses performance / robustness issues arising when trying to express more secure programming models (minimising privilege) above conventional architectures that provide only MMU-based protection.

[wikipedia-capsicum]: https://en.wikipedia.org/wiki/Capsicum_(Unix)

### Design Goals

1. Fine-Grained Memory Protection & Safety
2. Highly Scalable Software Compartmentalization = Privilege Separation
3. Minimize (Ambient) Privilege
4. Incremental Adoptability from Current ISAs & Software Stacks
    - no disruption of RISC design choices
    - composed with conventional ring-based privilege & virtual memory
5. Low Performance Overhead for Memory Protection
6. Significant Performance Improvements for Software Compartmentalization
7. Formal Grounding
8. Programmer-Friendly Underpinnings

#### Fine-Grained Memory Protection & Safety

!!! abstract "Definition: Memory Safety"

    Memory safety is the property of a program where memory pointers used always point to valid memory, i.e. allocated and of the correct type/size. Memory safety is a correctness issue – a memory unsafe program may crash or produce nondeterministic output depending on the bug.

    [Source](https://stanford-cs242.github.io/f18/lectures/05-1-rust-memory-safety.html)

Traditional programming languages, such as C, are inherently memory unsafe. Modern programming languages such as Rust are memory safe as long as you are not using its `unsafe` features. CHERI allows do this at an even lower level. CHERI's new memory protection features allow memory-unsafe programming languages (C/C++) to support strong, compatible, and efficient protection against currently widely exploited vulnerabilities and porgramming errors.

CHERI employs fine-grained memory protection of spatial, referential, and temporal memory (for memory-unsafe programming languages) by utilizing capabilities instead of integers & modest OS extensions.

#### Scalable Software Compartmentalization

!!! warning "This Section is TODO"

One can use CHERI as an alternative means to construct software isolation & controlled communication, which does not need to be strictly MMU-based (note: MMU is used because virtual memory is used).

#### Minimizing (Ambient) Privilege

CHERI allows software privilege to be minimized at two granularities:

1. **Fine-Grained Code Protection**
    - enables fine-grain protection and intentional use via POLA
    - by introducing in-address-space _memory_ capabilities replacing integer virtual addresses representations of code & data pointers
    - aim: minimize rights available to be exercised on instruction-by-instruction basis
    - protection policies can (to a large extent) be based on information already present in program descriptions
    - non-probabilistic protection against broad range of memory- and pointer-based vulnerabilities & exploit techniques (buffer overflows, format-string attacks, pointer injection, data-pointer-corruption attacks, control-flow attacks)
    - achieved through code re-compilation on CHERI
2. **Secure Encapsulation & Intentional Use**
    - coarser granularity
    - through highly scalable in-address-space software compartmentalization
    - by implementing _object_ capabilities
    - aim: minimize set of rights available to larger isolated software components, building on efficient architectural support for strong software encapsulation
    - grounded in explicit descriptions of isolation and communication

### Already Developed

CHERI provides an architecture-neutral capability-based protection model, instantiated in various commodity base architectures such as CHERI-MIPS, CHERI-RISC-V, ARM's prototype [_Morello_](./consequences.md#morello) architecture, and a sketched version of CHERI-x86-64.

---

The following work has already been done or partly finished:

1. **ISA changes**: introduce [architectural capabilities][docs-capabilities]
2. **New microarchitecture**: demonstrating that capabilities can be implemented efficiently in hardware, including support for efficient tagged memory to protect capabilities in memory, and compressed capabilities to reduce memory overhead
3. **Formal models**: of CHERI-extended ISAs (used as the architecture definition, as readable documentation, for architecture design exploration, for automatic construction of executable ISA-level simulators, for automatic test generation, and for mechanized verification of formal statements and proofs of the architecture’s security properties)

This will enable:

1. **New software construction models** that use capabilities to provide fine-grained memory protection and scalable software compartmentalization
2. **Language and compiler extensions** to use capabilities in implementing memory-safe C, and Foreign Function Interfaces (FFIs) for higher-level managed languages
3. **OS extensions** to use (and support application use of) fine-grained memory protection (spatial, referential, and (non-stack) temporal memory safety) and abstraction extensions to support scalable software compartmentalization
4. Application-level adaptations to operate correctly with CHERI memory protection and software compartmentalization

CHERI is a developed, evaluated, and demonstrated approach (through hardware-software prototypes) with a full software stack, including an adapted version of the Clang/LLVM compiler suite with support for capability-based C/C++, and a full UNIX-style OS (CheriBSD, based on FreeBSD) implementing spatial, referential, and (currently for user space) non-stack temporal memory safety.

[docs-capabilities]: ./capabilities.md

### Portability

CHERI-aware code is portable across underlying architectures, except for

- architecture-specific compiler backend code
- machine-dependent aspects of the OS kernel (e.g., early boot, context switching, exception handling)
- user space runtime (e.g., run-time linker)

### Hybrid Capability Architecture

One of the key design goals of CHERI is to support the continued use of C/C++ (or alike languages) and virtual-memory-based hypervisors, operating systems, and applications.

Several design choices stem from these goals, and extensions generally conform to architectural expectations:

1. capabilities on MMU-enabled systems describe virtual addresses
2. default data capability (DDC) constrains integer-relative memory accesses
3. program-counter capability (PCC) constrains instruction fetches

## The CHERI Microarchitecture

!!! quote

    A principal design goal of the CHERI architecture has been to add new architectural primitives with only limited impact on the overall microarchitecture of contemporary processor and memory-subsystem designs.

    Source: `[1]`

### Key Challenges

1. **Tagged Memory**: conventional DRAM does not support capability tagging
    - protection model does not require particular implementation of tagging, just that tags be suitably protected & properly coherent with the data they protect
    - -> non-uniform distribution of capabilities -> use hierarchical tag table
    - tag controllers and tag caches: a hierarchical page table is the best fit for most work (due to minimal overhead for DRAM)
2. **Capability Compression**: capabilities would be 4x (not 2x) bigger
    - capabilities consist of a series of fields of natural integer register size for the architecture
    - 3 extra virtual addresses: bottom bound, capability address, upper bound
    - exploit redundancy between these 4 addresses
    - CHERI Concentrate is the current compression scheme
3. Others
    1. Increase in bus/data path width
    2. DDC adds an extra `add` impact on critical path

But: essential elements - like pipeline structure, memory subsystem designs including caches, MMUs - retain their current structure.
