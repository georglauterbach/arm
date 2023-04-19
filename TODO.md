# TODOs

## Open Issues

- KMP: why do you need _Kernel Memory Isolation_?
- define _spatial_, _temporal_ & _referential_ safety/protection: - "In addition to imposing spatial protection (bounds and permission checks) and referential protection (integrity and provenance validity enforcement), CHERI can also be used to implement strong C/C++-language temporal memory safety." what does temporal mean here?
- TCB is ever reoccurring
- what is "capability-oblivious code"
- provide more (improved, rigorous) definitions
- note that we're using GAS syntax

## Thesis TODO

- use SIUnitX for bit (and other units)
- provide better introduction, including
    - motivation
    - revised naming convention

## Reading List

1. read section in `[1]` about implementation in ARMv8
2. read `[2]` Architecture-Neutral Protection Model
3. <https://www.cl.cam.ac.uk/research/security/ctsrd/cheri/workshops/2016eurosys/>
4. ASPLOS
    1. '21 CubicleOS
    2. '22 Introduction to CHERI
5. OSDI
    1. '22 CAP-VMs: Cap-Based Isolation in the Cloud
6. GitHub: Security Analysis of CHERI ISA by MSRC
7. HUAWEI: System Partitions by Bohdan Trach
8. <https://www.thegoodpenguin.co.uk/blog/introducing-arm-morello-cheri-architecture/>
9. Morello
    1. General: <https://www.arm.com/architecture/cpu/morello?#Guide>
    2. Development Platform and Software Getting Started Guide: <https://developer.arm.com/documentation/den0132/latest/>
    3. Prototype  Arch Overview: <https://developer.arm.com/documentation/den0133/0100/Morello-prototype-architecture>
