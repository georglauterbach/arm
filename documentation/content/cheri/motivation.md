# Motivation

There are multiple **backgrounds and motivations** to consider when asking: "Why do we need CHERI?". This page tries to shed some light on them.

## Strong Memory Protection (& Scalable Software Compartmentalization)

!!! warning "This Section is Work in Progress"

Current ISAs, and software based on such architectures, use virtual memory as a mechanism for memory protection and isolation. While this approach is widely and very successfully deployed, it has inherent limitations. Not only is the granularity at which memory is allocated bound to the page size, and with it the access control to this memory, but it can also impose substantial scalability limits on a large set of communicating processes due to its usage of dedicated address spaces.

CHERI **extends conventional processor ISAs with architectural capabilities to enable fine-grained memory protection and highly scalable software compartmentalization (= privilege separation)**. A **hybrid capability-system** approach allows architectural capabilities to be integrated cleanly with contemporary RISC architectures & microarchitectures, as well as with MMU-based C/C++- language software stacks. As a consequence, CHERI should be able to eliminate some of the shortcomings of virtual memory.

!!! quote

    There is little recent work in the area of hardware-software approaches, despite a pressing need for vulnerability mitigation in C-language Trusted Computing Bases (TCBs) such as language runtimes and web browsers, which are neither easily proven correct nor easily replaced with type-safe alternatives.

    Source: `[3]`

Capability-based systems implement the _Principle of Least Authority_ (POLA).

## Rust's Unsafe Pointers

### Theoretical Background

#### Aliasing

Aliasing is a very important concept in compilers & programming language semantics.

!!! abstract "Definition: Aliasing"
    ~ study of the observability of modifications to memory.

A pointer is basically just a nickname for memory. Aliasing's primary function is for the compiler to semantically caches memory accesses in order to either assume memory has not been modified or infer that a write is not necessary.

Variables are semantically unaliased until you take a reference to them. Accesses (we usually say pointers) **alias** if they refer to the same memory location. Aliasing of 2 pointers is unimportant when

1. only one is ever used
2. both only read

Hence, we actually only care about **accesses**. We define the following shorthands: memory is _anonymous_ if a programmer cannot refer to it by name or pointer; memory is _unaliased_ if currently exists only one way to refer to it.

#### Alias Analysis & Pointer Provenance

Aliasing rules are some of the most fundamental parts of programming language's semantics & optimizations. Violations will lead to UB! We quickly introduce two concepts:

1. Allocations
    - abstractly describe individual vars & heap allocations
    - fresh allocation (variable decl, malloc) always unaliased
2. **Pointer Provenance**
    - permission to access unaliased allocation can be delegated (e.g. by deriving a pointer from it)
    - tracking this chain of custody = pointer provenance

In a proper memory model, all accesses to allocations must have provenance tracking back to that allocation. If provenance is unsatisfied, a programmer broke out of the "sandbox" / pulled a pointer from thin air - which is very bad! Tracking provenance for compiler allows us to prove that 2 accesses don't alias: 2 pointers, separate provenance => no aliasing, good codegen.

There is a simple trick: use an analysis that answers a query with "YES", "NO", or "MAYBE", and then convert "MAYBE" to whatever is safer: "Do 2 accesses alias? MAYBE => YES" (i.e. be conservative). Hence, LLVM's `GetElementPointer (GEP)` instruction is almost always emitted with `inbounds` (i.e. “I promise this offset won’t break the pointer out of its allocation sandbox and completely trash aliasing and provenance”).

### Current Problems in Rust

#### References in Modern Rust

Reference make _very_ strong guarantees in _modern_ Rust. For a reference to (an abstract type) `T`, this includes:

1. ref is aligned
2. ref is non-null
3. pointed-to-mem is allocated & at least `size_of::<T>()` bytes
4. if T has invalid values, pointed-to-mem does not contain one

One conclusion that we can draw is to cleanly separate references and raw pointers at each API boundary!

#### The Issue

!!! quote "Integer to pointer casts are the devil!"

This is valid Rust:

```rust
// Masking off a tag someone packed into a pointer:
let mut addr = my_pointer as usize;
addr = addr & !0x1;
let new_pointer = addr as *mut T;
*new_pointer += 10;
```

For the compiler, this is (currently) absolutely fine. But it is a huge pain for people

1. defining Rust's memory model
2. trying to build sanitizers that catch UB
3. that are Rust/LLVM/C(++) compiler devs

For this to _possibly_ work with pointer provenance & alias analysis, these operations must pervasively "infect" all integers **on the assumption that they might be pointers**!

!!! note
    Setting `usize` to 128bit does not quite do the trick. Rust assumes `usize` to be pointer-sized, but it also uses `usize` to index into arrays: CHERI supports 64 bit indices (obviously), but LLVM is instructed to generate 128bit parameters (because that's `usize`'s size). If you don''t make `usize` 128bit (but only `64bit`), `usize as *mut T` does not make sense anymore.

### Introduce CHERI (?)

CHERI actually introduces [**a special function**](https://www.cl.cam.ac.uk/techreports/UCAM-CL-TR-947.pdf#page=28):

```c
void* cheri_address_set(void* capability, vaddr_t address)
```

where `vaddr_t` is like `size_t` "address-space sized pointers"; CHERI does not allow simple casts from addresses to non-address-types.

This operation is useful for provenance as well; by associating int-to-pointer operation with an existing pointer, we reestablish the provenance chain of custody. The new pointer is derived from the old one.

---

So **the solution** is to define a distinction between pointers & addresses! We need to redefine `usize` as address-sized, which is `<=` pointer-sized (and usually `==` (not for CHERI)), and define `pointer.addr() -> usize` and `pointer.with_addr(usize) -> pointer` methods. Then deprecate `usize as pointer` and `pointer as usize`.

We see that we do not need CHERI here; but this is not to say CHERI is useless - it can and should be used alongside these new methods to further improve safety and security. The two solutions are orthogonal!

### Sources

1. "Rust's Unsafe Pointer Types Need An Overhaul", 4 Apr 2023, <https://faultlore.com/blah/fix-rust-pointers/>
2. "Attempting To Understand Stacked Borrows", 4 Apr 2023, <https://rust-unofficial.github.io/too-many-lists/fifth-stacked-borrows.html>
