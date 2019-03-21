---
layout: post
title: "Introduction to AArch64 Architecture"
date:	Tue Jan 29 17:31:51 EEST 2019
---

**ARM**

ARM is a family of Reduced Instruction Set Computer (RISC) architectures for computer processors that has become the predominant CPU for smartphones.

**ARM Architecture**

Cortex-A Highest performance Optimized for rich operating systems
Application profiles implement a traditional ARM architecture with multiple modes and support a virtual memory system architecture based on an MMU(Memory management Unit). Cortex-A processors provide a range of solutions for devices that make use of a rich operating system such as Linux or Android.

Cortex-R Fast response Optimized for high-performance, hard real-time applications.
Real-time profiles implement a traditional ARM architecture with multiple modes and support a protected memory system architecture based on MPU(Memory protection Unit). The Cortex-R processors target high-performance real-time applications such as hard disk controllers.

Cortex-M Smallest/lowest power Optimized for discrete processing and micro-controller.
Micro-controller profiles implement a programmers' model designed for fast interrupt processing, with hardware stacking of registers and support for writing interrupt handlers in high-level languages. The processor is designed for integration into an FPGA and is ideal for use in very low power applications.

The latest ARM architecture, ARMv8, introduces 64-bit capability alongside the existing 32-bit mode.

ARMv8 has two execution modes A64 – 64-bit registers and memory accesses, new instruction set A32 (optional) – backwards compatible with ARMv7-A.

**AArch64 Privilege model**

There are 4 Privilege levels: PL3 – highest, PL0 – lowest

In ARMv8, the four privilege levels extend this to provide support for virtualization and security

EL0 - This is unprivileged and is used for User-land Applications.

EL1 - This is privileged and is used for running an OS Kernel like Linux.

EL2 - This has a higher level of privilege and can be used to run a hypervisor.

EL3 - This is the highest level of privilege and is used to control (and protect) access to TrustZone.

ARMv8-A provides two security states, Secure and Non-secure. The Non-secure state is also referred to as the Normal World. This enables an Operating System (OS) to run in parallel with a trusted OS on the same hardware, and provides protection against certain software attacks and hardware attacks.
EL3 is always Secure, EL2 is always Non-Secure and EL0/1 can be Secure or Non-Secure

ARM TrustZone technology enables the system to be partitioned between the Normal and Secure worlds.

**TrustZone**

A Trusted Execution Environment (TEE) is a secure area inside a main processor, it runs in parallel of the operating system, in an isolated environment, and it guarantees that the code and data loaded in the TEE are protected.

**AArch64 Registers**

General purpose Registers
	
    Wn	32-bits	General purpose registers 0-31
    Xn	64-bits	General purpose registers 0-31
    WZR	32-bits	Zero register (Reads as 0, writes are ignored, a way to ignore results)
    XZR	64-bits	Zero register (Reads as 0, writes are ignored, a way to ignore results)
    SP	64-bits Stack pointer

    X0 – X7    arguments and return value
    X8 – X18   temporary registers caller-saved
    X19 – X28  callee-saved registers
    X29        frame pointer
    X30        link register
    SP         stack pointer

There are separate link registers for function calls and exceptions

	X30 – Updated by branch with link instructions (BL & BLR)
	      Use RET instruction to return from sub-routines
	
	ELR_ELn – Updated on exception entry
	          Use ERET instruction to return from exceptions

Each exception level has its own stack pointer `SP_EL0`, `SP_EL1`, `SP_EL2` and `SP_EL3`

**Floating point/SIMD registers**

32 bit float registers

    S0-S7   arguments and return value
    S8-S15	Callee saved register
	S16-S15 Corruptible registers
	S24-S31 Corruptible registers

64 bit SIMD registers

	D0-D7   arguments and return value
	D8-D15	Callee saved register
	D16-D15 Corruptible registers
	D24-D31 Corruptible registers

128 bit SIMD registers

    V0-V7   arguments and return value
    V8-V15	Callee saved register
	V16-V15 Corruptible registers
	V24-V31 Corruptible registers

**The PC (program counter) is not a general purpose register, and cannot be directly accessed by most instructions.**

You can use ADR to get the the address of a PC relative offset

	adr x0 label	

**AArch64 Exceptions**

ARMv8-A exceptions interrupt the processor and change the control flow of the program. 

    SVC Supervisor Call attempts to access EL1 from EL0. SVC that generates a supervisor call, which are are normally used to request privileged operations or access to system resources from an operating system.
    HVC Hypervisor Call attempts to access EL2
    SMC Secure Monitor Call attempts to access EL3
    HLT Halting software breakpoint Instruction
    BRK software breakpoint instruction

**Exception handling in ARMv8**

In ARMv8, a new exception model has been introduced which defines the concept of exception levels.
When an exception occurs, the processor branches to an exception vector table and runs the corresponding handler. In ARMv8, each exception level has its own exception vector table.

	
| Offset from VBAR_EL1 | Exception type | Exception set level |
|---------------------------|-------------|--------------|
|+0x000 |Synchronous | Current EL with SP0 |
|+0x080|IRQ/vIRQ|"|
|+0x100|FIQ/vFIQ|"|
|+0x180|SError/vSError|"|
|+0x200|Synchronous|Current EL with SPx|
|+0x280|IRQ/vIRQ|"|
|+0x300|FIQ/vFIQ|"|
|+0x380|SError/vSError|"|
|+0x400|Synchronous|Lower EL using ARM64
|+0x480|IRQ/vIRQ|"|
|+0x500|FIQ/vFIQ|"|
|+0x580|SError/vSError|"|
|+0x600|Synchronous|Lower EL with ARM32|
|+0x680|IRQ/vIRQ|"|
|+0x700|FIQ/vFIQ|"|
|+0x780|SError/vSError|"|

Example on Linux Kernel el1_sync
```
el1_sync:
 kernel_entry 1
 mrs	x1, esr_el1			// read the syndrome register
 lsr	x24, x1, #ESR_ELx_EC_SHIFT	// exception class
 cmp	x24, #ESR_ELx_EC_DABT_CUR	// data abort in EL1
 b.eq	el1_da
 cmp	x24, #ESR_ELx_EC_IABT_CUR	// instruction abort in EL1
 b.eq	el1_ia
 cmp	x24, #ESR_ELx_EC_SYS64		// configurable trap
 b.eq	el1_undef
 cmp	x24, #ESR_ELx_EC_SP_ALIGN	// stack alignment exception
 b.eq	el1_sp_pc
 cmp	x24, #ESR_ELx_EC_PC_ALIGN	// pc alignment exception
 b.eq	el1_sp_pc
 cmp	x24, #ESR_ELx_EC_UNKNOWN	// unknown exception in EL1
 b.eq	el1_undef
 cmp	x24, #ESR_ELx_EC_BREAKPT_CUR	// debug exception in EL1
 b.ge	el1_dbg
 b	el1_inv
```
**AArch64 Instructions**

Data first loaded into registers, modified, and then stored back in memory or simply discarded once it’s no longer required.

	[instr] [cond] [dest]  [src] [operand]

Example

	mov w0, #5	;move 5 to w0 register
	add w2,w1,#3	;w2 = w1 + 3 

No load/store multiple instructions.

Conditional select

	CSEL x2, x4, x5 ;cond implements x2 = if cond then x4 else x5
	You can define CMOV x1, x2 with new condition  CSEL x1, x2, x1, cond	

No direct access to CPSR register but with System instruction MRS,MSR

AArch64 system configuration is controlled through system registers

	TTBR0_EL1 – can be accessed from EL1, EL2 and EL3
	TTBR0_EL2 – can be accessed from EL2 and EL3

Accessed using MSR and MRS instructions

MSR - Move System register to general-purpose register

MRS - Move general-purpose register to System register

Example

	MRS x0, TTBR0_EL1 ; Move TTBR0_EL1 into x0
	MSR TTBR0_EL1, x0 ; Move x0 into TTBR0_EL1

You can find more instructions on the following PDF

[ARMv8 A64 Quick Reference](https://courses.cs.washington.edu/courses/cse469/18wi/Materials/arm64.pdf)


**AArch64 MMU**

The AArch64 MMU is used to convert from virtual address to physical address and setting the memory attributes, such as access permissions that include read and write permissions for different privilege levels, memory type, and cache policies.

AArch64 Linux uses either 3 levels or 4 levels of translation table with the 4KB page configuration, allowing 39-bit (512GB) or 48-bit (256TB) virtual addresses, respectively, for both user and kernel. With 64KB pages, only 2 levels of translation tables, allowing 42-bit (4TB) virtual address, are used but the memory layout is the same. User addresses have bits 63:48 set to 0 while the kernel addresses have the same bits set to 1. Upper 8 bits of the address can be configured for Tagged Pointers.

Further separate TTBR register for user and kernel TTBRx selection is given by bit 63 of the virtual address.

**Translation Lookaside Buffer (TLB)**

A translation lookaside buffer (TLB) is a memory cache that is used to reduce the time taken to access a user memory location. It is a part of the chip's memory-management unit (MMU). The TLB stores the recent translations of virtual memory to physical memory and can be called an address-translation cache.
	
