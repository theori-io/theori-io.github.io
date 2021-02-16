---
layout: post
title: "Compromising virtualization without attacking the hypervisor"
author: theori
description:
categories: [ research ]
tags: [ Hypervisor, exploit ]
comments: true
featured: false
image: assets/images/2020-10-21/XenVBDStateDiagram.svg
---

This post explains a denial-of-service (and potentially out-of-bounds write) vulnerability ([CVE-2020-27675]{:target="_blank"}, [XSA-331]{:target="_blank"}) we discovered in the Xen paravirtualization driver in Linux, and how a virtualization platform may be compromised without direct attacks to the hypervisor.

## Background

[Xen][Xen Project]{:target="_blank"} is a type-1 hypervisor used by automotive, defense, and enterprise customers. Conceived at the University of Cambridge Computer Laboratory, it has a record of leading the cloud and virtualization market for several years. Today, Xen is being developed by the Xen Project under the auspices of the Linux Foundation with support from Intel.

### Hypervisor and Driving Hardware

Contrary to type-2 hypervisors which rely on facilities provided by the operating system to control the hardware, type-1 hypervisors run directly on the hardware and require their own device drivers to operate.
Such drivers would intuitively be executed at the same level as the hypervisor itself.
One instance is VMware ESXi, which comes with its own driver stack like a conventional operating system and provides interfaces for third-party driver development.

This approach, however, is not without its own drawbacks; hardware support is limited as fewer drivers are available to the hypervisor.
A possible solution would be to provide a compatibility (shim) layer so that it could reuse drivers written for existing operating systems.
Hardware support in this case is less niche, at the additional maintenance cost of keeping the shim up to date with the original operating system.

Another route taken by some type-1 hypervisors, such as Xen, is to designate a guest as a driver container and grant it privileges to directly control the hardware.
The privileged guest does not necessarily share the same address space as the hypervisor, however, which mitigates some class of memory corruption vulnerabilities.
This trades off performance due to context switching between the driver guest and the hypervisor, which may be more costly in runtime than a shim layer built into the hypervisor.

### The Xen hypervisor

In Xen's terminology, a guest is called a "domain," and the highest privileged guest and an unprivileged guest are named "dom0" and "domU" respectively.
Although ACPI and fundamental platform devices are handled by dom0, other devices can be assigned to select domUs using IOMMU.
Combined with disaggegation of other system services to dedicated domains, this brings a Xen system closer to a microkernel architecture.
In a typical Xen system however, dom0 usually drives all hardwares by default, making it equivalent to the host machine of a type-2 hypervisor.

This naturally extends the trust model (and its attack surface) from the hypervisor itself to the privileged guest.
Even if the hypervisor itself remains uncompromised, any faults in the privileged guest would still bring down the entire system, or possibly open doors into the hypervisor and in turn the hardware.
A dom0 which is bloated and/or hosts complicated interfaces brings down the system's security to that of a typical type-2 hypervisor despite all the associated performance costs.

### Xen paravirtualization architecture

Full virtualization is designed to run unmodified guest operating systems; in contrast, paravirtualization adapts the guest to the hypervisor and the guest takes steps to optimize itself accordingly.
Unlike fully virtualized devices which emulate real hardware, paravirtualized "devices" require a specialized interface to operate.
Xen's paravirtualization interface is called [Paravirt-ops][pvops]{:target="_blank"}, which includes an IPC mechanism which works as a bus that allows different guests to interact with each other.
This IPC is backed by two primitives: grant tables and event channels.
[Grant tables][gnttab]{:target="_blank"} allow guests to share pages with other guests, which is akin to shared memory in conventional operating systems.
Handles used to identify such pages are called grant references, which are allocated by the sharing (initiating) guest.
[Event channels][evtchn]{:target="_blank"} are used to signal events from a guest to another guest (or to forward IPIs and physical IRQs), in the form of virtual interrupts.
Together, they are used to create a queue for inter-guest communication.

A [Xen paravirtualized device][pvdev]{:target="_blank"} consists of a frontend and a backend.
The frontend resides in the consumer guest. Its operating system usually expose it just like a normal hardware device for application use.
The backend resides in the provider guest (usually dom0), and is either attached to another device driver (in case of block devices) or simply acts as the other end of the interface (in case of network interfaces).
After dom0 binds both parties together with a paravirtualized device, they enter a state machine (discussed in [§ Exploitation](#Exploitation)) where they negotiate supported features (and run appropriate "hotplug" scripts) locally before finally establishing a connection.
Specifically, the frontend first allocates the queue resources and passes their handles to the backend, which uses the handles to communicate and implement a functional device.

Note that both parties, frontend and backend, still have to exchange resource handles to establish a queue.
Such out-of-band communications are performed via _another_ paravirtualized queue prepared by dom0.
This queue is used to access [XenStore][store]{:target="_blank"}, an ephemeral key-value store with discretionary access control.
XenStore mimics a file system with file change notification support, which guests use to transition each other's state machine, communicate necessary parameters, and notify another guest of creation of a queue (or removal thereof).
The XenStore daemon runs in dom0 and notifies the hypervisor of the queue resources, which in turn passes the queue handles to each paravirtualized domain.

## The Bug

The vulnerability resides in the Linux kernel's event channel implementation.
Below is a stack trace of a crash caused by this vulnerability.

```
BUG: unable to handle kernel NULL pointer dereference at 000000000000001c
PGD 0 P4D 0 
Oops: 0000 [#1] SMP PTI
CPU: 0 PID: 0 Comm: swapper/0 Tainted: G                  [REDACTED]
RIP: 0010:evtchn_from_irq+0x[REDACTED]
Code: [REDACTED]
...(snip)...
Call Trace:
 <IRQ>
 disable_dynirq+0x[REDACTED]
 mask_ack_dynirq+0x[REDACTED]
 handle_edge_irq+0x[REDACTED]
 generic_handle_irq+0x[REDACTED]
 __evtchn_fifo_handle_events+0x[REDACTED]
 __xen_evtchn_do_upcall+0x[REDACTED]
 xen_evtchn_do_upcall+0x[REDACTED]
 xen_hvm_callback_vector+0x[REDACTED]
 </IRQ>
...(snip)...
Kernel panic - not syncing: Fatal exception in interrupt
Kernel Offset: disabled
```

As discussed earlier, Xen events are delivered to the guest in the form of IRQs.
The `evtchn_from_irq` function in [`drivers/xen/events/events_base.c`][events_base.c] is as follows.

```c
/*
 * Accessors for packed IRQ information.
 */
unsigned int evtchn_from_irq(unsigned irq)
{
	if (WARN(irq >= nr_irqs, "Invalid irq %d!\n", irq))
		return 0;

	return info_for_irq(irq)->evtchn;
}
```

Analysis revealed that `info_for_irq(irq)` returned a NULL pointer when the crash above took place.
This function simply casts the `handler_data`, a field provided by the kernel for storing per-IRQ data pointers, into a pointer to a structure specific to the Xen driver.

```c
/* Get info for IRQ */
struct irq_info *info_for_irq(unsigned irq)
{
	return irq_get_handler_data(irq);
}
```

The cause of the NULL value can be traced back to `xen_free_irq`, which is used to release an event channel's associated IRQ after the channel is closed.
The function removes `handler_data` sans proper synchronization with the interrupt handler.

```c
static void xen_free_irq(unsigned irq)
{
	struct irq_info *info = irq_get_handler_data(irq);

	if (WARN_ON(!info))
		return;

	list_del(&info->list);

	irq_set_handler_data(irq, NULL);

	WARN_ON(info->refcnt > 0);

	kfree(info);

	/* Legacy IRQ descriptors are managed by the arch. */
	if (irq < nr_legacy_irqs())
		return;

	irq_free_desc(irq);
}
```

The function uses `irq_set_handler_data` to set `handler_data` to `NULL`, which causes subsequent invocations of `info_for_irq` for the IRQ to return `NULL` as well.
The `irq_set_handler_data` function is as follows.

```c
/**
 *	irq_set_handler_data - set irq handler data for an irq
 *	@irq:	Interrupt number
 *	@data:	Pointer to interrupt specific data
 *
 *	Set the hardware irq controller data for an irq
 */
int irq_set_handler_data(unsigned int irq, void *data)
{
	unsigned long flags;
	struct irq_desc *desc = irq_get_desc_lock(irq, &flags, 0);

	if (!desc)
		return -EINVAL;
	desc->irq_common_data.handler_data = data;
	irq_put_desc_unlock(desc, flags);
	return 0;
}
EXPORT_SYMBOL(irq_set_handler_data);
```

`irq_set_handler_data` holds the per-IRQ descriptor spinlock, which is shared with `handle_edge_irq`,
Since IRQs are preemptive and time critical, the per-IRQ descriptor spinlock is the only synchronization primitive that can be used to block the handler.
However, spinlocks in the Linux kernel are not recursive, which means that `xen_free_irq` by itself makes no attempt to stop the interrupt from firing at other times.
Any interrupt which begins its processing (holds the spinlock) between the short time window between setting `handler_data` to `NULL` and freeing the `IRQ` will observe the `NULL` value as a result.

A thing to note is that the Linux kernel implements "sparse IRQs" which stores IRQ descriptors in a radix tree instead of a linear array.
Sparse IRQs are unconditionally enabled in kernel configuration for most architectures (except for some embedded processors).
A consequence of sparse IRQs is that descriptors for new IRQs are dynamically allocated, instead of reusing an existing slot from a linear array.
`irq_free_desc` is responsible for freeing the IRQ descriptor, and by itself avoids racing with the interrupt by utilizing read-copy-update (RCU) to defer its actual operation until after any pending interrupt handlers finishes executing.

## Exploitation

As discussed earlier, paravirtualized devices make extensive use of event channels.
The frontend is responsible for providing the event channel handles; furthermore, the state machine allows us to renegotiate and reallocate resources with the backend.
Since the vulnerability resides in the closing of event channels, we shall make event channel interrupts race against closing the channel itself.
To achieve this, we repeatedly open and close the event channel while flooding events on it until the takes place.
For the former part, we shall first look into the state machine of paravirtualized devices.

```c
/* The state of either end of the Xenbus, i.e. the current communication
   status of initialisation across the bus.  States here imply nothing about
   the state of the connection between the driver and the kernel's device
   layers.  */
enum xenbus_state
{
	XenbusStateUnknown      = 0,
	XenbusStateInitialising = 1,
	XenbusStateInitWait     = 2,  /* Finished early
					 initialisation, but waiting
					 for information from the peer
					 or hotplug scripts. */
	XenbusStateInitialised  = 3,  /* Initialised and waiting for a
					 connection from the peer. */
	XenbusStateConnected    = 4,
	XenbusStateClosing      = 5,  /* The device is being closed
					 due to an error or an unplug
					 event. */
	XenbusStateClosed       = 6,

	/*
	* Reconfiguring: The device is being reconfigured.
	*/
	XenbusStateReconfiguring = 7,

	XenbusStateReconfigured  = 8
};
```

A total of 9 states are defined, and their numeric values are shared across all Xen devices; however, not all states are used by every type of device, and the usage also differs between frontend and backend.
For instance, `XenbusStateConnected` is used by both ends while `XenbusStateInitWait` is only used by the backend.
State transitions usually take place from a state of lower number to a higher one, although exact transitions differ between devices as well.

The most commonly used devices are virtual block devices (vbd) and virtual network interfaces (vif).
We specifically target the former, since attacks against the latter are crippled by the tear-down delay imposed (rather inadvertently) by the Linux networking subsystem.
The state machine for a Xen virtual block device backend is as follows.

{:.text-center}
![Xen VBD state machine diagram][vbddfa]

The exact conditions where a transition occurs are as follows:

 - **Transition to `XenbusStateInitWait`:** On dom0 request, the backend initializes the device.
   If successful, the backend transitions to this state.
 - **Transition to `XenbusStateConnected`:** When the frontend transitions to either `XenbusStateInitialised` or `XenbusStateConnected`, the backend attempts to connect to the queue.
   If such connection has already been established, the backend first disconnects to the frontend  nd reconnects.
   If successful, the backend transitions to this state.
 - **Transition to `XenbusStateClosing`:** When the frontend transitions to `XenbusStateClosing`, the backend simply transitions to this state as well. Nothing else is done.
 - **Transition to `XenbusStateClosed`:** When the frontend transitions to `XenbusStateClosed`, the backend disconnects from the queue.
   Whether successful or not, the backend transitions to this state.
   This transition may also happen preemptively if the device is removed for some other reason.

Note that it is permitted to transition from `XenbusStateClosing` or `XenbusStateClosed` back to `XenbusStateConnected`, which is mainly to support reloading drivers.
This is not a bug per se; nonetheless, this allows us to quickly connect and disconnect the device alternatively without assistance from dom0.
The proof of concept operates by rapidly switching between `XenbusStateConnected` and `XenbusStateClosing`, thereby triggering re-connection.

If the race condition takes place, the execution flow in the backend side may be as follows.

```
           vCPU 1                           vCPU 2
             |                                |
             |                           [interrupt]
             |                                |
+-------------------------+    +-----------------------------+
| unbind_from_irqhandler  |    |   xen_hvm_callback_vector   |
+-------------------------+    +-----------------------------+
             |                                :
+-------------------------+    +-----------------------------+
|        free_irq         |    |     handle_irq_for_port     |
+-------------------------+    +-----------------------------+
             :                                |
+-------------------------+                   |
|    __unbind_from_irq    |                   |
+-------------------------+    +-----------------------------+
             |                 |irq = get_evtchn_to_irq(port)|
+-------------------------+    |             ...             |
|    xen_evtchn_close     |    |   desc = irq_to_desc(irq)   |
+-------------------------+    |             ...             |
             |                 |    handle_edge_irq(desc)    |
+-------------------------+    +-----------------------------+
|  irq_set_handler_data   |                   |
+-------------------------+                   |
             |                                |
             |                 +-----------------------------+
             |                 | raw_spin_lock(&desc->lock)  |
             |                 +-----------------------------+
             |                                :
             |                 +-----------------------------+
             |                 |       evtchn_from_irq       | <-- NULL pointer dereference
             |                 +-----------------------------+
```

Another scenario, which is rather hypothetical, is when the released IRQ number is then taken by a subsequent unrelated IRQ allocation.
If it comes from a different hot-plugged physical device, it would use a totally different structure for `handler_data` rather than `irq_info`.
This may result in returning an arbitrary large value for the event channel handle.
As the handle is used as an index into an array which is shared with the hypervisor and stores various flags, a large value may lead to out-of-bounds access.

The `disable_dynirq` function in the call trace is as follows.

```c
static void disable_dynirq(struct irq_data *data)
{
	int evtchn = evtchn_from_irq(data->irq);

	if (VALID_EVTCHN(evtchn))
		mask_evtchn(evtchn);
}
```

If the IRQ specified by `data->irq` is taken by another device, then a completely different `handler_data` will be retrieved, possibly resulting in an arbitrary `evtchn` value.

There are two ABIs for event channels as of this writing: 2-level and FIFO.
`mask_evtchn` uses the appropriate implementation based on the current ABI in use.
FIFO is the newer interface provided by Xen 4.4 (released in 2014) or later.
Since both are affected in a similar manner, we specifically look into the FIFO case.

```c
static void evtchn_fifo_mask(unsigned port)
{
	event_word_t *word = event_word_from_port(port);
	sync_set_bit(EVTCHN_FIFO_BIT(MASKED, word), BM(word));
}
```

`evtchn_fifo_mask` in turn passes its argument (the event channel handle) to `event_word_from_port`, which uses it as an index into an array sans any bound checks.

```c
static inline event_word_t *event_word_from_port(unsigned port)
{
	unsigned i = port / EVENT_WORDS_PER_PAGE;

	return event_array[i] + port % EVENT_WORDS_PER_PAGE;
}
```

The definition for `event_array` is as follows.

```c
static event_word_t *event_array[MAX_EVENT_ARRAY_PAGES] __read_mostly;
```

Its elements are assigned kernel memory allocated via `__get_free_page`.

```c
static int evtchn_fifo_setup(struct irq_info *info)
{
...
	array_page = (void *)__get_free_page(GFP_KERNEL);
	if (array_page == NULL) {
		ret = -ENOMEM;
		goto error;
	}
	event_array[event_array_pages] = array_page;
...
}
```

If this case happens, the execution flow may be as follows.
Note that, unless using a real-time scheduler in the hypervisor, operations in a VCPU (even with interrupts "disabled") can be preempted and delayed by an arbitrary amount of time, whereas temporal nondeterminism for physical CPUs primarily comes from accesses to RAM due to cache miss.

```
           vCPU 1                           vCPU 2
             |                                |
             |                           [interrupt]
             |                                |
+-------------------------+    +-----------------------------+
| unbind_from_irqhandler  |    |   xen_hvm_callback_vector   |
+-------------------------+    +-----------------------------+
             |                                :
+-------------------------+    +-----------------------------+
|        free_irq         |    |     handle_irq_for_port     |
+-------------------------+    +-----------------------------+
             :                                |
+-------------------------+                   |
|    __unbind_from_irq    |                   |
+-------------------------+    +-----------------------------+
             |                 |irq = get_evtchn_to_irq(port)|
+-------------------------+    |             ...             |
|    xen_evtchn_close     |    |   desc = irq_to_desc(irq)   |
+-------------------------+    |             ...             |
             |                 |    handle_edge_irq(desc)    |
+-------------------------+    +-----------------------------+
|  irq_set_handler_data   |                   |
+-------------------------+                   |
             |                                |
+-------------------------+    +-----------------------------+
|      irq_free_desc      |    | raw_spin_lock(&desc->lock)  |
+-------------------------+    +-----------------------------+
             :                                :
             :                 +-----------------------------+
             :                 |                             |
+-------------------------+    |                             |
|  (new IRQ with same #)  |    | # preempted by hypervisor # |
+-------------------------+    |                             |
             |                 |                             |
             |                 +-----------------------------+
             |                                :
             |                 +-----------------------------+
             |                 |       evtchn_from_irq       |
             |                 +-----------------------------+
             |                                |
             |                 +-----------------------------+
             |                 |    event_word_from_port     | <-- out-of-bounds access
             |                 +-----------------------------+
```

## Conclusions

Although the vulnerability's effect is mostly limited to denial of service, system availability is a critical requirement for cloud-based service providers and automotive systems.
Disaggregation (i.e. relocating backends into a separate domain) is an effective way to mitigate this vulnerability, although many setups are known not to practice it and place backends in the most critical domain (enumerating affected service providers is out of scope of this post).
To some extent, this yields an interesting case where a virtualized infrastructure can be compromised without attacking the hypervisor itself.

[CVE-2020-27675]: https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2020-27675
[XSA-331]: https://xenbits.xen.org/xsa/advisory-331.html
[Xen Project]: https://xenproject.org/
[pvops]: https://wiki.xen.org/wiki/XenParavirtOps
[gnttab]: https://wiki.xen.org/wiki/Grant_Table
[evtchn]: https://wiki.xen.org/wiki/Event_Channel_Internals
[pvdev]: https://wiki.xen.org/wiki/Paravirtualization_(PV)#PV_IO_Drivers
[store]: https://wiki.xen.org/wiki/XenStore
[events_base.c]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/xen/events/events_base.c
[vbddfa]: /assets/images/2020-10-21/XenVBDStateDiagram.svg
