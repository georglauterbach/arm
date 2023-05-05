# ISA v8

## TODO

Read about object caps!! (page 116)

---

## Contents

- go into detail about POLA & ambient authority
- second guiding principle is the **principle of intentional use**: where many privileges are available to a piece of software, the privilege to use should be explicitly named rather than implicitly selected
- allows software privilege to be minimized at two granularities:
  1. Fine-grained code protection: fine-grain protection and intentional use by introducing in-address-space memory capabilities (provides strong, non-probabilistic protection against a broad range of memory- and pointer-based vulnerabilities and exploit techniques)
  2. Secure encapsulation: intentional use through the robust and efficient implementation of highly scalable in- address-space software compartmentalization – for example, implementing object cap- abilities
- "CHERI is designed to support incremental adoption within current security-critical, C- and C++-language Trusted Computing Bases (TCBs) (like OS kernels) [...], one of the key contributions of this work is CHERI’s hybrid capability-system architecture"
- hybrid refers to combining aspects from conventional architectures, system software, and language/compiler choices with capability- oriented design
- key forms of hybridization in the CHERI design include:
  1. **RISC capability system**: capability-system model is blended with a conventional RISC user-mode architecture without disrupting the majority of key RISC design choices
  2. **MMU-enabled capability system**: cleanly and usefully composed with conventional ring-based privilege and virtual memory implemented by processor MMUs
  3. **C-language capability system**: CHERI can be targeted by a C/C++-language compiler with strong compatibility, performance, and protection properties
  4. **Hybrid system software**: CHERI supports a range of OS models including conventional MMU- based virtual-memory designs, hybridized designs that host capability-based software within multiple virtual address spaces, and pure single-address-space capability systems
  5. **Incremental adoptability**:
- CHERI is an architecture-neutral protection model **with architecture-specific map- pings** (such as CHERI-ARM)

### Design Goals

- central design goals aimed at dramatically improving the security of con- temporary C-language TCB

1. Fine-grained memory protection
2. Software compartmentalization
3. Formal modelling and verification
4. A viable transition path (Compilers must use capability-relative loads and stores for capability-aware code, but the structure (and often format) of those instructions remain the same, and although code is generated to use capabilities rather than integers for pointers, the vast major- ity of source code remains the same)

> We are concerned with satisfying the need for trustworthy systems and networks, where trustworthiness is a multidimensional measure of how well a system or other entity satisfies its various requirements – such as those for security, system integrity, and reliability, as well as human safety, and total-system survivability, robustness, and resilience, notably in the presence of a wide range of adversities such as hardware failures, software flaws, malware, accidental and intentional misuse, and so on.

### Architecture Neutrality and Architectural Instantiations

- CHERI consists of an architectural-neutral protection model, and a set of instantiations of that model across multiple ISAs.

> The aim of the CHERI protection model [...] is to support two vulnerability mitigation objectives:
>
> 1. fine-grained pointer and memory protection within address spaces;
> 2. primitives to support both scalable and programmer-friendly compartmentalization within address spaces.
>
> [...] In contrast to MMU-based protection, this is done **by protecting references to code and data (pointers), rather than the location of code and data (virtual addresses)**. This is accomplished via an in-address-space capability-system model: the architecture provides a new primitive, the **capability**, that software components (such as the OS, compiler, run-time linker, compartmentalization runtime, heap allocator, etc.) can use to implement strongly protected pointers within virtual address spaces.

- cap = tokens of authority that are unforgeable and delegatable
- CHERI capabilities are integer virtual addresses that have been extended with metadata to protect their integrity, limit how they are manipulated, and control their use
- metadata includes
  1. a **tag** implementing strong integrity protection (differentiating valid and invalid cap- abilities);
  2. **bounds** limiting the range of addresses that may be dereferenced;
  3. **permissions** controlling the specific operations that may be performed;
  4. and also sealing, used to support higher-level software encapsulation.
- mention provenance, integrity & monotonicity
- CHERI capabilities may be held in registers or in memories, and are loaded, stored, and dereferenced using CHERI-aware instructions that expect capability operands rather than integer virtual addresses
- On hardware reset, initial capabilities are made available to software via special and general-purpose capability registers
- All other capabilities will be derived from these initial valid capabilities through valid capability transformations.

> In order to continue to support non-CHERI-aware code, dereference of integer virtual ad- dresses via legacy instruction is transparently indirected via a default data capability (DDC) for loads and stores, or a program-counter capability (PCC) for instruction fetch.

- mention hybrid and pure cap mode

## Protection Model

### Underlying Principles

- POLA
    - This is expressed in terms of architectural privileges (e.g., by al- lowing restrictions to be imposed in terms of bounds, permissions, etc., encapsulating a software-selected but hardware-defined set of rights) and at higher levels of abstraction in software (e.g., by allowing sealed capabilities to refer to encapsulated code and data incorporating both a software-selected and software-defined set of rights).
- Principle of Intentional Use (POIU): When multiple rights are available to a program, the selec- tion of rights used to authorize work on behalf of the program should be explicit, rather than implicit in the architecture or another layer of software abstraction
    - avoids confused deputy

### Strong Protection of Pointers

- key purpose of the CHERI protection model is to provide architectural primitives to support strong protection for C and C++-language pointers
- typically, language-level pointer types are implemented using architectural integers in registers and in memory
- rationale:
    1. large number of vulnerabilities in Trusted Computing Bases (TCBs), and many of the application exploit techniques, arise out of bugs involving pointer manipulation, corruption, and use
        - Virtual memory fails to address these problems as (a) it is concerned with protecting data mapped at virtual addresses rather than being sensitive to the context in which a pointer is used to reference the address – and hence fails to assist with misuse of pointers; and (b) it fails to provide adequate granularity, being limited to page granularity – or even more coarse-grained “large pages” as physical memory sizes grow.
    2. Strong integrity protection, fine-grained bounds checking, encapsulation, and monoton- icity for pointers can be used to construct efficient isolation and controlled communica- tion
        - foundations on which we can build scalable and programmer-friendly compartment- alization within address spaces
        - Virtual memory also fails to address these problems, as (a) it scales poorly, paying a high performance penalty as the degree of compartmentalization grows; and (b) it offers poor programmability, as the medium for sharing is the virtual-memory page rather than the pointer-based programming model used for code and data sharing within processes

- CHERI capabilities are also targeted more broadly at compiler and language-runtime use, allowing program structure and dynamic memory allocation to dir- ect their use.
- CHERI enforces strict integrity, provenance validity, monotonicity, bounds, per- missions, and encapsulation on pointers, mitigating common vulnerabilities and exploit tech- niques.

### Architectural Capabilities

> In current systems, software typically implements pointers as integer values stored in two ar- chitectural forms: in integer registers, and in memory. Architectural capabilities are a new architectural data type likewise stored in register and memory, and containing an integer value that will most frequently be interpreted as an address. Capabilities also contain a number of other fields that contain additional metadata associated with the address, such as bounds and permissions, as well as a tag protecting their integrity.

About the tag:

> However, there is also a 1-bit tag that may be inspected via the instruction set, but is not visible via byte-wise loads and stores. This tag is used to record whether the capability is valid; it is preserved by legal capability operations but cleared by other architectural operations on that memory.

Some of CHERI’s protections are for pointers themselves (e.g., their integrity and provenance validity), whereas others are for the pointee data or code referenced by pointers (e.g., bounds and permissions). CHERI’s sealing feature protects both a pointer (via immutability) and the pointee (via non-dereferenceability).

Extending architectures with capability registers and suitable memory storage naturally aligns with many current architectural and microarchitectural design choices, as well as software- facing considerations such as compiler code generation, stack layout, operating-system beha- vior, and so on.

- Capability **tags** for pointer integrity and provenance (Section 2.3.1)
    - Each location that can hold a capability – whether a capability register or a capability-sized, capability-aligned word of memory – has an associated 1-bit tag that consistently and atomic- ally tracks capability validity for the value stored at that location
    - Tags atomically follow capabilities into and out of capability registers when their values are loaded from, or stored to, tagged memory. Stores of other non-capability types – e.g., of bytes or half words – automatically and atomically clear the tag in the destination memory location
    - allows in-memory pointer corruption by data stores to be detected on next attempted dereference
    - An untagged capability value is simply data
    - operations that dereference or otherwise use a capability require that the capability have its tag set – i.e., be a valid capability
    - valid tag is also required to use a capability to seal or unseal another capability, to jump to that capability, to use it to set the architectural compartment ID, or to call it for the purposes of domain transition
    - Valid capabilities can be constructed only by deriving them from existing valid capabilities, which ensures pointer provenance
    - (In a few cases, a capability may derive from multiple other capability values. For example, a sealed capability is derived from both the authorizing sealing capability and an original data capability. Similarly, an explicitly unsealed capability is derived from both the sealed capability and the capability that authorizes its unsealing.)
    - prototypes implement tagged memory using partitioned memory, with tags and associated capability-sized units linked close to the memory controller, and propagated by the cache hierarchy in order to provide strong atomicity with the data it protects
- Capability **bounds** to limit the dereferenceable range of a pointer (Section 2.3.2)
    - lower and upper bounds for the memory they authorize access to
    - address may move out of bounds (and perhaps back in again (required for de-facto C-lang compatibility)), attempts to dereference (e.g., via a load, store, or instruction fetch) an out-of-bounds capability will throw a hard- ware exception
    - bounds-compression scheme places restrictions on "addresses could be arbitrarily out of bounds", as bounds compression de- pends on redundancy between the address and bounds, which is reduced when addresses are substantially outside of their bounds
    - most bounds originate in the userspace language runtime or compiler-generated code, including the run-time linker for function pointers and global data, the heap allocator for pointers to heap allocations, and generated code for pointers taken to stack allocations
- Capability **permissions** to limit the use of a pointer (Section 2.3.3)
    - additionally extend addresses with a permissions mask controlling how the cap- ability may be used
    - load, store, instruction fetch, etc.
- Capability **monotonicity** and guarded manipulation to prevent privilege escalation (Section 2.3.4)
    - new capab- ilities must be derived from existing capabilities only via valid manipulations that may narrow (but never broaden) rights ascribed to the original capability
    - enforced by 4 mechanisms:
        - **limited expressivity**: some instructions are prevented, by design, from expressing an increase of rights due to the expression of their operands and implementation. For example, per- missions on capabilities are modified using a bitwise ‘and’ operation, and hence cannot express an increase in permissions.
        - **Exceptions on monotonicity violation**: Some instructions are able to represent non-monotonic operations, but attempts to use them non-monotonically will lead to an exception being delivered
        - **Stripping the tag in register write-back**: As an alternative to throwing an exception, a non- monotonic operation might succeed in writing back a new capability – but with the tag bit cleared, preventing future dereference
        - **Stripping the tag in memory store**: Tagged memory ensures that direct modification of capabilities (regardless of monotonicity) stored in memory using data store instructions will clear the tag on affected in-memory capabilities
    - As a result of these combined architectural features, guarded ma- nipulation implements non-bypassable capability monotonicity
    - > Monotonicity allows reasoning about the set of reachable rights for executing code, as they are limited to the rights in any capability registers, and inductively, the set of any rights reach- able from those capabilities – but no other rights, which would require a violation of monotonicity.
    - key foundation for fine-grained compartmentalization, as it prevents delegated rights from being used to gain access to other undelegated areas of memory
    - monotonicity contributes to the implementation of the principle of intentional use, in that capabilities not only cannot be used for operations beyond those for which they are au- thorized, but also cannot inadvertently be converted into capabilities describing more broad rights.
    - **notable exceptions**: invocation of sealed capabilities & exception delivery (Where non-monotonicity is present, control is transferred to code trusted to utilize a gain in rights appropriately)
    - non-monotonicity is required to support protection-domain transition from one domain holding a limited set of rights to destination domain that holds rights unavailable to the origin- ating domain
- Capability **sealing** to implement software encapsulation (Section 2.3.6)
    - allows capabilities to be marked as immutable and non-dereferenceable, causing hardware exceptions to be thrown if attempts are made to modify, dereference, or jump to them
    - two forms:
        1. pairs of capabilities sealed using a common object type
            - primarily designed to support the linking of a pair of code and data capabil- ities for use together during domain transition
            - jump-like instruction, CInvoke, allows the two sealed capabilities to be atomically unsealed as control flow transfers to the code pointed to by the code capability, if their object types match (valid, sealed, and have matching object types)
            - used to implement controlled priv- ilege escalation for the purposes of domain transition (e.g. in-address-space compartmentalization)
        2. stand-alone sealed entry capabilities (sentry capabilities)
            - simply seal a single code capability
            - can be jumped to leading to an atomic unsealing and control-flow transfer
            - primarily been used to strength control-flow robustness within a single protection domain by preventing the undesired manip- ulation and use of code pointers
            - Jump-and-link instructions acting on sealed entry capabilities also generate a sealed return capability
- Capability **object types** to enable a software object-capability model (Section 2.3.7)
    - additional piece of metadata, an object type, updated when a capability undergoes (un)sealing
    - allow multiple sealed capabilities to be indelibly (and indivisibly) linked
    - kernel or language runtime can avoid expensive checks to confirm that they are intended to be used together
- **Sealed capability invocation** to implement non-monotonic domain transition (Section2.3.8)
    - destination execution environment has well-defined and reliable properties (controlled target program-counter capability and additional data capability that can be used to authorize domain transition)
    - CInvoke behaves much like a conventional jump to register, permitting an in-address-space domain switch without changing rings
    - tests required by the CInvoke mechanism describe a Cartesian product of method rights (indicated by the sealed code capability) and object rights (sealed data capability) to this environment
    - Regardless of how the environment came to have these sealed capabilities, it is free to pair any sealed code capability with any sealed data capability and have the CInvoke tests pass.
- Capability **flow control** to limit pointer propagation (Section 2.3.10)
    - particularly subject to a historic criticism of capability-system models – namely, that capability propagation makes it difficult to track down and revoke rights
    - cap PTE & TLB bits: extends entries to authorize loading and storing of capabilities
    - cap load and store bits: extend the load and store permissions on capabilities themselves
    - Capability control-flow permissions:
- Capability **compression** to reduce the in-memory overhead of pointer metadata (Section 2.3.11)
- **Hybridization**
    - with integer pointers (Section 2.3.12)
        - Processors implementing CHERI capabilities also support existing programs compiled to use conventional integer pointers
        - Default Data Capability (DDC) DDC indirects and controls legacy instructions that load and store relative to integer addresses rather than capabilities.
        - Program Counter Capability (PCC) PCC extends the conventional program counter with capability metadata, indirecting and controlling instruction fetches.
    - with MMU-based virtual memory (Section 2.3.13)
        - compose naturally with, and complement, the Virtual-Memory (VM) mod- els commonly implemented using commodity Memory Management Units
        - Capabilities are within rather than between address spaces
    - with ring-based privilege (Section 2.3.14)
        - Conventional architectures employ ring-based mechanisms to control use of architectural privilege
        - use of privileged features within privileged rings, other than in accessing virtual memory as the supervisor, depends on the program-counter capability having a suitable hardware permission set
        - > This feature similarly allows code within kernels, microkernels, and hypervisors to be com- partmentalized, preventing bypass of the capability model within the kernel virtual address space through control of virtual memory features
- Failure modes and exception delivery (Section 2.3.15)
- Capability revocation (Section 2.3.16)

> While the design of CHERI capabilities is primarily focused on the protection of pointers, the pointer interpretation of capabilities depends entirely on a capability’s permissions mask. If the mask authorizes load, store, and fetch instructions, then the capability has a pointer interpretation. Capabilities are not required to have those permissions set, however, allowing capabilities to be used for other purposes – for example, to protect other critical data types from in-memory corruption

### Isolation, Controlled Communication, and Compartmentalization

In software compartmentalization, larger complex bodies of software are decomposed into multiple components that run in isolation from one another, having only selectively delegated rights to the broader application and system, and limited further attack surfaces.
Software compartmentalization is build on two primitives: software isolation and controlled communication

> It is essential to CHERI’s design that exercise of non-monotonicity support reliable transfer of control to code trusted with newly acquired rights.

### OS Support

Operating systems may be modified in a number of forms to support CHERI, depending on whether the goal is additional protection in userspace, in the kernel itself, or some combination of both. Typical kernel deployment patterns, some of which are orthogonal and may be used in combination, might be:

1. Minimally modified kernel: The kernel enables CHERI support in the processor, initializes register state during context creation, and saves/restores capability state during context switches, with the goal of supporting use of capabilities in userspace.
2. Capability domain switching in userspace: ...
3. **Fine-grained capability protection in the kernel**: kernel is extended to support fine-grained memory protection throughout its design, replacing all kernel pointers with capabilities
4. **Capability domain switching in the kernel**: Support for a capability-aware kernel is extended to include support for fine-grained, capability-based compartmentalization within the kernel itself. This in effect implements a microkernel-like model in which components of the kernel, such as filesystems, network processing, etc., have only limited access to the overall kernel environment delegated using capabilities.
5. Pure-capability operating system: Aclean-slateoperating-systemdesignmightchoosetomin- imize or eliminate MMU use in favor of using the CHERI capability model for all protec- tion and separation

## Object Caps

- two forms of capabilities:
  1. capabilities that describe regions of memory and offer bounded-buffer “segment” semantics
  2. object capabilities that permit the implementation of protected subsystems

> In our model, object capabilities are **represented by a pair of sealed code and data capabilities**, which provide the necessary information to implement a protected subsystem domain transition. Object capabilities are “invoked” using the CInvoke instruction.

## Historical Context

- capability security, microkernel OS design, and language-based constraints
    - apparently disparate areas of research are linked by a duality, observed by Jim Morris in 1973, between the enforcement of data types and safety goals in programming languages on one hand, and the hardware and software protection techniques explored in operating systems (J. H. Morris Jr. ‘Protection in programming languages’. In: Communications of the ACM 16.1 (1973), pp. 15–21. DOI: 10.1145/361932.361937.) on the other hand

### Cap Systems

- Throughout the 1970s and 1980s, high-assurance systems were expected to employ a capability- oriented design that would map program structure and security policy into hardware enforcement
- Systems such as the CAP Computer at Cambridge [165] and Ackerman’s DEC PDP-1 architecture at MIT [2] attempted to realize this vision through embedding notions of capabilities in the memory management unit of the CPU
- Levy provides a detailed exploration of segment- and capability-oriented computer system design through the mid-1980s in Capability-Based Computer Systems [78]

Dennis and Van Horn’s seminal text on capability systems [40] defines a capability as

> a structure that locates by means of [a unique code or effective name] some computing object, and indicates the actions that the computation may perform with respect to that object.

- what's a “computing object”?
  1. memory cap system: = "span of memory" => closely resemble traditional segmented memory architectures.
  2. software object cap sys: = "identified with pairs of code and private data"
  3. hardware obj cap sys: "move core aspects of the software object capability model into hardware"

### Bounds Checks & Fat Pointers

- CHERI's goal (unlike prior systems): map C-lang pointers into caps

> In later versions of the ISA, we adopt ideas from the C fat-pointer literature, which differentiate the idea of a delegated region from a current pointer: while the base and bounds are subject to guarded manipulation rules, we allow the offset to float within and beyond the delegated region. Only on dereference are protections enforced, allowing a variety of imaginative pointer operations to be supported.

### Realizing Capability Systems

#### Mem Layout

Two approaches have emerged: making the type distinction intrinsically associated with the bits in question or associating the type with the access path taken to those bits

1. "tagged architectures" with "tagged memory"
    - at least one bit is associated with a granule of memory no larger than a capability
    - indicates whether the associated granule contains capability-typed bits or data-typed bits
    - CHERI: one bit per capability-sized and suitably-aligned piece of memory

#### Indirection of Reference to Capabilities

> In a traditional von Neumann architecture, the primitive indirection mechanism is the interpretation of an in- teger as an index into memory. Virtualization of this architecture is most often accomplished by inserting a mapping function from integer “virtual addresses” to integer “physical addresses” before the latter are given to the hardware’s memory subsystem.

- many implement- ations of capability systems refer to capabilities by integers
    - capabilities available to the current process are, at least logically, enumerated in a translation table (the “C-list” of the process) and the process uses integers to index into this table
    - especially popular in software capability systems, as integers abound and a plethora of key-value mapping data structures are well-understood
    - Software cap- ability kernels can readily implement this design on behalf of their (user) programs, even on commodity hardware
    - costs:
        - while the process as a whole may satisfy the principle of least authority, the use of integer indices still presents a challenge to the principle of intensional use
        - Even when indices and offsets are maintained separately, there is nothing, architecturally, that ensures the provenance of the index as such: to confuse a program into acting using an unintended subset of its authority, it would suffice to corrupt the bits of an integer used as an index.
        - complicate sharing between processes (When sending capabilities to another process, the sender must marshal those capab- ilities into a (to be) shared segment, as the indices by which the sender refers to these capabilit- ies are useless to the recipient.)
- CHERI takes a subtly different approach, in which capabilities are loaded into CPU re- gisters. The instruction stream combines in-register capabilities with offsets to make memory accesses.

### Microkernels

> Denning has argued that the failures of capability hardware projects were classic failures of large systems projects, an underestimation of the complexity and cost of reworking an entire system design, rather than fundamental failures of the capability model [39].

- after CISC => RISC transition, attention turned to microkernels
- Carnegie Mellon’s HYDRA [26, 171] embodied this approach, in which microkernel mes- sage passing between separate tasks stood in for hardware-assisted security domain crossings at capability invocation
- HYDRA developed a number of ideas, including the relationship between capabilities and object references, refined the object-capability paradigm, and further pursued the separation of policy and mechanism
- MACH explore the decomposition of a large and decidedly un-robust operating system kernel
    - services in user space -> message passing between independent processes necessary (Lampson’s model for capability security was, in fact, based on pure message passing between isolated processes [75])
    - Unfortunately, the shift to message passing also invalidated Fabry’s semantic argument for capability systems, namely, that by offering a single namespace shared by all protection domains, the distributed system programming prob- lem could be avoided [45].
- notion that the microkernel, rather than the hardware, is responsible for implementing the protection semantics of capabilities also aligned well with the simultaneous research of RISC designs
