---
layout: post
title: "Memory Tagging"
date:	Sun Apr 07 17:30:54 EEST 2019
---

**ARM Memory Tagging**

Memory tagging is a tag assigned to each memory allocation, so all accesses to memory must be via a pointer with correct tag, if the tag is wrong the Operating System can report it to user or note the process in which occured.

Furthermore memory tagging can be used for debugging in development to detect memory errors.

Allocations are done with align allocations by 16-bytes and choose a 4-bit tag which same tag is set to the corresponding pointer and memory. Deallocations are working with re-tag the memory with a different tag.

Example

```c
char *p = new char[20];
```

20 bytes needs to allocated, memory allocator aligns 32 bytes, 16 bytes will be used for tag granularity, the tag will be on the 4 most significant bits of pointer and memory, so when the attacker will try to dereferencing pointer p at position 32, tag values will not be correct between pointer and memory and overflow will be caught.

**Memory Tagging Extension instructions**

ARM implementated the memory tagging hardware support with tag size of 4-bit and tag granularity of 16-bytes and they have introduced some of the following new instructions

IRG - Insert Random Tag inserts a random logical address tag into the address in the first source register and writes the result to the destination register.

    IRG Xd|SP , Xn|SP; // irg  x0, sp;

CMPP - Compare with TAG subtracts the 56-bit address held in the second source register from the 56-bit address held in the first source register, updates the condition flags based on the result of the subtraction, and discards the result.

	CMPP <Xn|SP>, <Xm|SP> 

ADDG - Add with Tag adds an immediate value scaled by the Tag granule to the address in the source register, modifies the Logical Address Tag of the address using an immediate value, and writes the result to the destination register. Tags specified in GCR_EL1 system register.

	ADDG <Xd|SP>, <Xn|SP>, #<uimm6>, #<uimm4>; // addg  x19, x0, #16, #1

STG - Store Allocation Tag stores an Allocation Tag to memory. The address used for the store is calculated from the base register and an immediate signed offset scaled by the Tag granule. The Allocation Tag is calculated from the Logical Address Tag in the source register.

	STG [<Xn|SP>], #<simm>; // stg  x0, [x0]

LDG - Load Allocation Tag loads an Allocation Tag from a memory address, generates a Logical Address Tag from the Allocation Tag and merges it into the destination register. The address used for the load is calculated from the base register and an immediate signed offset scaled by the Tag granule.

	LDG <Xt>, [<Xn|SP>{, #<simm>}]


ST2G - Store Allocation Tags stores an Allocation Tag to two Tag granules of memory. The address used for the store is calculated from the base register and an immediate signed offset scaled by the Tag granule. The Allocation Tag is calculated from the Logical Address Tag in the source register.

	ST2G <Xt|SP>, [<Xn|SP>], #<simm>; // st2g  x8, [sp], #32


New System Registers for tagging

	TCO - Tag Check Override When ARMv8.5-MemTag is implemented, this register allows tag checks to be disabled globally.
	TFSRE0_EL1 - Tag Fail Status Register Holds accumulated Tag Check Fails occurring in EL0 which are not taken precisely.
	TFSR_EL1 - Tag Fail Status Register Holds accumulated Tag Check Fails occurring in EL1 which are not taken precisely.
	TFSR_EL2 - Tag Fail Status Register Holds accumulated Tag Check Fails occurring in EL2 which are not taken precisely.
	TFSR_EL3 - Tag Fail Status Register Holds accumulated Tag Check Fails occurring in EL3 which are not taken precisely.
	RGSR_EL1 - Random Allocation Tag Seed Register.
	GCR_EL1 - Tag Control Register.


**Compiler Memory Tagging**

**LLVM HWASAN**

Memory tagging can be found on CLANG toolchain and is called HWSAN hardware-assisted address sanitizer, and it works on AArch64 because it relies on the Top-Byte-Ignore(TBI) feature with 8-bit tag size and 16-bytes tag granularity. The tag validation is performed by compiler instrumentation during runtime but there is an overhead on cpu and memory.

*Top-byte ignore (TBI) is a feature introduced with ARMv8 that provides facilities for memory tagging by ignoring the most significant 8 bits of the virtual address.*

Memory Access Example

```
// clang -O2 --target=aarch64-linux -fsanitize=hwaddress -c main.c
	
<func1>:
   0:	d344dc08	ubfx	x8, x0, #4, #52 // shadow offset
   4:	39400108	ldrb	w8, [x8] // load shadow tag
   8:	d378fc09	lsr	x9, x0, #56 // extract address tag
   c:	6b08013f	cmp	w9, w8 // compare tags
  10:	54000061	b.ne	1c <foo+0x1c>  // jump on mismatch
  14:	b9400000	ldr	w0, [x0] // original load
  18:	d65f03c0	ret
  1c:	d4212040	brk	#0x902 // trap
```

The memory tagging extension is optional in ARMv8.5-A. Compiling with armclang toolchain with option -mmemtag-stack, the compiler uses memory tagging instructions that are not available for architectures without the memory tagging extension.

	-mmemtag-stack // enables the generation of stack protection code that uses the memory tagging extension.

The following optional extensions to Armv8.5-A architecture will be supported by GCC version 9 as well

	-march=armv8.5-a+rng // Random Number Generation instructions
    -march=armv8.5-a+memtag // Memory Tagging Extension

References

	https://llvm.org/devmtg/2018-10/slides/Serebryany-Stepanov-Tsyrklevich-Memory-Tagging-Slides-LLVM-2018.pdf
	https://developer.arm.com/docs/ddi0596/latest/base-instructions-alphabetic-order
	http://www.keil.com/support/man/docs/armclang_ref/armclang_ref_lnk1549304794624.htm
	https://gcc.gnu.org/gcc-9/changes.html
