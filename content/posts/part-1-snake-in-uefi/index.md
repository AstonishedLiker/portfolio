+++
title = "(Part 1) Snake in... UEFI?"
description = "Seems crazy, right? Writing a game that runs before any operating system exists sounds complicated, obscure, and wildly impractical. So why would anyone bother?"
date = "2026-02-08T23:53:21.000+01:00"
cover = ""
tags = ["edk2", "uefi", "firmware"]
+++

_Seems crazy, right? Writing a game that runs **before any operating system exists** sounds complicated, obscure, and wildly impractical.  
So why would anyone bother?_

It turns out there are actually good reasons to do exactly that, and it's far less magical (and far more interesting) than it sounds.

In this article, I'll go over the basics of UEFI and the [EDK2](https://github.com/tianocore/edk2) toolchain (a must-have for UEFI development), and document the very real pain of setting up [LLDB](https://lldb.llvm.org/) to debug [QEMU](https://www.qemu.org/) at the firmware level.

## Why?!

To the uninitiated, programming in at the firmware level is complex, especially in the context of UEFI: no [floating point arithmetic](https://en.wikipedia.org/wiki/Floating-point_arithmetic), silent [undefined behavior](https://en.wikipedia.org/wiki/Undefined_behavior).

### A bit of background information

According to [Verified Market Reports](https://www.verifiedmarketreports.com/product/bios-basic-input-output-system-market/),
> _"UEFI (Unified Extensible Firmware Interface) held the largest share at 65%, while Metal EFI accounted for 35%."_
> 
> "Metal EFI" here means Legacy BIOS

As it turns out, UEFI is the standard for virtually all modern PCs, laptops, and as of recently Android devices, [which starting from Android 16, should include Google's Generic Bootloader (GBL), which implements UEFI protocols](https://source.android.com/docs/core/architecture/bootloader/generic-bootloader).

Essentially, the wide-spread adoption of UEFI results from standarization, _arguably_ improved security, and the need to overcome BIOS limitations in modern computing environments, driving more features and improvements, like faster boot times, cross-platform compatibility, and easier OS development.

### The outcome

Knowing your way around UEFI is a must-have to understand how billions of devices work at the firmware nowadays.

The idea to make a simple game like Snake _now makes sense_: learning to develop in a hostile environment, starting with the basics!

## What's UEFI, technically speaking?

Before we dive into anything programming-related, we must first understand the standard we're working with.

### An UEFI-compliant device's boot flow

> ![Diagram of the PI (platform initialization) boot phases](pi-boot-flow-diagram.png)
> _Source: https://igor-blue.github.io/2021/02/04/secure-boot.html_

Here's essentially what happens when you power a UEFI-compliant device on:

- After the circuitry is initialized, and, if on an Intel system, Intel ME's BootGuard verified the **IBB** (also known as **Initial Boot Block**, which contains the firmware code responsible for initializing the system), the CPU will eventually jump to the **SEC (Security) phase**.
    - This phase verifies the **PEI (Pre-EFI Execution Environment)** before handing it over control.
- The **PEI (Pre-EFI Execution Environment)** will bring up the DRAM (as, up to that point, the firmware used the CPU's **C**ache **A**s **R**am, also known as **CAR**), and then will discover the available hardware on the platform to perform some early initialization on said hardware.
    - Findings in this phase are transferred via **HOBs (Hand-Off Blocks)** that are transmitted to the next phase, the **DXE (Driver Execution Environment) Phase**.
- Afterwards, the **DXE (Driver Execution Environment) phase**, implemented as [`DxeMain` in EDK2](https://github.com/tianocore/edk2/blob/master/MdeModulePkg/Core/Dxe/DxeMain/DxeMain.c), will invoke the DXE dispatcher, the latter sweeping all the DXE drivers until they all execute **exactly once** (well, almost, continue reading!).
    - Those sweeps happen because DXE drivers may depend on protocols that haven't been initialized yet, requiring the DXE drivers that implement said protocols to run first.
    - **A DXE driver may never load if its dependencies are never satisfied!**
- Last but not least, the next phase is called the **BDS (Boot Device Selection) phase**, implemented as [`BdsDxe` in EDK2](https://github.com/tianocore/edk2/blob/f0542ae07d5e5e4d311bf8ae33bf26b4b1acf9f4/MdeModulePkg/Universal/BdsDxe/BdsEntry.c), is responsible for booting a storage medium, or into another DXE driver (like so-called "BIOS Setup Menus"), after DXE phase is over.

## That's great, but what about Snake?

Right, Snake. All UEFI phases up to BDS are, [under normal circumstances](https://www.binarly.io/logofail), controlled by the **IBV** (**Independent Bios Vendor**, eg. [AMI](https://www.ami.com/), [Insyde](https://www.insyde.com/)) and the **OEM** (**Original Equipement Manufacturer**, eg. [Acer](http://acer.com/), [Gigabyte](https://gigabyte.com/)).

That lets us with only one possible way of running Snake in UEFI: making a **UEFI application**.

There are almost no differences from a **UEFI application** (known as `UEFI_APPLICATION` in EDK2) and a **DXE driver** (known as `UEFI_DRIVER` in EDK2), aside from the fact that the latter loads before BDS, may utilize Depex (and therefore have dependencies), and is stored on the SPI chip (the chip where the firmware's code is at). They essentially have the same capabilities.

## What's Next?

Now that you understand *why* UEFI development matters and *how* the boot process works, we're ready to get practical!

In **Part 2**, we'll:
- Install and configure the EDK2 toolchain
- Understand the structure of UEFI projects (`.inf`, `.dsc`, and `.dec` files)
- Write and compile a minimal "Hello World" UEFI application
- Run it in QEMU and see our code execute at the firmware level

And if you think compiling is the hard part... wait until we try debugging it!

**Stay tuned!**
