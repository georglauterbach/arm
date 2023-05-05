# Cap-Based VMs

- memory capabilities to isolate application components while supporting efficient data sharing
- without mandating ap- plication code to be capability-aware
- cVMs share a single virtual address space safely, each having only capabilities to access its own memory
- cVMs efficiently exchange data through two capability-based primitives assisted by a small trusted monitor
    - an asynchronous read/write interface to buffers shared between cVMs
    - a call interface to transfer control between cVMs
- existing cloud stacks face fundamental tension when \[services\] are compartmentalized but must communicate: they must either copy data or rely on page table modifications, both of which are expensive operations that involve a privileged intermediary, such as the hypervisor or OS kernel, and lead to coarse-grained interfaces designed around page granularity and larger TCBs

- cVMs avoid the need to port microservices to use capability instructions, circumventing compatibility problems that typically plague memory capabilities
- memory capabilities impose flexible bounds on all memory accesses, which allows software components to be isolated without page table modifications or adherence to page boundaries
- this offers a new opportunity to design memory sharing primitives between isolated compartments with zero-copy semantics.

> Using memory capabilities as part of a cloud stack for microservices, however, raises new challenges: the cloud stack must (i) support existing capability-unaware microservices without cumbersome code changes, bespoke compiler support, or manual management of capabilities across isolation bound- aries; (ii) remain compatible with existing OS abstractions, e.g., POSIX interfaces, all while keeping the TCB small; and (iii) offer efficient IPC-like primitives for otherwise untrusted components to share data safely and take advantage of the potential zero-copy sharing enabled by capabilities.

- CHERI architecture [64, 68] implements capabilities as an al- ternative to traditional memory pointers

CHERI protects capabilities by enforcing three properties:

1. Provenance validity: ensures that a capability can only be “derived”, i.e., constructed, from another valid capability, i.e., we cannot cast a sequence of bytes to a capability
2. Capability integrity: capabilities stored in memory cannot be modified (achieved through transparent memory tagging)
3. Capability monotonicity: when a capability is stored in a register, it is only possible to reduce its bounds and permissions

- CHERI ensure that compartments can coexist in the same address space, and will be isolated as long as their initial set of capabilities points to disjoint data and code in memory

- hybrid-cap code relies on two new capability registers:

1. default data capability (`DDC`)
2. program counter capability (`PCC`)

used implicitly by capability-unaware instructions.

## cVM Design

1. Separate namespaces
2. Bypassed communication
3. Low-overhead isolation
4. Compatibility

> CHERI also provides a CInvoke instruction to securely call functions using a pair of capabilities: the target function address, and an arbitrary value that is only meaningful to the callee function (e.g., an identifier for an object managed by the callee). A callee thus first “seals” the two capabilities using a CSeal instruction, passes them to any relevant callers, and unseals them when correctly called via CInvoke.

### Isolation Boundaries

Each program compartment contains the code and data of its binary, its dependencies (shared libraries), and the standard C library; the cVM also contains the library OS, which provides the OS functionality.

Isolation boundaries are enforced by giving each its own default CHERI capabilities using the pcc and dcc regis- ters (see §2.2) with non-overlapping address ranges; to allow

1. program -> libOS
2. 2 libOS -> Intravisor calls

cVMs use extra capabilities that grant controlled access to functions outside the respective compartment.

---

cVMs need to implement the equivalent of user/kernel separation using CHERI capabilities in user space

1. When loading a program,
2. a set of capabilities is therefore given to the syscall handler functions of the library OS;
3. the standard C library uses these capabilities to invoke system calls on the library OS through the CInvoke instruction;
4. while the rest of the application remains capability-unaware

The library OS has full access to the programs that it manages.

---

cVMs also need to implement the equivalent of guest/host (or VM/hypervisor) separation using CHERI capabilities in userspace.

1. When creating a cVM
2. the Intravisor installs capabilities to its own host system call handlers
3. on the new library OS instance

In turn, the library OS uses `CInvoke` to invoke Intravisor operations.

### Cap Management

The use of CHERI capabilities introduces two problems that cVMs must avoid:

1. avoiding the need for application code to become capability-aware
2. performance problems when revoking capabilities
