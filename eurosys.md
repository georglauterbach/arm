# Eurosys Workshop about Microkernels

- CHERI strongly influences by historic microkernel thinking: HYDRA, PSOS, Mach, etc
- Hybrid arch -> hybrid OS?
    - Multi-address-space OS
        - CHERI memory protection within tasks – and kernel?
        - CHERI for kernel bypass on capability operations
        - CHERI to compartmentalize within microkernel
    - Single-address-space OS
        - CHERI compartmentalization model – some MMU use
        - CHERI compartmentalization model – MMU for full-system virtualization only
        - CHERI compartmentalization model – no MMU use
- DARPA CRASH: If you could revise the fundamental principles of computer-system design to improve security... what would you change?
- De-conflate virtualization and protection
- CHERI brings
    - pointer integrity
    - bounds checking
    - permission checking
- Target C/C++-language TCBs – OS kernels, monolithic applications, language runtimes
- Valid userspace pointer set – pointers not generated using derivation rules are not part of the valid provenance tree and should not be dereferenceable
- compresses caps: Exchange bounds precision for reduced capability size
- Every valid pointer is derived from precisely one object (e.g. malloc() or stack allocation)
- Pointer arithmetic moves the offset
- Bounds are never implicitly changed

## Possible Compartments of a Microkernel

1. Boot
2. Context switch IPC
3. Kernel Space
    1. Mem allocator
    2. UART
    3. Scheduler
    4. Filesystem
    5. Core services
4. User space
    1. User binaries
    2. Other services

## Principle of least privilege

Every program and every privileged user of the system should operate using the least amount of privilege necessary to complete the job.

Saltzer 1974 - CACM 17(7)
Saltzer and Schroeder 1975 - Proc. IEEE 63(9)
Needham 1972 - AFIPS 41(1)
