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

* DPDK Loading and Launching: The DPDK and its application are linked as a single application and must be loaded by some means.
* Core Affinity/Assignment Procedures: The EAL provides mechanisms for assigning execution units to specific cores as well as creating execution instances.
* System Memory Reservation: The EAL facilitates the reservation of different memory zones, for example, physical memory areas for device interactions.
* PCI Address Abstraction: The EAL provides an interface to access PCI address space.
* Trace and Debug Functions: Logs, dump_stack, panic and so on.
* Utility Functions: Spinlocks and atomic counters that are not provided in libc.
* CPU Feature Identification: Determine at runtime if a particular feature, for example, Intel® AVX is supported. Determine if the current CPU supports the feature set that the binary was compiled for.
* Interrupt Handling: Interfaces to register/unregister callbacks to specific interrupt sources.
* Alarm Functions: Interfaces to set/remove callbacks to be run at a specific time.

## EAL in a Linux-userland Execution Environment

In a Linux user space environment, the DPDK application runs as a user-space application using the pthread library. PCI information about devices and address space is discovered through the /sys kernel interface and through kernel modules such as **uio_pci_generic**, or **igb_uio**. Refer to the UIO: User-space drivers documentation in the Linux kernel. This memory is mmap’d in the application.

The EAL performs physical memory allocation using mmap() in hugetlbfs (using huge page sizes to increase performance). This memory is exposed to DPDK service layers such as the Mempool Library.

At this point, the DPDK services layer will be initialized, then through pthread setaffinity calls, each execution unit will be assigned to a specific logical core to run as a user-level thread.

The time reference is provided by the CPU Time-Stamp Counter (TSC) or by the HPET kernel API through a mmap() call.

**UIO overview**
![Alt text](/pic/UIO_overview.png)

**EAL Initialization in a Linux Application Environment**

Part of the initialization is done by the start function of glibc. A check is also performed at initialization time to ensure that the micro architecture type chosen in the config file is supported by the CPU. Then, the main() function is called. The core initialization and launch is done in **rte_eal_init()** (see the API documentation). It consist of calls to the pthread library **(more specifically, pthread_self(), pthread_create(), and pthread_setaffinity_np())**.

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
### PCI Access

The EAL uses the **/sys/bus/pci** utilities provided by the kernel to scan the content on the PCI bus. To access PCI memory, a kernel module called uio_pci_generic provides a **/dev/uioX** device file and resource files in **/sys** that can be mmap’d to obtain access to PCI address space from the application. The DPDK-specific **igb_uio** module can also be used for this. Both drivers use the uio kernel feature (userland driver).

### User Space Interrupt Event

#### User Space Interrupt and Alarm Handling in Host Thread

The EAL creates a host thread to poll the UIO device file descriptors to detect the interrupts. Callbacks can be registered or unregistered by the EAL functions for a specific interrupt event and are called in the host thread asynchronously. The EAL also allows timed callbacks to be used in the same way as for NIC interrupts.

*****
**Note**

In DPDK PMD, the only interrupts handled by the dedicated host thread are those for link status change (link up and link down notification) and for sudden device removal.
*****

#### RX Interrupt Event

The receive and transmit routines provided by each PMD don’t limit themselves to execute in polling thread mode. To ease the idle polling with tiny throughput, it’s useful to pause the polling and wait until the wake-up event happens. The RX interrupt is the first choice to be such kind of wake-up event, but probably won’t be the only one.

EAL provides the event APIs for this event-driven thread mode. Taking linuxapp as an example, the implementation relies on epoll. Each thread can monitor an epoll instance in which all the wake-up events’ file descriptors are added. The event file descriptors are created and mapped to the interrupt vectors according to the UIO/VFIO spec. From bsdapp’s perspective, kqueue is the alternative way, but not implemented yet.

EAL initializes the mapping between event file descriptors and interrupt vectors, while each device initializes the mapping between interrupt vectors and queues. In this way, EAL actually is unaware of the interrupt cause on the specific vector. The eth_dev driver takes responsibility to program the latter mapping.

***
**Note**

Per queue RX interrupt event is only allowed in VFIO which supports multiple MSI-X vector. In UIO, the RX interrupt together with other interrupt causes shares the same vector. In this case, when RX interrupt and LSC(link status change) interrupt are both enabled**(intr_conf.lsc == 1 && intr_conf.rxq == 1)**, only the former is capable.
***
The RX interrupt are controlled/enabled/disabled by ethdev APIs - **‘rte_eth_dev_rx_intr_*’**. They return failure if the PMD hasn’t support them yet. The **intr_conf.rxq** flag is used to turn on the capability of RX interrupt per device.



#### Device Removal Event

This event is triggered by a device being removed at a bus level. Its underlying resources may have been made unavailable (i.e. PCI mappings unmapped). The PMD must make sure that on such occurrence, the application can still safely use its callbacks.

This event can be subscribed to in the same way one would subscribe to a link status change event. The execution context is thus the same, i.e. it is the dedicated interrupt host thread.

Considering this, it is likely that an application would want to close a device having emitted a Device Removal Event. In such case, calling **rte_eth_dev_close()** can trigger it to unregister its own Device Removal Event callback. Care must be taken not to close the device from the interrupt handler context. It is necessary to reschedule such closing operation.

#### Blacklisting

The EAL PCI device blacklist functionality can be used to mark certain NIC ports as blacklisted, so **they are ignored by the DPDK.** The ports to be blacklisted are identified using the PCIe* description **(Domain:Bus:Device.Function).**

## Memory Segments and Memory Zones (memzone)

The mapping of physical memory is provided by this feature in the EAL. As physical memory can have gaps, the memory is described in a table of descriptors, and each descriptor (called **rte_memseg** ) describes a contiguous portion of memory.

On top of this, the memzone allocator’s role is to reserve contiguous portions of physical memory. These zones are identified by a unique name when the memory is reserved.

The rte_memzone descriptors are also located in the configuration structure. This structure is accessed using **rte_eal_get_configuration()**. The lookup (by name) of a memory zone returns a descriptor containing the physical address of the memory zone.

Memory zones can be reserved with specific start address alignment by supplying the align parameter (by default, they are aligned to cache line size). The alignment value should be a power of two and not less than the cache line size (64 bytes). Memory zones can also be reserved from either 2 MB or 1 GB hugepages, provided that both are available on the system.


## DPDK Memory initialization

#### internal configuration
struct internal_config internal_config;

rte_config.mem_config;

1. eal_hugepage_info_init()
2. rte_eal_config_create()
3. rte_eal_hugepage_init()


***


```c
/*
 * internal configuration structure for the number, size and
 * mount points of hugepages
 */
struct hugepage_info {
        uint64_t hugepage_sz;   /**< size of a huge page */
        const char *hugedir;    /**< dir where hugetlbfs is mounted */
        uint32_t num_pages[RTE_MAX_NUMA_NODES];
                                /**< number of hugepages of that size on each socket */
        int lock_descriptor;    /**< file descriptor for hugepage dir */
};

/**
 * internal configuration
 */
struct internal_config {
        volatile size_t memory;           /**< amount of asked memory */
        volatile unsigned force_nchannel; /**< force number of channels */
        volatile unsigned force_nrank;    /**< force number of ranks */
        volatile unsigned no_hugetlbfs;   /**< true to disable hugetlbfs */
        unsigned hugepage_unlink;         /**< true to unlink backing files */
        volatile unsigned no_pci;         /**< true to disable PCI */
        volatile unsigned no_hpet;        /**< true to disable HPET */
        volatile unsigned vmware_tsc_map; /**< true to use VMware TSC mapping
                                                                                * instead of native TSC */
        volatile unsigned no_shconf;      /**< true if there is no shared config */
        volatile unsigned create_uio_dev; /**< true to create /dev/uioX devices */
        volatile enum rte_proc_type_t process_type; /**< multi-process proc type */
        /** true to try allocating memory on specific sockets */
        volatile unsigned force_sockets;
        volatile uint64_t socket_mem[RTE_MAX_NUMA_NODES]; /**< amount of memory per socket */
        uintptr_t base_virtaddr;          /**< base address to try and reserve memory from */
        volatile int syslog_facility;     /**< facility passed to openlog() */
        /** default interrupt mode for VFIO */
        volatile enum rte_intr_mode vfio_intr_mode;
        const char *hugefile_prefix;      /**< the base filename of hugetlbfs files */
        const char *hugepage_dir;         /**< specific hugetlbfs directory to use */
        const char *mbuf_pool_ops_name;   /**< mbuf pool ops name */
        unsigned num_hugepage_sizes;      /**< how many sizes on this system */
        struct hugepage_info hugepage_info[MAX_HUGEPAGE_SIZES];
};


/*
 * The global RTE configuration structure.
 */
struct rte_config {
        uint32_t master_lcore;       /**< Id of the master lcore */
        uint32_t lcore_count;        /**< Number of available logical cores. */
        uint32_t service_lcore_count;/**< Number of available service cores. */
        enum rte_lcore_role_t lcore_role[RTE_MAX_LCORE]; /**< State of cores. */

        /** Primary or secondary configuration */
        enum rte_proc_type_t process_type;

        /** PA or VA mapping mode */
        enum rte_iova_mode iova_mode;

        /**
         * Pointer to memory configuration, which may be shared across multiple
         * DPDK instances
         */
        struct rte_mem_config *mem_config;
} __attribute__((__packed__));


/**
 * the structure for the memory configuration for the RTE.
 * Used by the rte_config structure. It is separated out, as for multi-process
 * support, the memory details should be shared across instances
 */
struct rte_mem_config {
        volatile uint32_t magic;   /**< Magic number - Sanity check. */

        /* memory topology */
        uint32_t nchannel;    /**< Number of channels (0 if unknown). */
        uint32_t nrank;       /**< Number of ranks (0 if unknown). */

        /**
         * current lock nest order
         *  - qlock->mlock (ring/hash/lpm)
         *  - mplock->qlock->mlock (mempool)
         * Notice:
         *  *ALWAYS* obtain qlock first if having to obtain both qlock and mlock
         */
        rte_rwlock_t mlock;   /**< only used by memzone LIB for thread-safe. */
        rte_rwlock_t qlock;   /**< used for tailq operation for thread safe. */
        rte_rwlock_t mplock;  /**< only used by mempool LIB for thread-safe. */

        uint32_t memzone_cnt; /**< Number of allocated memzones */

        /* memory segments and zones */
        struct rte_memseg memseg[RTE_MAX_MEMSEG];    /**< Physmem descriptors. */
        struct rte_memzone memzone[RTE_MAX_MEMZONE]; /**< Memzone descriptors. */

        struct rte_tailq_head tailq_head[RTE_MAX_TAILQ]; /**< Tailqs for objects */

        /* Heaps of Malloc per socket */
        struct malloc_heap malloc_heaps[RTE_MAX_NUMA_NODES];

        /* address of mem_config in primary process. used to map shared config into
         * exact same address the primary process maps it.
         */
        uint64_t mem_cfg_addr;
} __attribute__((__packed__));



/**
 * Structure used to store informations about hugepages that we mapped
 * through the files in hugetlbfs.
 */
struct hugepage_file {
        void *orig_va;      /**< virtual addr of first mmap() */
        void *final_va;     /**< virtual addr of 2nd mmap() */
        uint64_t physaddr;  /**< physical addr */
        size_t size;        /**< the page size */
        int socket_id;      /**< NUMA socket ID */
        int file_id;        /**< the '%d' in HUGEFILE_FMT */
        int memseg_id;      /**< the memory segment to which page belongs */
        char filepath[MAX_HUGEPAGE_PATH]; /**< path to backing file on filesystem */
};


/**
 * Physical memory segment descriptor.
 */
struct rte_memseg {
        phys_addr_t phys_addr;      /**< Start physical address. */
        RTE_STD_C11
        union {
                void *addr;         /**< Start virtual address. */
                uint64_t addr_64;   /**< Makes sure addr is always 64 bits */
        };
        size_t len;               /**< Length of the segment. */
        uint64_t hugepage_sz;       /**< The pagesize of underlying memory */
        int32_t socket_id;          /**< NUMA socket ID. */
        uint32_t nchannel;          /**< Number of channels. */
        uint32_t nrank;             /**< Number of ranks. */
} __rte_packed;

rte_memseg
rte_mem_config
rte_config
internal_config
hugepage_info
hugepage_file

```

***
```c
/* structure to hold a pair of head/tail values and other metadata */
struct rte_ring_headtail {
        volatile uint32_t head;  /**< Prod/consumer head. */
        volatile uint32_t tail;  /**< Prod/consumer tail. */
        uint32_t single;         /**< True if single prod/cons */
};

/**
 * An RTE ring structure.
 *
 * The producer and the consumer have a head and a tail index. The particularity
 * of these index is that they are not between 0 and size(ring). These indexes
 * are between 0 and 2^32, and we mask their value when we access the ring[]
 * field. Thanks to this assumption, we can do subtractions between 2 index
 * values in a modulo-32bit base: that's why the overflow of the indexes is not
 * a problem.
 */
struct rte_ring {
        /*   
         * Note: this field kept the RTE_MEMZONE_NAMESIZE size due to ABI
         * compatibility requirements, it could be changed to RTE_RING_NAMESIZE
         * next time the ABI changes
         */
        char name[RTE_MEMZONE_NAMESIZE] __rte_cache_aligned; /**< Name of the ring. */
        int flags;               /**< Flags supplied at creation. */
        const struct rte_memzone *memzone;
                        /**< Memzone, if any, containing the rte_ring */
        uint32_t size;           /**< Size of ring. */
        uint32_t mask;           /**< Mask (size-1) of ring. */
        uint32_t capacity;       /**< Usable size of ring */

        /** Ring producer status. */
        struct rte_ring_headtail prod __rte_aligned(PROD_ALIGN);

        /** Ring consumer status. */
        struct rte_ring_headtail cons __rte_aligned(CONS_ALIGN);
};


=========================================================================


/**
 * @internal
 * The generic data structure associated with each ethernet device.
 *
 * Pointers to burst-oriented packet receive and transmit functions are
 * located at the beginning of the structure, along with the pointer to
 * where all the data elements for the particular device are stored in shared
 * memory. This split allows the function pointer and driver data to be per-
 * process, while the actual configuration data for the device is shared.
 */
struct rte_eth_dev {
        eth_rx_burst_t rx_pkt_burst; /**< Pointer to PMD receive function. */
        eth_tx_burst_t tx_pkt_burst; /**< Pointer to PMD transmit function. */
        eth_tx_prep_t tx_pkt_prepare; /**< Pointer to PMD transmit prepare function. */
        struct rte_eth_dev_data *data;  /**< Pointer to device data */
        const struct eth_dev_ops *dev_ops; /**< Functions exported by PMD */
        struct rte_device *device; /**< Backing device */
        struct rte_intr_handle *intr_handle; /**< Device interrupt handle */
        /** User application callbacks for NIC interrupts */
        struct rte_eth_dev_cb_list link_intr_cbs;
        /**
         * User-supplied functions called from rx_burst to post-process
         * received packets before passing them to the user
         */
        struct rte_eth_rxtx_callback *post_rx_burst_cbs[RTE_MAX_QUEUES_PER_PORT];
        /**
         * User-supplied functions called from tx_burst to pre-process
         * received packets before passing them to the driver for transmission.
         */
        struct rte_eth_rxtx_callback *pre_tx_burst_cbs[RTE_MAX_QUEUES_PER_PORT];
        enum rte_eth_dev_state state; /**< Flag indicating the port state */
} __rte_cache_aligned;


/**
 * @internal
 * The data part, with no function pointers, associated with each ethernet device.
 *      
 * This structure is safe to place in shared memory to be common among different
 * processes in a multi-process configuration.
 */
struct rte_eth_dev_data {
        char name[RTE_ETH_NAME_MAX_LEN]; /**< Unique identifier name */

        void **rx_queues; /**< Array of pointers to RX queues. */
        void **tx_queues; /**< Array of pointers to TX queues. */
        uint16_t nb_rx_queues; /**< Number of RX queues. */
        uint16_t nb_tx_queues; /**< Number of TX queues. */

        struct rte_eth_dev_sriov sriov;    /**< SRIOV data */

        void *dev_private;              /**< PMD-specific private data */

        struct rte_eth_link dev_link;
        /**< Link-level information & status */

        struct rte_eth_conf dev_conf;   /**< Configuration applied to device. */
        uint16_t mtu;                   /**< Maximum Transmission Unit. */

        uint32_t min_rx_buf_size;
        /**< Common rx buffer size handled by all queues */

        uint64_t rx_mbuf_alloc_failed; /**< RX ring mbuf allocation failures. */
        struct ether_addr* mac_addrs;/**< Device Ethernet Link address. */
        uint64_t mac_pool_sel[ETH_NUM_RECEIVE_MAC_ADDR];
        /** bitmap array of associating Ethernet MAC addresses to pools */
        struct ether_addr* hash_mac_addrs;
        /** Device Ethernet MAC addresses of hash filtering. */
        uint16_t port_id;           /**< Device [external] port identifier. */
        __extension__
        uint8_t promiscuous   : 1, /**< RX promiscuous mode ON(1) / OFF(0). */
                scattered_rx : 1,  /**< RX of scattered packets is ON(1) / OFF(0) */
                all_multicast : 1, /**< RX all multicast mode ON(1) / OFF(0). */
                dev_started : 1,   /**< Device state: STARTED(1) / STOPPED(0). */
                lro         : 1;   /**< RX LRO is ON(1) / OFF(0) */
        uint8_t rx_queue_state[RTE_MAX_QUEUES_PER_PORT];
        /** Queues state: STARTED(1) / STOPPED(0) */
        uint8_t tx_queue_state[RTE_MAX_QUEUES_PER_PORT];
        /** Queues state: STARTED(1) / STOPPED(0) */
        uint32_t dev_flags; /**< Capabilities */
        enum rte_kernel_driver kdrv;    /**< Kernel driver passthrough */
        int numa_node;  /**< NUMA node connection */
        struct rte_vlan_filter_conf vlan_filter_conf;
        /**< VLAN filter configuration. */
};

==========================================================================
How to operate the user space Driver register, take pci address as an example:

struct rte_pci_bus rte_pci_bus = {
        .bus = {
                .scan = rte_pci_scan,
                .probe = rte_pci_probe,
                .find_device = pci_find_device,
                .plug = pci_plug,
                .unplug = pci_unplug,
                .parse = pci_parse,
                .get_iommu_class = rte_pci_get_iommu_class,
        },
        .device_list = TAILQ_HEAD_INITIALIZER(rte_pci_bus.device_list),
        .driver_list = TAILQ_HEAD_INITIALIZER(rte_pci_bus.driver_list),
};

rte_pci_scan
  pci_scan_one (lib/librte_eal/bsdapp/eal/eal_pci.c)
  {
    ....
    dev->mem_resource[i].addr = (void *)(bar.pbi_base & ~((uint64_t)0xf));
    dev->mem_resource[i].phys_addr = bar.pbi_base & ~((uint64_t)0xf);
    ....
  }  

/*
 * It means the pci address is marked during scan the pci mem/io space
 */

/*
 * Later, go through the special networking device Init, the pci mem/io * * space will be valued to special structure, for example below:
 *
 */

eth_ixgbe_dev_init()  (drivers/net/ixgbe/ixgbe_ethdev.c)
{
....
  hw->hw_addr = (void *)pci_dev->mem_resource[0].addr;
....
}

/*
 * Let check an example, how to operate driver register
 *
 */

#define IXGBE_WRITE_REG(hw, reg, value) \
        IXGBE_PCI_REG_WRITE(IXGBE_PCI_REG_ADDR((hw), (reg)), (value))

#define IXGBE_PCI_REG_WRITE(reg, value)                 \
        rte_write32((rte_cpu_to_le_32(value)), reg)

#define IXGBE_PCI_REG_ADDR(hw, reg) \
        ((volatile uint32_t *)((char *)(hw)->hw_addr + (reg)))

static __rte_always_inline void
rte_write32(uint32_t value, volatile void *addr)
{
        rte_io_wmb();
        rte_write32_relaxed(value, addr);
}

      static __rte_always_inline void
rte_write32_relaxed(uint32_t value, volatile void *addr)
{
        *(volatile uint32_t *)addr = value;
}

IXGBE_WRITE_REG
  IXGBE_PCI_REG_WRITE---->IXGBE_PCI_REG_ADDR ----> get the pci address
    rte_write32
      rte_write32_relaxed
        (volatile uint32_t *)addr = value;

/*
 * So, what's time to map the pci addess to user space? check the flow below
 *
 */

pci_uio_map_resource_by_index
  pci_map_resource
    mapaddr = mmap(requested_addr, size, PROT_READ | PROT_WRITE,MAP_SHARED | additional_flags, fd, offset);

pci_uio_map_resource_by_index() (lib/librte_eal/linuxapp/eal/eal_pci_uio.c)
{
....
  mapaddr = pci_map_resource(pci_map_addr, fd, 0,(size_t)dev->mem_resource[res_idx].len, 0);
....
  maps[map_idx].phaddr = dev->mem_resource[res_idx].phys_addr;
  maps[map_idx].addr = mapaddr;
....
}

/*
 * Check the information above, the hw_addr is the mmap address, it can be operated directly.
 *
 * Done, this is the progress of that how to operate the user space driver register
 *
 */
==========================================================================
Regarding how to crate a application for dpdk, please make this function as an example below:

/*
 * The key functions:
 * 1. rte_eth_dev_configure
 * 2. rte_eth_rx_queue_setup
 * 3. rte_eth_tx_queue_setup
 * 4. rte_eth_dev_start
 * 
 */

examples/kni/main.c

/* Initialise a single port on an Ethernet device */
static void
init_port(uint8_t port)
{
        int ret;
        uint16_t nb_rxd = NB_RXD;
        uint16_t nb_txd = NB_TXD;

        /* Initialise device and RX/TX queues */
        RTE_LOG(INFO, APP, "Initialising port %u ...\n", (unsigned)port);
        fflush(stdout);
        ret = rte_eth_dev_configure(port, 1, 1, &port_conf);
        if (ret < 0)
                rte_exit(EXIT_FAILURE, "Could not configure port%u (%d)\n",
                            (unsigned)port, ret);

        ret = rte_eth_dev_adjust_nb_rx_tx_desc(port, &nb_rxd, &nb_txd);
        if (ret < 0)
                rte_exit(EXIT_FAILURE, "Could not adjust number of descriptors "
                                "for port%u (%d)\n", (unsigned)port, ret);

        ret = rte_eth_rx_queue_setup(port, 0, nb_rxd,
                rte_eth_dev_socket_id(port), NULL, pktmbuf_pool);
        if (ret < 0)
                rte_exit(EXIT_FAILURE, "Could not setup up RX queue for "
                                "port%u (%d)\n", (unsigned)port, ret);

        ret = rte_eth_tx_queue_setup(port, 0, nb_txd,
                rte_eth_dev_socket_id(port), NULL);
        if (ret < 0)
                rte_exit(EXIT_FAILURE, "Could not setup up TX queue for "
                                "port%u (%d)\n", (unsigned)port, ret);

        ret = rte_eth_dev_start(port);
        if (ret < 0)
                rte_exit(EXIT_FAILURE, "Could not start port%u (%d)\n",
                                                (unsigned)port, ret);

        if (promiscuous_on)
                rte_eth_promiscuous_enable(port);
}




```
***
