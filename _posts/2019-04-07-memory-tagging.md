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

MTE Example

```lldb
// Compiled with following options -march=armv8.5-a+rng -march=armv8.5-a+memtag -fsanitize=memtag 
// llvm-objdump -d test

0000000100007f04 _main:
100007f04: 7f 23 03 d5                 	hint #27
100007f08: ff 03 01 d1                 	sub	sp, sp, #64
100007f0c: fd 7b 03 a9                 	stp	x29, x30, [sp, #48]
100007f10: fd c3 00 91                 	add	x29, sp, #48
100007f14: 08 00 00 b0                 	adrp	x8, #4096
100007f18: 08 09 40 f9                 	ldr	x8, [x8, #16]
100007f1c: 08 01 40 f9                 	ldr	x8, [x8]
100007f20: a8 83 1f f8                 	stur	x8, [x29, #-8]
100007f24: 08 00 80 d2                 	mov	x8, #0
100007f28: e8 13 c8 9a                 	<unknown>
100007f2c: 08 01 81 91                 	<unknown>
100007f30: 08 09 20 d9                 	<unknown>
100007f34: 09 00 80 52                 	mov	w9, #0
100007f38: 09 01 00 b9                 	str	w9, [x8]
100007f3c: 08 00 00 90                 	adrp	x8, #0
100007f40: 08 d1 3e 91                 	add	x8, x8, #4020
100007f44: ea 03 00 91                 	mov	x10, sp
100007f48: 48 01 00 f9                 	str	x8, [x10]
100007f4c: 00 00 00 90                 	adrp	x0, #0
100007f50: 00 c0 3e 91                 	add	x0, x0, #4016
100007f54: 13 00 00 94                 	bl	#76 <_printf+0x100007fa0>
```

If we will check the unknown instructions with llvm machine code we will see the new instructions for Memory tagging

```bash
echo "0xE8 0x13 0xC8 0x9A" | llvm-mc --disassemble -triple=aarch64 --mattr=+mte -show-encoding
	.text
	irg	x8, sp, x8              // encoding: [0xe8,0x13,0xc8,0x9a]

```

```bash
echo "0x08 0x01 0x81 0x91" | llvm-mc --disassemble -triple=aarch64 --mattr=+mte -show-encoding
	.text
        addg	x8, x8, #16, #0           // encoding: [0x08,0x01,0x81,0x91]
```

```bash
echo "0x08 0x09 0x20 0xd9" | llvm-mc --disassemble -triple=aarch64 --mattr=+mte -show-encoding
	.text
	stg	x8, [x8]                // encoding: [0x08,0x09,0x20,0xd9]
```


References

	https://llvm.org/devmtg/2018-10/slides/Serebryany-Stepanov-Tsyrklevich-Memory-Tagging-Slides-LLVM-2018.pdf
	https://developer.arm.com/docs/ddi0596/latest/base-instructions-alphabetic-order
	http://www.keil.com/support/man/docs/armclang_ref/armclang_ref_lnk1549304794624.htm
	https://gcc.gnu.org/gcc-9/changes.html
	https://github.com/llvm-mirror/llvm/blob/1338c14d0676434da021911c8824f00425d4895d/test/MC/AArch64/armv8.5a-mte.s
