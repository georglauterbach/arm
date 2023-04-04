---
hide:
  - navigation
---

# :door: Introduction

The ARM Architecture is integrated into a range of technologies, ranging from SoC devices (like smartphones), microcomputers, embedded devices, servers and even supercomputers. ARM exposes a common ISA & worflow to help with interoperability across different implementations of the architecture.

## What Actually Is an _Architecture_?

!!! abstract "Definition: Architecture"
    ~ is a **functional specification**. In case of the Arm architecture, it's the functional specification of a processor. It specifies how a processor will behave, i.e. what instructions it has and what the instructions do. It is **a contract between hardware and software**, specifying what functionality software can rely on the hardware to provide. Some features are optional, see [this subsection](#architecture-vs-microarchitecture).

### Architecture vs. Microarchitecture

TODO

## Profiles

Profiles allow tailoring the architecture to different use cases while sharing several base features. There are three profiles:

1. **A** (Application): high-performance | complex (operating) systems
2. **R** (Real-Time): common in infrastructure with real-time demands | networking & embedded devices
3. **M** (Microcontroller): small but highly power efficient | IoT devices
