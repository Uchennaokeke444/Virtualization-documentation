---
title: Virtual Processor
description: Hypervisor Virtual Processor Interface
keywords: hyper-v
author: alexgrest
ms.author: hvdev
ms.date: 10/15/2020
ms.topic: reference
---

# Virtual Processors

Each partition may have zero or more virtual processors.

## Virtual Processor Indices

A virtual processor is identified by a tuple composed of its partition ID and its processor index. The processor index is assigned to the virtual processor when it is created, and it is unchanged through the lifetime of the virtual processor.

A special value HV_ANY_VP can be used in certain situations to specify “any virtual processor”. A value of HV_VP_INDEX_SELF can be used to specify one’s own VP index.

```c
typedef UINT32 HV_VP_INDEX;

#define HV_ANY_VP ((HV_VP_INDEX)-1)

#define HV_VP_INDEX_SELF ((HV_VP_INDEX)-2)
 ```

A virtual processor’s ID can be retrieved by the guest through a hypervisor-defined MSR (model-specific register) HV_X64_MSR_VP_INDEX.

```c
#define HV_X64_MSR_VP_INDEX 0x40000002
 ```

## Virtual Processor Idle Sleep State

Virtual processors may be placed in a virtual idle processor power state, or processor sleep state. This enhanced virtual idle state allows a virtual processor that is placed into a low power idle state to be woken with the arrival of an interrupt even when the interrupt is masked on the virtual processor. In other words, the virtual idle state allows the operating system in the guest partition to take advantage of processor power saving techniques in the OS that would otherwise be unavailable when running in a guest partition.

A partition which possesses the AccessGuestIdleMsr privilege may trigger entry into the virtual processor idle sleep state through a read to the hypervisor-defined MSR `HV_X64_MSR_GUEST_IDLE`. The virtual processor will be woken when an interrupt arrives, regardless of whether the interrupt is enabled on the virtual processor or not.

## Virtual Processor Assist Page

The hypervisor provides a page per virtual processor which is overlaid on the guest GPA space. This page can be used for bi-directional communication between a guest VP and the hypervisor. The guest OS has read/write access to this virtual VP assist page.

A guest specifies the location of the overlay page (in GPA space) by writing to the Virtual VP Assist MSR (0x40000073). The format of the Virtual VP Assist Page MSR is as follows:

| Bits      | Field           | Description                                                                 |
|-----------|-----------------|-----------------------------------------------------------------------------|
| 0         | Enable          | Enables the VP assist page                                                  |
| 11:1      | RsvdP           | Reserved                                                                    |
| 63:12     | Page PFN        | Virtual VP Assist Page PFN                                                  |

## Virtual Processor Run Time Register

The hypervisor’s scheduler internally tracks how much time each virtual processor consumes in executing code. The time tracked is a combination of the time the virtual processor consumes running guest code, and the time the associated logical processor spends running hypervisor code on behalf of that guest. This cumulative time is accessible through the 64-bit read-only HV_X64_MSR_VP_RUNTIME hypervisor MSR. The time quantity is measured in 100ns units.

## Non-Privileged Instruction Execution Prevention (NPIEP)

Non-Privileged Instruction Execution (NPIEP) is a feature that limits the use of certain instructions by user-mode code. Specifically, when enabled, this feature can block the execution of the SIDT, SGDT, SLDT, and STR instructions. Execution of these instructions results in a #GP fault.

This feature must be configured on a per-VP basis using [HV_X64_MSR_NPIEP_CONFIG_CONTENTS](datatypes/HV_X64_MSR_NPIEP_CONFIG_CONTENTS.md).

## Guest Spinlocks

A typical multiprocessor-capable operating system uses locks for enforcing atomicity of certain operations. When running such an operating system inside a virtual machine controlled by the hypervisor these critical sections protected by locks can be extended by intercepts generated by the critical section code. The critical section code may also be preempted by the hypervisor scheduler. Although the hypervisor attempts to prevent such preemptions, they can occur. Consequently, other lock contenders could end up spinning until the lock holder is re-scheduled again and therefore significantly extend the spinlock acquisition time.

The hypervisor indicates to the guest OS the number of times a spinlock acquisition should be attempted before indicating an excessive spin situation to the hypervisor. This count is returned in CPUID leaf 0x40000004. A value of 0 indicates that the guest OS should not notify the hypervisor about long spinlock acquisition.

The [HvCallNotifyLongSpinWait](hypercalls/HvCallNotifyLongSpinWait.md) hypercall provides an interface for enlightened guests to improve the statistical fairness property of a lock for multiprocessor virtual machines. The guest should make this notification on every multiple of the recommended count returned by CPUID leaf 0x40000004. Through this hypercall, a guest notifies the hypervisor of a long spinlock acquisition. This allows the hypervisor to make better scheduling decisions.