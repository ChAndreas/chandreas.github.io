---
layout: post
title:  "ARM Pointer Authentication"
date:   Wed Sep 20 16:35:51 EEST 2018
---

The ARMv8.3 pointer authentication extension adds functionality to detect modification of pointer values, mitigating certain classes of attack such as
ROP/JOP attacks. In essence, it attaches a cryptographic signature to pointer values. Those signatures can be verified before a pointer is used. 
An attacker with lack of the key used to create the signatures, cannot create valid pointers for use in an exploit.

When CONFIG_ARM64_POINTER_AUTHENTICATION is selected, and relevant HW support is present, the kernel will assign a random APIAKey value to each process at exec*() time.
The key is assigned at exec() time, not at process creation. So threads share a key, and parent and child will share a key after fork() until one of them calls exec().

New instructions are added which can be used to:

    Insert a PAC into a pointer
    Strip a PAC from a pointer
    Authenticate strip a PAC from a pointer

PAC instruction sign pointers become not usable pointer (Pointer + PAC)
AUT instruction authenticate PAC if PAC match the result of the original pointer else the result will be an invalid pointer (fault).
XPAC instruction stip PAC remove authentication and restored to the original pointer

The new PAC instruction can be used to calculate the authentication code and stored within pointer value. The value containing the authentication code cannot be dereferenced directly, since, without the sign-extension bits, it is no longer recognized as a valid address.

Regaining a usable pointer requires using the AUT instruction, which will recalculate the authentication code and compare it to what is found in te authenticated pointer value. If the two match, the authentication code will be removed otherwise, the pointer will be modified to ensure a fault should it be dereferenced. Thus, any attempt to use a pointer that lacks a proper authentication code will lead to a crash. 

GCC 7 compiler include basic support for pointer authentication in the form of the -msign-return-address option.

Example without arm pointer authentication

    0000000000000724 <main>:
    724:	a9bf7bfd	stp	x29, x30, [sp, #-16]!
    728:	910003fd	mov	x29, sp
    72c:	90000000	adrp	x0, 0 <_init-0x598>
    730:	911fa000	add	x0, x0, #0x7e8
    734:	97ffffb7	bl	610 <printf@plt>
    738:	d503201f	nop
    73c:	a8c17bfd	ldp	x29, x30, [sp], #16
    740:	d65f03c0	ret
    744:	00000000	.inst	0x00000000 ; undefined

Example with arm pointer authentication

    0000000000000724 <main>:
    724:	d503233f	paciasp
    728:	a9bf7bfd	stp	x29, x30, [sp, #-16]!
    72c:	910003fd	mov	x29, sp
    730:	90000000	adrp	x0, 0 <_init-0x598>
    734:	911fc000	add	x0, x0, #0x7f0
    738:	97ffffb6	bl	610 <printf@plt>
    73c:	d503201f	nop
    740:	a8c17bfd	ldp	x29, x30, [sp], #16
    744:	d50323bf	autiasp
    748:	d65f03c0	ret
    74c:	00000000	.inst	0x00000000 ; undefined

