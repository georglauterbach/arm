# TODOs

## Open Issues

- KMP: why do you need _Kernel Memory Isolation_?
- define _spatial_, _temporal_ & _referential_ safety/protection: - "In addition to imposing spatial protection (bounds and permission checks) and referential protection (integrity and provenance validity enforcement), CHERI can also be used to implement strong C/C++-language temporal memory safety." what does temporal mean here?
- TCB is ever reoccurring
- what is "capability-oblivious code"
- provide more (improved, rigorous) definitions
- note that we're using GAS syntax

## Thesis TODO

- provide convention for word styling
    - smallcaps: proper nounds
    - italics: technical, specialized terms
    - bold:
    - underline:
- use SIUnitX for bit (and other units)
- provide better introduction, including
    - motivation
    - revised naming convention

## Reading List

1. read up on the Arm architecture
2. read section in `[1]` about implementation in ARMv8
3. read `[2]` Architecture-Neutral Protection Model
4. <https://www.cl.cam.ac.uk/research/security/ctsrd/cheri/workshops/2016eurosys/>
5. ASPLOS
    1. '21 CubicleOS
    2. '22 Introduction to CHERI
6. OSDI
    1. '22 CAP-VMs: Cap-Based Isolation in the Cloud
7. GitHub: Security Analysis of CHERI ISA by MSRC
8. HUAWEI: System Partitions by Bohdan Trach
9. <https://www.thegoodpenguin.co.uk/blog/introducing-arm-morello-cheri-architecture/>
10. Morello
    1. General: <https://www.arm.com/architecture/cpu/morello?#Guide>
    2. Development Platform and Software Getting Started Guide: <https://developer.arm.com/documentation/den0132/latest/>
    3. Prototype  Arch Overview: <https://developer.arm.com/documentation/den0133/0100/Morello-prototype-architecture>
