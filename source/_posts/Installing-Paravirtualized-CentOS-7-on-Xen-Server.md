title: Installing Paravirtualized (PVH) CentOS 7 on XenServer Part 1
author: Mohamad mehdi Kharatizadeh
tags:
  - Xen
  - XenServer
  - Citrix
  - Virtualization
  - Paravirtualization
  - Hypervisor
categories: []
date: 2017-12-25 22:04:00
---
In these series of articles, I'll discuss and demonstrate how to install a paravirtualized CentOS 7 guest VM in Citrix XenServer. First we'll have a discussion about Xen, different virtualization types and specifcally PV mode. Then we move on to installation procedures.

In this article, I'll talk about Xen and virtualization types a bit and we get to installation procedures in subsequent articles. **PVH** or **Paravirtualization with Hardware Assisted Virtualization** is relatively new in Xen, but unfortunately although supported by Xen, XenServer provides no reasonable way of defining and creating PVH's. Also installation procedure is quite tricky when compared to a PV-HVM virtual machine creation. We'll have to get our hands dirty on kernel, initramfs and also partitioning. Lot's of information is scattered on the web about PVH mostly through Xen project's wiki and this is exactly why I wanted to write a clean article which provides theoretical stuff behine virtualization types and also a relatively installation guide for CentOS 7.

The problem of PVH currently is huge memory access overhead on AMD64 systems. Technically without extra protected mode rings (we'll discuss this shortly) isolation of DomU address spaces has to be done through a technique known as *shadow paging* which affects memory performance in guest VM. Even though this limitation imposes a limit on real world scenarios where PVH still can not be used to reduce a virtual machine overhead (well I know we are using containers, right? but many things can not be deployed using containers, like NTP, iSCSI and Samba) it is still fun to play with, and I'm going to show you how.. Well, We'll get there shortly...

## Xen

[Xen](http://www.xenproject.org/) is a powerful and opensource hypervisor developed since 2003. By hypervisor we mean it's software which directly controls hardware directly and is able to run virtual machines. Much like VMWare ESXi or Hyper-V. Back in it's time, Xen introduced several techniques which were unique to itself and this delivered to it's popularity. In terms of performance, capabilities and stability it's still superior to it's opensource rival KVM (Kernel Virtual Machine) which is independently integrated into Linux kernel and is backed by RedHat. Citrix is the company which has acquired Xen and develops commercial hypervisor product XenServer which is built upon core Xen project.

One of key differences compared to hosted virtualization (like VirtualBox or VMWare Workstation) when using hypervisors is the notion of domains: Each virtual machine runs in a separate context which we call a **domain**, and the key difference is the primary booted operating system which lets administrators maintain hypervisor, configure VMs and etc. which we call **Dom0**. At first glance one might not understand difference of Dom0 and normal operating systems hosting virtual machines, but under the hood, they are totally different from each other.

In a hosted virtual machine environment, host operating system boots first and then runs virtual machines in user address space. Whereas in Hypervisors, like Xen, Xen boots first, sets up syscall APIs and Dom0 operating system boots afterwards. There are two main differences:

1. The order of boot, Hypervisor boots first and runs Dom0 afterwards.
2. Dom0 and subsequent DomU's access hardware indirectly using APIs provided by Hypervisor.

![xen architecture](https://wiki.xen.org/images/thumb/6/63/Xen_Arch_Diagram.png/500px-Xen_Arch_Diagram.png)

*Image taken from Xen project [wiki](https://wiki.xen.org/wiki/Xen_Project_Software_Overview) page*

Fact #2 is the reason why a normal operating system kernel can not be run as a Dom0: It has to support and interface hardware using system calls provided by Hypervisor. This is why Xen project has it's own patched Linux Kernels, and when compared to KVM, KVM does not need any patched kernel because it's support is already integrated into kernel code itself.

Keep this *patched* kernel thing in mind, because we are going to reference this a bit further when we get to paravirtualization.

## What is Virtualized?

Ok let's slow it down a little bit.. Let's talk about what's to virtualize: what is what that makes a virtual machine? The answer to this question allows better understanding of virtualization types:

1. PC Architecture; And by this I mean BIOS or EFI Firmware, ACPI, PCI, MMU and so forth. Let's not talk about full architecture virtualization since QEMU can only do that, for example, running ARM instructions over X86.
2. Hardware; Devices like VGA, Network Card and Disk controller.
3. Privileged Instructions; CPU instructions that has to be virtualized somehow to let guest operating system coexist with host operating system. These instructions are like APIC (Advanced Programmable Interrupt Controller), Page Table Extensions and so forth.

## How is Virtualized?

Now that we know what's to virtualize, let's talk about how these components can be virtualized under the hood of a hypervisor.

### PC Architecture Virtualization

Xen and also KVM get help from good ol' [QEMU](https://www.qemu.org/) for this purpose. For those who have experienced KVM using libvirt have experienced this firsthand. I'm not going to talk about QEMU, But All I've got to say about it is that it is a monster, which you can use to bring up an ARM or Android device (I'm not talking about X86 ARM images here) or even develop and debug BIOS and EFI Firmwares (If you happen to work for major major motherboard manufacturers by chance). QEMU has been around for quite some time, and apart that it's not multi-threaded (this feature is under development) It's a beast! This is why both Xen and KVM get help from QEMU when it comes to PC Architecture virtualization. QEMU provides BIOS (SMBios), PCI, etc. and takes care of the first of our list.. If you have time, I suggest taking up a look at QEMU project.

### Hardware Devices

Well, we could take one of three strategies when it comes to virtualizing our hardware devices. Let's list them down:

1. **Passthrough**: Redirect data to physical hardware itself. This can be done quite easily for USB devices, things get nasty when we come across IO devices such as Network Card or GPUs, the term we call **PCI Passthrough**. The reason of this difficulty is that IO to these devices is controlled by IOMMU controller, which host operating system has access to, and can not share it with anyone (due to obvious reasons). PCI Passthrough can not be performed unless CPU provides a way of splitting IOMMU between host operating system and guest operating systems, extension known as VT-d or Virtualization Technology for Directed I/O in Intel processors (a feature existing only on Xeon processors until recently integrated into Core architecture as well). VT-d proivdes IOMMU groups, which a hypervisor can use to perform PCI passthrough and prevent Dom0 and guest operating systems to intercept each other when accessing IOMMU.
2. **Software Emulation**: This solution is quite obvios: We need to create emulated hardware in software. This is the case when we come across devices such as VGA and SoundCard when use virtualization except we are using PCI passthrough for this purpose.
3. **Paravirtualization**: Well, this term is quite tricky.. But the answer to low performance and huge overhead of a fully emulated hardware is simple: Do we really need to emulate everyhing hardware in software? The answer is No, and we call that Paravirtualization. *Paravirtualization* works in device driver level in guest operating system: Instead of fully emulating the hardware, Device Driver simply calls system calls provided by Hypervisor. This provides smallest overhead and near to native performance when it comes to IO devices such as disk drives and network cards. Nowadays, All hypervisor solutions virtualize disk drives and network cards using paravirtualized drivers. This is why KVM is the perfect virtualization solution for desktops... It requires no change to kernel and offers the best performance using VirtIO drivers (RedHat develops VirtIO drivers for Windows guests too).

### Privileged Instructions

In order to abstract and isolate privileged instructions (like page tables, etc.) a nested context switch has to occur. Why? Because Virtual Machines context switch with each other, And in turn, Processes running in guest operating systems have to context switch too.

Generally three mechanisms have been practiced to achieve privilege instruction virtualization:

1. **Binary Address Translation**: In this strategy, hypervisor replaces machine executable code prior to execution whenever necessary, And incorporates other software techniques for things such as page tables. This mode is still supported by VMWare Workstation. This is slow, apart from the fact that finding and replacing code is slow, It requires flushing CPU instruction cache over and over again which degrades overall system performance. The pros is that is requires no extra protected mode rings (see below) or CPU extension support.
2. **Extra Protected Mode Rings**: I'm not going to talk about X86 protected modes, Since we will have to get deep into PC architecture history... The days of segmented memory model and revolution of virtual memory and protected mode. Suffice to say that historically X86 architecture introduced protected mode to isolate kernel space from user space applications, which calls them *rings*, and reserved extra rings for device drivers sitting between kernel and user space apps, which was never used in reality and was taken over like a hack by hipervisors to perform nested context switches. These extra rings were removed in AMD64 architecture, so this mechanism is not in practice anymore.
3. **Hardware Assisted Virtualization**: The mechanism widely in practice today, is using extensions (extended instructions to AMD64) provided by CPU to perform these context switches, handle page tables and so forth. As I've said already, extra protected mode rings were removed in AMD64 architecture so virtualization can not be done without using CPU extensions anymore (unless we use binary address translation which is very slow).

Prior to emergence of AMD64 architecture, This nested context switch was performed quite easily. Since X86 architecture supported more than two protected rings (generally a user space and kernel space was only in use) and third one was used by hypervisor for this purpose. Originally the idea was that device drivers could sit in a space between kernel space and user space which never saw it's reality, instead it was incorporated like a hack in hypervisors to perform nested context switches. Unfortunately, AMD did not see this as helpful and thought that if they removed extra protected rings, they would remove complexity from their architecture (or maybe they just wanted to clean up architectural mess). 

After AMD64 was widespread, extensions to AMD64 was introduced by both Intel and AMD to enable virtualization, now having removed extra protected rings. These extensions are called VT/x in Intel CPUs and enables Hypervisors to perform nested context switches for virtualization.

## Virtualization Types

Ok.. we are getting there.. Hold on!

Difference between virtualization types is the way above components can be virtualized by different strategies. I have taken terminologies defined in [Xen project's wiki page](https://wiki.xen.org/wiki/Xen_Project_Software_Overview).

1. **Full Virtualization (FV)**: Simply emulate everything in software apart from privileged instructions. This mechanism was in use prior to emergence of AMD64 architecture which removed support of extra protected rings.
2. **Full Paravirtualization (PV)**: Use modified operating system kernel to access architecture hardware and devices through system calls provided by Hypervisor. In this mode, not even hardware architecture needs to be emulated. The strategy is simple: Find every place in Kernel code that accesses hardware in some way and replace it with system calls provided by Hypervisor. Interesting enough, a Linux kernel can be patched to boot as paravirtualized guest with a hypervisor written in only a 1000 lines of code! This mode of virtualization was introduced by Xen project initially and has contributed to Xen popularity in some point. In terms of lightness you can consider PV something between containers nowadays and virtual machines. PVs require more resources than containers but less resources than VMs and boot quite instantly! Unfortunately PV relied on extra protected mode rings which we talked about them earlier, and these extra rings were removed in AMD64 architecture, so we really can't use full paravirtualization anymore in AMD64 architecture.
3. **Hardware Assisted Virtualization (HVM)**: Emulate architecture and devices in software like Full Virtualization but take help from virtualization extensions to virtualize privileged instructions. This is generally the method in use since emergance of AMD64 in products such as Oracle VirtualBox and VMWare Workstation.
4. **Hardware Assisted Virtualization with Paravirtualized(PV) Drivers (PV-HVM)**: Emulate architecture and some devices in software like Full Virtualization and Hardware Assisted Virtualization but use Paravirtualized Drivers for IO devices like Disk and Network Cards. This is the case for Xen, KVM, VMWare ESXi and so forth.
5. **Paravirtualization with Hardware Assisted Virtualization (PVH)**: This method revives paravirtualization by incorporating hardware assisted virtualization for virtualization of privileged instructions. This is exactly the mode we are going to get our hands dirty on. When compared to PV-HVM, PVH is quite light weight, fast to boot and has less overhead. It also can be used to boot a more lightweight Dom0 machine. 

## To be Continued:

In subsequent article, we'll get to all the fun: I'll show you step by step instructions of installing a CentOS7 PVH-enabled guest under Citrix XenSever. So please stay tuned ^_^