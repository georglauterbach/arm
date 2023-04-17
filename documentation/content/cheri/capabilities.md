# Capabilities

At the core of CHERI are its capabilities. This page introduces them and explains concepts and goals.

!!! quote

    CHERI blends traditional paged virtual memory with an in-address-space capability model that includes capability values in registers, capability instructions, and tagged memory to enforce capability integrity.

    [Source][cheri-isa-specification]

## Definition

!!! abstract "Definition: Capability"

    ~ unforgeable and delegable tokens of authority, used to implement

    - explicit pointers (declared in the language);
    - implied pointers (used by runtime and generated code) (e.g. in C for memory protection to directly mitigate a broad range of known vulnerability types & exploit techniques).

    Capabilities can describe fine-grained regions of memory, and can be substituted for data or code pointers in generated code, protecting data and improving control-flow robustness.

    Capabilities are hardware-supported descriptions of permissions that can be used, in place of integer addresses, to refer to data, code, and objects in protected ways. Basically, they are pointers tagged with extra metadata that the hardware maintains and validates.

    Breaking out of the sandbox causes the hardware to throw an exception.

## Description

Capabilities protect (virtual) addresses (code or data pointers)

1. such as source-language pointers;
2. used in the underlying implementations of language features such as local and global variables, thread-local storage, return addresses, vtable pointers, inter-library linkage.

**All memory accesses (loads, stores, instruction fetch) must be authorized by a capability!** Capabilities are held in registers and in memory (as with existing kinds of hardware-supported data (integers, floats, vectors)). They are loaded, stored and manipulated using new **capability-aware instructions**.

This composes well with current RISC architectures, microarchitectures, compiler implementations, operating-system designs & application structure.

The authoritative architecture reference is the [**CHERI ISA Specification**][cheri-isa-specification]. It describes the overall research approach, architecture-neutral protection model, mappings into \[...\] 32/64-bit RISC-V architectures and it provides a detailed design rationale for a number of key CHERI design choices.

[cheri-isa-specification]: https://www.cl.cam.ac.uk/techreports/UCAM-CL-TR-951.pdf

## Memory vs. Object Capabilities

Vulnerability mitigation is achieved through two capability-based techniques aimed at user-level C-language TCBs:

1. **Memory Capabilities**: implemented by the ISA and compiler, providing an incrementally deployable replacement for pointers within address spaces, mitigating memory-based exploits
2. **Object Capabilities**: implemented by the operating system over the memory-capability foundation, providing scalable, and likewise incrementally adoptable, software compartmentalization

## Memory Capabilities

### Visualization

![A Capability Visualized](../images/cheri/capability.png){ loading=lazy }

Capabilities are twice (actually 2x + 1) the width of native integer pointer type of the baseline architecture, and they consist of

1. an integer address of the natural size for the architecture ("**full precision address**") (light blue, upper half in the image above)
2. **metadata**, compressed to fit in the remaining bits (lower half)
3. a **validity tag** (1 Bit) (not displayed in the image above)

The validity tag is maintained in registers and in memory by the architecture.

### Components

Each element of a capability contributes to the protection model and is enforced by hardware. The elements are:

1. **Validity Tag**: tracks the validity of a capability
    - 129th Bit
    - if invalid, a capability cannot be used for load, store, instruction fetch, or other operations
    - it is still possible to extract fields from an invalid capability, including address
    - **capability-aware instructions** maintain this tag (if desired) as caps are loaded / stored, and when capability fields are accessed, manipulated, and used – as long as the [rules for capability usage](#architectural-rules-for-capability-usage) are respected
2. **Bounds**: (lower/upper) describe the portion of the address space to which a capability authorizes loads, stores, and/or instruction fetches
    - compressed to reduce the memory footprint
    - provide compromise between memory consumption & bounds precision
    - sometimes also called **Slice**
    - basically the pointer's sandbox
3. **Permissions**: (mask) controls how a capability can be used
    - examples: restricting loading/storing of data and/or capabilities; prohibiting instruction fetch
4. **Object Type**: indicates whether the capability is sealed for this object type
    - when a capability is sealed, it cannot be modified or dereferenced
    - sealed capabilities are used to implement opaque pointer types
    - foundation of controlled [non-monotonicity](#reachable-capability-monotonicity) used to support fine-grained, in-address-space compartmentalization

### Architectural Rules for Capability Usage

When capabilities are in memory, valid capabilities must be naturally aligned as that is the granularity at which in-memory tags are maintained. Partial or complete overwrites with data, rather than a complete overwrite with a valid capability, lead to the in-memory tag being cleared, preventing corrupted capabilities from later being dereferenced.

#### About Execution in General

##### Reachable Capability Monotonicity

In any execution of arbitrary code, until execution is yielded to another domain, the set of reachable capabilities (those accessible to the current program state via registers, memory, sealing, unsealing, and constructing sub-capabilities) cannot increase. I.e. it prevents new capability values with greater rights from being derived from prior capability values with fewer rights. This is an essential foundation for software compartmentalization.

At boot time, the architecture provides initial capabilities to the firmware (allowing: data access & instruction fetch across full AS), and all tags are cleared in memory. Further capabilities are derived in accordance with monotonicity property (architecture -> firmware -> bootloader -> hypervisor -> OS -> application). At each stage in the derivation chain, bounds & permissions may be restricted to further limit access.

##### Controlled Non-Monotonicity

There are some legitimate use cases though where monotonicity might prevent current design patterns from functioning properly:

1. memory allocation: the preferred solution here is re-derivation, i.e. keep a more privileged capability and derive one more
2. **exception handling**: when an exception is thrown, the existing architectural mechanism performs a ring transition and transfers control to a well-defined (and protected) vector; suitably privileged code also gains access to additional capability registers providing additional rights to exception handler, which may be distinct from those held by the interrupted code; this is typically used to grant exception handler access to data and further kernel capabilities
3. **CCall to sealed capabilities**: there is a new control flow instruction `CCall`; compare two sealed operand registers, and if they have the same object type, unseal and install first capability in `%pcc`; this transfers control to a well-defined and protected vector and it grants access to additional data capability
4. **jump to sentry capability**: similar to `CCall`, but allows domain transition to be implemented without requiring the use of exceptions or ring transitions

##### System Calls

The kernel may only use capability bounds passed into a syscall. This prevents the “confused deputy” problem, where a more privileged party uses an excess of privilege when acting on behalf of a less privileged party, performing operations that were not intended to be authorized.

#### Protection

CHERI protects capabilities by enforcing three properties:

1. **Provenance Validity**: ensures that a capability can only be derived (constructed) from another valid capability, i.e., it is not possible to cast an arbitrary byte sequence to a capability.
2. **Capability Integrity**: capabilities stored in memory cannot be modified, which CHERI achieves through transparent memory tagging (i.e. automatically taking care of the validity tag, clearing it if necessary - and doing so in a mannger that is transparent to software).
3. **Capability Monotonicity**: requires that, if a capability is stored in a register, its bounds and permissions can only be reduced, e.g., a read-only capability cannot be turned into a read-write one.

#### Single Instructions

When a capability is in a register, capabilities can be used as operands to **capability-aware instructions** that inspect, manipulate, dereference, otherwise operate on capabilities. One important aspect is that **instructions expect either a capability OR an integer, they will NEVER dynamically select one** interpretation or another based on tag value!

##### Capability-Aware Instructions

1. **Retrieve capability fields**: retrieve integer values for various capability fields (including their tag, address, permissions, object type)
    - generally includes conditional move and comparison instructions for certain fields to improve the density of generated code
2. **Manipulate capability fields**: set or modify a field
    - for various capability fields (including address, permissions, object type
    - includes capability pointer arithmetic instructions
    - subject to monotonicity
3. **Load or store via capabilities**: load integer, capability, or other values via suitably authorized capability
    - may include instructions to access data relative to the program counter capability
4. **Control flow**: perform jump or jump-and-link-register to capability destination
5. **Special capability registers**: retrieve and set values of special capability registers
    - e.g. of the exception program-counter capability (`%epcc`) during exception handling
6. **Compartmentalization**: support fast protection-domain transitions

There are two important concepts to keep in mind when using capabilities with single instructions:

##### Provenance Validity

Valid capabilities can only be constructed by instructions that do so explicitly from other valid capabilities. This applies to capabilities in memory and in registers.

##### Capability Monotonicity

When an instruction constructs a new capability (except in sealed capability manipulation/exception raising), it cannot exceed permissions and bounds of the capability it was derived from.

### Registers

Capabilities move between registers and memory. Tags keep track of the flow of valid (uncorrupted) capabilities though the system (controlling the future use of capability values). Tagging capability registers themselves, and not just memory location, allows for the implementation of capability-oblivious code.

#### General-Purpose Capability Registers

Capabilities can be held in

1. **architectural registers** extended to hold a capability tag at the full capability data width
2. tagged memory

General-purpose capability registers can be implemented in two ways with respect to the general-purpose register file:

1. **Split Capability Register File**
    - introduces new general-purpose capability register file
    - in style of a floating-point register file that complements existing general-purpose integer register files
2. **Merged Capability Register File**
    - extends existing general-purpose integer registers to include tag
    - additional width required to hold capabilities

Both of these approaches work. The differences manifest in the micro-architecture, memory footprint and software stack tradeoffs.

#### Special Capability Registers

Some special-purpose registers require extensions (to capability width): The program counter (`%pc`) becomes the program counter capability (`%pcc`). Some entirely new capability-width special-purpose registers are required too: e.g. default data capability (`%ddc`) which automatically indirects and controls all integer-relative loads and stores, allowing non-capability-aware code to be constrained using a capability.

## Object Capabilities

With object capabilities, we decompose applications into isolated components, each granted only the rights it requires to operate. The compartmentalization granularity determines the degree of program decomposition - the more fine-grained. the better the mitigation against vulnerabilities due to POLA.

!!! quote

    The clean separation of policy and mechanism in object-capability systems aligns elegantly with the RISC philosophy: with fine-grained protection “fast paths” implemented in hardware, policy definition can be left to the OS, compiler, and application.

    Source: `[3]`

The object capability model is implemented by the kernel and user space runtime, and supported by the ISA and the compiler-directed memory protection. Object encapsulation is a model for isolation, object invocation a means for controlled communication. Capability-based protection therefore implements encapsulation.

A kernel could then implement object invocations via hardware-accelerated domain transitions.

!!! quote

    CHERI supports efficient, synchronous domain switching modeled on function invocation rather than asynchronous inter-process message passing. This enables the obvious compartmentalization strategy to “cut” applications at function-call boundaries (e.g., library APIs).

    Source: `[3]`

## Pure-Capability-Systems vs. Hybrid Systems

In pure-capability-systems, everything is accessible only an associated capability. Hybrid capability systems relax this restriction by also allowing some _ambient authority_ (i.e. the ability to access arbitrary system objects).
