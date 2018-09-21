---
layout: post
title:  "Control Flow Integrity"
date:   Wed Sep 20 16:15:51 EEST 2018
---

To improve security modern systems contain many mitigation strategies that try to make it harder to exploit security vulnerabilities.
One of those mitigation technologies that is recently added in Android 9 is Control Flow Integrity.

CFI is a security mechanism that disallows changes to the original control flow graph of a compiled binary(Control-flow hijacking: Arbitrary code execution), making it significantly harder to perform such attacks.

CFI restricts the control-flow of an application to valid execution traces by monitoring the program runtime and compare it with precomputed state and if an invalid state an alert is raised and terminate the app.

The CFI mechanism consists of two abstract components: the (often static) analysis component that recovers the Control-Flow Graph (CFG) (Paths defined by application's Control-Flow Graph) of the application (at different levels of precision) and the dynamic enforcement mechanism that restricts control flows according to the generated CFG.

CFI assumes that executable code is read-only, otherwise an attacker could simply overwrite code and remove the runtime monitors.

Indirect control-flow transfers are further divided into forward-edge control-flow transfers and backward-edge control-flow transfers.

Forward-edge control-flow transfers direct code forward to a new location and are used in indirect jump and indirect call instructions, which are mapped at the source code level to, e.g., switch statements, indirect calls, or virtual calls.

The backward-edge is used to return to a location that was used in a forward-edge earlier, e.g., when returning from a function call through a return instruction. 

Example CFI on Android Makefile (Implementing system CFI)

Android.mk

    LOCAL_SANITIZE := cfi
    # Optional features
    LOCAL_SANITIZE_DIAG := cfi
    LOCAL_SANITIZE_BLACKLIST := cfi_blacklist.txt

Example CFI in blueprint files

Android.bp

    sanitize: {
        cfi: true,
        diag: {
            cfi: true,
        },
        blacklist: "cfi_blacklist.txt",
    },
