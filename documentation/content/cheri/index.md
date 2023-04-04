# :cherries: Capability Hardware Enhanced RISC Instructions (CHERI)

!!! warning "Work In Progress"

## About

Current ISAs, and software based on such architectures, use virtual memory as a mechanism for memory protection and isolation. While this approach is widely and very successfully deployed, it has inherent limitations. Not only is the granularity at which memory is allocated bound to the page size, and with it the access control to this memory, but it can also impose substantial scalability limits on a large set of communicating processes due to its usage of dedicated address spaces.

CHERI enhances conventional ISAs with architectural capabilities to enable fine-grained memory protection and highly scalable software compartmentalization. As a consequence, CHERI should be able to eliminate some of the shortcomings of virtual memory.

## TODO

- reifies & implements "sandboxing" model from previous section
- pointers tagged with extra metadata that HW maintains & validates
- breaking out of sandbox => HW throws exception
- pointer contains slice (or range (of memory)) & actual address
- slice = pointer's sandbox
- pointers derived = equal/smaller sandbox
- 129th bit = metadata valid bit

## Introduction

CHERI **extends conventional processor ISAs with architectural capabilities to enable fine-grained memory protection and highly scalable software compartmentalization**. A hybrid capability-system approach allows architectural capabilities to be integrated cleanly with contemporary RISC architectures & microarchitectures, as well as with MMU-based C/C++- language software stacks.
