# DPDK Overview

The main goal of the DPDK is to provide a simple, complete framework for fast packet processing in data plane applications. Users may use the code to understand some of the techniques employed, to build upon for prototyping or to add their own protocol stacks. Alternative ecosystem options that use the DPDK are available.

The framework creates a set of libraries for specific environments through the creation of an Environment Abstraction Layer (EAL), which may be specific to a mode of the Intel® architecture (32-bit or 64-bit), Linux* user space compilers or a specific platform. These environments are created through the use of make files and configuration files. Once the EAL library is created, the user may link with the library to create their own applications. Other libraries, outside of EAL, including the Hash, Longest Prefix Match (LPM) and rings libraries are also provided. Sample applications are provided to help show the user how to use various features of the DPDK.

The DPDK implements a run to completion model for packet processing, where all resources must be allocated prior to calling Data Plane applications, running as execution units on logical processing cores. The model does not support a scheduler and all devices are accessed by polling. The primary reason for not using interrupts is the performance overhead imposed by interrupt processing.

In addition to the run-to-completion model, a pipeline model may also be used by passing packets or messages between cores via the rings. This allows work to be performed in stages and may allow more efficient use of code on cores.

**components_architecture**
![Alt text](/pic/components_architecture.png)

# Environment Abstraction Layer

The Environment Abstraction Layer (EAL) is responsible for gaining access to low-level resources such as hardware and memory space. It provides a generic interface that hides the environment specifics from the applications and libraries. It is the responsibility of the initialization routine to decide how to allocate these resources (that is, memory space, PCI devices, timers, consoles, and so on).

Typical services expected from the EAL are:

    DPDK Loading and Launching: The DPDK and its application are linked as a single application and must be loaded by some means.
    Core Affinity/Assignment Procedures: The EAL provides mechanisms for assigning execution units to specific cores as well as creating execution instances.
    System Memory Reservation: The EAL facilitates the reservation of different memory zones, for example, physical memory areas for device interactions.
    PCI Address Abstraction: The EAL provides an interface to access PCI address space.
    Trace and Debug Functions: Logs, dump_stack, panic and so on.
    Utility Functions: Spinlocks and atomic counters that are not provided in libc.
    CPU Feature Identification: Determine at runtime if a particular feature, for example, Intel® AVX is supported. Determine if the current CPU supports the feature set that the binary was compiled for.
    Interrupt Handling: Interfaces to register/unregister callbacks to specific interrupt sources.
    Alarm Functions: Interfaces to set/remove callbacks to be run at a specific time.
***
**UIO overview**
![Alt text](/pic/UIO_overview.png)
***

**EAL Initialization in a Linux Application Environment**

Part of the initialization is done by the start function of glibc. A check is also performed at initialization time to ensure that the micro architecture type chosen in the config file is supported by the CPU. Then, the main() function is called. The core initialization and launch is done in rte_eal_init() (see the API documentation). It consist of calls to the pthread library (more specifically, pthread_self(), pthread_create(), and pthread_setaffinity_np()).

![Alt text](/pic/EAL_Init.png)
***
**NOTE:**
Initialization of objects, such as memory zones, rings, memory pools, lpm tables and hash tables, should be done as part of the overall application initialization on the master lcore. The creation and initialization functions for these objects are not multi-thread safe. However, once initialized, the objects themselves can safely be used in multiple threads simultaneously.

**More Detailed information please refer to the link:
http://dpdk.org/doc/guides/prog_guide/env_abstraction_layer.html**

**I believe every guy would like to study DPDK must carefully browse it**

***

***
**Note**

In DPDK PMD, the only interrupts handled by the dedicated host thread are those for link status change (link up and link down notification) and for sudden device removal.
***
### RX Interrupt Event

The receive and transmit routines provided by each PMD don’t limit themselves to execute in polling thread mode. To ease the idle polling with tiny throughput, it’s useful to pause the polling and wait until the wake-up event happens. The RX interrupt is the first choice to be such kind of wake-up event, but probably won’t be the only one.

EAL provides the event APIs for this event-driven thread mode. Taking linuxapp as an example, the implementation relies on epoll. Each thread can monitor an epoll instance in which all the wake-up events’ file descriptors are added. The event file descriptors are created and mapped to the interrupt vectors according to the UIO/VFIO spec. From bsdapp’s perspective, kqueue is the alternative way, but not implemented yet.

EAL initializes the mapping between event file descriptors and interrupt vectors, while each device initializes the mapping between interrupt vectors and queues. In this way, EAL actually is unaware of the interrupt cause on the specific vector. The eth_dev driver takes responsibility to program the latter mapping.

***
**Note**

Per queue RX interrupt event is only allowed in VFIO which supports multiple MSI-X vector. In UIO, the RX interrupt together with other interrupt causes shares the same vector. In this case, when RX interrupt and LSC(link status change) interrupt are both enabled(intr_conf.lsc == 1 && intr_conf.rxq == 1), only the former is capable.
***
The RX interrupt are controlled/enabled/disabled by ethdev APIs - ‘rte_eth_dev_rx_intr_*’. They return failure if the PMD hasn’t support them yet. The intr_conf.rxq flag is used to turn on the capability of RX interrupt per device.
