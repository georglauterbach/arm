# :cherries: Capability Hardware Enhanced RISC Instructions (CHERI)

!!! warning "Work In Progress"

## About

Current ISAs, and software based on such architectures, use virtual memory as a mechanism for memory protection and isolation. While this approach is widely and very successfully deployed, it has inherent limitations. Not only is the granularity at which memory is allocated bound to the page size, and with it the access control to this memory, but it can also impose substantial scalability limits on a large set of communicating processes due to its usage of dedicated address spaces.

CHERI **extends conventional processor ISAs with architectural capabilities to enable fine-grained memory protection and highly scalable software compartmentalization**. A **hybrid capability-system** approach allows architectural capabilities to be integrated cleanly with contemporary RISC architectures & microarchitectures, as well as with MMU-based C/C++- language software stacks. As a consequence, CHERI should be able to eliminate some of the shortcomings of virtual memory.

CHERI is a developed, evaluated, and demonstrated approach (through hardware-software prototypes) with a full software stack, including an adapted version of the Clang/LLVM compiler suite with support for capability-based C/C++, and a full UNIX-style OS (CheriBSD, based on FreeBSD) implementing **spatial, referential, and (currently for user space) non-stack temporal memory safety**.

Formal modeling and verification allows making strong claims about security properties of CHERI-enabled architectures. CHERI also facilitates software mitigation techniques such as sandboxing (also defends against future (currently unknown) vulnerability classes and exploit techniques). scalable compartmentalization enables fine-grained decomposition of OS (and application) code to limit the effects of security vulnerabilities to a degree unsupportable by current architectures.

## TODO

- reifies & implements "sandboxing" model from previous section
- pointers tagged with extra metadata that HW maintains & validates
- breaking out of sandbox => HW throws exception
- pointer contains slice (or range (of memory)) & actual address
- slice = pointer's sandbox
- pointers derived = equal/smaller sandbox
- 129th bit = metadata valid bit
- what is "capability-oblivious code"?
- read section in `[1]` about implementation in ARMv8
- "In addition to imposing spatial protection (bounds and permission checks) and referential protection (integrity and provenance validity enforcement), CHERI can also be used to implement strong C/C++-language temporal memory safety." what does temporal mean here?
