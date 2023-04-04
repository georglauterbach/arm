---
hide:
  - navigation
---

# :door: Introduction

The ARM Architecture is integrated into a range of technologies, ranging from SoC devices (like smartphones), microcomputers, embedded devices, servers and even supercomputers. ARM exposes a common ISA & worflow to help with interoperability across different implementations of the architecture.

## [Architectures][arm-dev-docs--introduction]

[arm-dev-docs--introduction]: https://developer.arm.com/documentation/102404/0201?lang=en

This section provides general information about what an architecture is, what subdivisions of it exist, and how all of this is connected.

### What Actually Is an _Architecture_?

!!! abstract "Definition: Architecture"
    ~ is a **functional specification**. In case of the Arm architecture, it's the functional specification of a processor. It specifies how a processor will behave, i.e. what instructions it has and what the instructions do. It is **a contract between hardware and software**, specifying what functionality software can rely on the hardware to provide. Some features are optional, see [this subsection](#architecture-vs-microarchitecture).

The architecture specifies

1. the **instruction set**: the function of each instruction & how instructions are represented in memory (encoding)
2. the **register set**: how many, sizes, functions & initial state
3. the **exception model**: different levels of privilege, types of exceptions & what happens when taking and returning from exceptions
4. the **memory model**: memory (access) ordering & cache behavior (when software must perform explicit maintenance)
5. debug, trace & profiling: setting and triggering breakpoints & what info can be captured by trace tools and in what format

### System Architecture

Systems include more that just a processor core. ARM provides **specifications** to describe requirements for systems containing a processor.

![Architecture Pyramid Model](./images/introduction/architecture-pyramid.png){ loading=lazy }

Specifications are the **basis of software compatibility**. The system architecture consists of

1. **Component Architecture Specification**
    - first layer providing a common programmer's model to software through ISA compatibility
2. **System Architecture**, containing
    1. **Base System Architecture** (BSA)
        - describes a hardware system architecture that system software can rely on
        - covers aspects of processor and system architecture, e.g. interrupt controller, timers, other common devices
        - provides reliable platform for standard OSs, hypervisors & firmware
        - applicable across different markets and use cases
        - other standards can build on BSA to provide market-specific standardization
    2. **Base Boot Requirements** (BRR)
        - establishes firmware interface requirements, e.g. PSCI, SMCCC, UEFI, ACPI, SMBIOS
        - covers requirements for systems based on the Arm architecture and that operating systems and hypervisors can rely on
        - provides the recipes for targeting specific use cases (point 3)
    3. xBSA
        1. SBBR: Specifying UEFI, ACPI, and SMBIOS requirements to boot generic, off-the-shelf operating systems and hypervisors
        2. EBBR: Specifying, along with the EBBR specification, UEFI requirements to boot generic, off-the-shelf operating systems
        3. LBRR: ...

### Architecture vs. Microarchitecture

The architecture does _not_ tell how a processor is built or how it works. The build and design is referred to as micro-architecture.

!!! abstract "Definition: Micro-Architecture"
    ~ tells you how a particular processor works. It includes pipeline length and layout, the number and sizes of caches, cycle counts for individual instructions & which optional features are implemented.

### Profiles

Profiles allow tailoring an architecture to different use cases while sharing several base features. There are three profiles:

1. **A** (Application): high-performance | complex (operating) systems
2. **R** (Real-Time): common in infrastructure with real-time demands | networking & embedded devices
3. **M** (Microcontroller): small but highly power efficient | IoT devices

### Other Architectures

The Arm architecture is the best known, but not the only one! Similar specifications for other components on SoCs include

1. GIC: Generic Interrupt Controller
2. SMMU (sometimes IOMMU): System Memory Management Unit
3. Generic Timer
4. AMBA: Advanced Microcontroller Bus Architecture

### Development & An Example

ARMv8.2-A is version 8 of the Arm architecture, profile A, revision 2 - for short, v8.2-A. v9-A builds on v8-A & adds Scalable Vector Extensions version 2 (SVE2), Transactional Memory Extensions (TME), etc. Some of the features that were optional in Armv8-A are mandatory in ARMv9-A. Updates to the architecture are published annually, adding new instructions and features: v9.0-A aligns with v8.6-A, inheriting all features + adding new features.

## [Privilege, State & Exceptions][arm-dev-docs--exception-model]

[arm-dev-docs--exception-model]: https://developer.arm.com/documentation/102412/0103

This section introduces the exception and privilege model. Modern software is developed to be split into different modules, each with a different level of access to system and processor resources. The Arm architectures enable this split by implementing different levels of privilege.

### Exception Levels (ELs) & Exception Model

![Exception Model](./images/introduction/exception-model.svg){ loading=lazy align=right }

There exist four ELs:

1. EL0 - least privilege, called "unprivileged execution"
2. EL1
3. EL2 - provides support for processor virtualization
4. EL3 - provides support for two [security states](#security-states)

Execution can move between ELs only on _taking_ an exception, or on _returning_ from an exception! On taking an exception, the EL either increases or remains the same. The EL cannot decrease on taking an exception. On returning from an exception, the EL either decreases or remains the same. The EL cannot increase on returning from an exception. The target EL is either implicit or defined by system registers; EL0 cannot be a target EL.

### Types of Privilege & Registers

There are two types of privilege relevant to the AArch64 (see [Execution States](#execution-states) later) exception model:

1. Privilege in the memory system
2. Privilege from the point of view of accessing processor resources

Both types of privilege are affected by the current privileged EL.

In the Arm architecture, registers are split into two main categories:

1. Registers that provide system control or status reporting
2. Registers that are used in instruction processing, for example to accumulate a result, and in handling exceptions

Configuration settings for AArch64 processors are held in a series of registers known as **System Registers**. The combination of settings in the System registers defines the **current processor context**.

### Security States

There can be two security states:

1. Secure State
    1. access both Secure and the Non-secure memory address space
    2. at EL3, access all the system control resources
2. Non-Secure State
    1. access only Non-secure memory address space
    2. cannot access the Secure system control resources

### Execution States

!!! abstract "Definition: Execution State"
    ~ defines the standard width of the general-purpose register and the available instruction sets. It also affects aspects of the memory model, execution model,  virtual memory system architecture (VMSA) and how exceptions are managed (exception model).

On ARMv8 and ARMv9, two execution states are supported:

1. **AArch64**
    - features 31 64-bit general-purpose registers, with a 64-bit Program Counter (PC), Stack Pointer (SP), and Exception Link Registers (ELRs)
    - provides a single instruction set: **A64**
    - defines the ARMv8 exception model, with four Exception levels, EL0-EL3, that provide an execution privilege hierarchy
    - features 48-bit Virtual Address (VA), held in 64-bit registers. The Cortex-A57 MPCore multiprocessor VMSA maps these to 44-bit Physical Address (PA) maps
    - defines a number of elements that hold the processor state (PSTATE). The A64 instruction set includes instructions that operate directly on various PSTATE elements
    - names each System register using a suffix that indicates the lowest Exception level that the register can be accessed
2. **AArch32** (backwards-compatible with implementations of the ARMv7-A architecture profile)
    - features 13 32-bit general purpose registers, and a 32-bit PC, SP, and Link Register (LR). Some of these registers have multiple Banked instances for use in different processor modes
    - provides 32 64-bit registers for Advanced SIMD and Floating-point support
    - provides two instruction sets, **A32** and **T32**. For more information, see Instruction set state
    - provides an exception model that maps the ARMv7 exception model onto the ARMv8 exception model and Exception levels
    - features 32-bit VAs. The VMSA maps these to 40-bit PAs
    - collects processor state into the Current Processor State Register (CPSR)

The processor can move between execution states only on a change of exception level. This change is also subject to [certain rules][arm-dev-docs--execution-state-change-rules].

[arm-dev-docs--execution-state-change-rules]: https://developer.arm.com/documentation/ddi0488/d/programmers-model/armv8-architecture-concepts/rules-for-changing-exception-state?lang=en
