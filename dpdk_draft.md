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


rte_config
  rte_mem_config
    rte_memseg      memseg[RTE_MAX_MEMSEG];
    rte_memzone     memzone[RTE_MAX_MEMZONE];
    malloc_heaps    malloc_heaps[RTE_MAX_NUMA_NODES];
    rte_tailq_head  tailq_head[RTE_MAX_TAILQ];


rte_mempool
  rte_memzone *mz;
  rte_mempool_cache *local_cache; /**< Per-lcore local cache */
  rte_mempool_objhdr_list elt_list; /**< List of objects in pool */
  rte_mempool_memhdr_list mem_list; /**< List of memory chunks */


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

==========================================================================
The key structures:

1. struct rte_eth_dev
2. struct rte_eth_dev_data /**< Pointer to device data */
3. struct eth_dev_ops      /**< Functions exported by PMD */
4.



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


/**< @internal Test if a port supports specific mempool ops */

/**
 * @internal A structure containing the functions exported by an Ethernet driver.
 */
struct eth_dev_ops {
        eth_dev_configure_t        dev_configure; /**< Configure device. */
        eth_dev_start_t            dev_start;     /**< Start device. */
        eth_dev_stop_t             dev_stop;      /**< Stop device. */
        eth_dev_set_link_up_t      dev_set_link_up;   /**< Device link up. */
        eth_dev_set_link_down_t    dev_set_link_down; /**< Device link down. */
        eth_dev_close_t            dev_close;     /**< Close device. */
        eth_dev_reset_t            dev_reset;     /**< Reset device. */
        eth_link_update_t          link_update;   /**< Get device link state. */

        eth_promiscuous_enable_t   promiscuous_enable; /**< Promiscuous ON. */
        eth_promiscuous_disable_t  promiscuous_disable;/**< Promiscuous OFF. */
        eth_allmulticast_enable_t  allmulticast_enable;/**< RX multicast ON. */
        eth_allmulticast_disable_t allmulticast_disable;/**< RX multicast OFF. */
        eth_mac_addr_remove_t      mac_addr_remove; /**< Remove MAC address. */
        eth_mac_addr_add_t         mac_addr_add;  /**< Add a MAC address. */
        eth_mac_addr_set_t         mac_addr_set;  /**< Set a MAC address. */
        eth_set_mc_addr_list_t     set_mc_addr_list; /**< set list of mcast addrs. */
        mtu_set_t                  mtu_set;       /**< Set MTU. */

        eth_stats_get_t            stats_get;     /**< Get generic device statistics. */
        eth_stats_reset_t          stats_reset;   /**< Reset generic device statistics. */
        eth_xstats_get_t           xstats_get;    /**< Get extended device statistics. */
        eth_xstats_reset_t         xstats_reset;  /**< Reset extended device statistics. */
        eth_xstats_get_names_t     xstats_get_names;
        /**< Get names of extended statistics. */
        eth_queue_stats_mapping_set_t queue_stats_mapping_set;
        /**< Configure per queue stat counter mapping. */

        eth_dev_infos_get_t        dev_infos_get; /**< Get device info. */
        eth_rxq_info_get_t         rxq_info_get; /**< retrieve RX queue information. */
        eth_txq_info_get_t         txq_info_get; /**< retrieve TX queue information. */
        eth_fw_version_get_t       fw_version_get; /**< Get firmware version. */
        eth_dev_supported_ptypes_get_t dev_supported_ptypes_get;
        /**< Get packet types supported and identified by device. */

        vlan_filter_set_t          vlan_filter_set; /**< Filter VLAN Setup. */
        vlan_tpid_set_t            vlan_tpid_set; /**< Outer/Inner VLAN TPID Setup. */
        vlan_strip_queue_set_t     vlan_strip_queue_set; /**< VLAN Stripping on queue. */
        vlan_offload_set_t         vlan_offload_set; /**< Set VLAN Offload. */
        vlan_pvid_set_t            vlan_pvid_set; /**< Set port based TX VLAN insertion. */

        eth_queue_start_t          rx_queue_start;/**< Start RX for a queue. */
        eth_queue_stop_t           rx_queue_stop; /**< Stop RX for a queue. */
        eth_queue_start_t          tx_queue_start;/**< Start TX for a queue. */
        eth_queue_stop_t           tx_queue_stop; /**< Stop TX for a queue. */
        eth_rx_queue_setup_t       rx_queue_setup;/**< Set up device RX queue. */
        eth_queue_release_t        rx_queue_release; /**< Release RX queue. */
        eth_rx_queue_count_t       rx_queue_count;
        /**< Get the number of used RX descriptors. */
        eth_rx_descriptor_done_t   rx_descriptor_done; /**< Check rxd DD bit. */
        eth_rx_descriptor_status_t rx_descriptor_status;
        /**< Check the status of a Rx descriptor. */
        eth_tx_descriptor_status_t tx_descriptor_status;
        /**< Check the status of a Tx descriptor. */
        eth_rx_enable_intr_t       rx_queue_intr_enable;  /**< Enable Rx queue interrupt. */
        eth_rx_disable_intr_t      rx_queue_intr_disable; /**< Disable Rx queue interrupt. */
        eth_tx_queue_setup_t       tx_queue_setup;/**< Set up device TX queue. */
        eth_queue_release_t        tx_queue_release; /**< Release TX queue. */
        eth_tx_done_cleanup_t      tx_done_cleanup;/**< Free tx ring mbufs */

        eth_dev_led_on_t           dev_led_on;    /**< Turn on LED. */
        eth_dev_led_off_t          dev_led_off;   /**< Turn off LED. */

        flow_ctrl_get_t            flow_ctrl_get; /**< Get flow control. */
        flow_ctrl_set_t            flow_ctrl_set; /**< Setup flow control. */
        priority_flow_ctrl_set_t   priority_flow_ctrl_set; /**< Setup priority flow control. */

        eth_uc_hash_table_set_t    uc_hash_table_set; /**< Set Unicast Table Array. */
        eth_uc_all_hash_table_set_t uc_all_hash_table_set; /**< Set Unicast hash bitmap. */

        eth_mirror_rule_set_t      mirror_rule_set; /**< Add a traffic mirror rule. */
        eth_mirror_rule_reset_t    mirror_rule_reset; /**< reset a traffic mirror rule. */

        eth_udp_tunnel_port_add_t  udp_tunnel_port_add; /** Add UDP tunnel port. */
        eth_udp_tunnel_port_del_t  udp_tunnel_port_del; /** Del UDP tunnel port. */
        eth_l2_tunnel_eth_type_conf_t l2_tunnel_eth_type_conf;
        /** Config ether type of l2 tunnel. */
        eth_l2_tunnel_offload_set_t   l2_tunnel_offload_set;
        /** Enable/disable l2 tunnel offload functions. */

        eth_set_queue_rate_limit_t set_queue_rate_limit; /**< Set queue rate limit. */

        rss_hash_update_t          rss_hash_update; /** Configure RSS hash protocols. */
        rss_hash_conf_get_t        rss_hash_conf_get; /** Get current RSS hash configuration. */
        reta_update_t              reta_update;   /** Update redirection table. */
        reta_query_t               reta_query;    /** Query redirection table. */

        eth_filter_ctrl_t          filter_ctrl; /**< common filter control. */

        eth_get_dcb_info           get_dcb_info; /** Get DCB information. */

        eth_timesync_enable_t      timesync_enable;
        /** Turn IEEE1588/802.1AS timestamping on. */
        eth_timesync_disable_t     timesync_disable;
        /** Turn IEEE1588/802.1AS timestamping off. */
        eth_timesync_read_rx_timestamp_t timesync_read_rx_timestamp;
        /** Read the IEEE1588/802.1AS RX timestamp. */
        eth_timesync_read_tx_timestamp_t timesync_read_tx_timestamp;
        /** Read the IEEE1588/802.1AS TX timestamp. */
        eth_timesync_adjust_time   timesync_adjust_time; /** Adjust the device clock. */
        eth_timesync_read_time     timesync_read_time; /** Get the device clock time. */
        eth_timesync_write_time    timesync_write_time; /** Set the device clock time. */

        eth_xstats_get_by_id_t     xstats_get_by_id;
        /**< Get extended device statistic values by ID. */
        eth_xstats_get_names_by_id_t xstats_get_names_by_id;
        /**< Get name of extended device statistics by ID. */

        eth_tm_ops_get_t tm_ops_get;
        /**< Get Traffic Management (TM) operations. */
        eth_pool_ops_supported_t pool_ops_supported;
        /**< Test if a port supports specific mempool ops */
};

```


```
Regarding how to init the rx_queues[], which will be used in rx progress


main() (examples/l3fwd-acl/main.c)
  rte_eth_rx_queue_setup
    dev->dev_ops->rx_queue_setup
      ixgbe_dev_rx_queue_setup
        dev->data->rx_queues[queue_idx] = rxq;

static const struct eth_dev_ops ixgbe_eth_dev_ops = {
....
  .rx_queue_setup       = ixgbe_dev_rx_queue_setup,
....
}

ixgbe_dev_rx_queue_setup()  (drivers/net/ixgbe/ixgbe_rxtx.c)
{
....
  dev->data->rx_queues[queue_idx] = rxq;
....
}

rte_eth_rx_queue_setup()
{
...
  ret = (*dev->dev_ops->rx_queue_setup)(dev, rx_queue_id, nb_rx_desc,
                                        socket_id, &local_conf, mp);
...
}

main()  (examples/l3fwd-acl/main.c)
{
....
  for (lcore_id = 0; lcore_id < RTE_MAX_LCORE; lcore_id++) {
  ....
    for (queue = 0; queue < qconf->n_rx_queue; ++queue) {
    ....
      ret = rte_eth_rx_queue_setup(portid, queueid, nb_rxd,
                                   socketid, NULL,pktmbuf_pool[socketid]);
    ....
    }
  }
....  
}

/*
 * One more time, this is the typical example for dpdk
 *
 */
main()
{
  rte_eth_dev_configure()
  rte_eth_tx_queue_setup()
  rte_eth_rx_queue_setup()
  rte_eth_dev_start()

}


```

```
***
Regarding the dpaa2 driver, this is the call chain below:

drivers/net/dpaa2/dpaa2_ethdev.c


static struct rte_dpaa2_driver rte_dpaa2_pmd = {
        .drv_type = DPAA2_ETH,
        .probe = rte_dpaa2_probe,
        .remove = rte_dpaa2_remove,
};

rte_dpaa2_probe()
{
....
 eth_dev->data->dev_private = rte_zmalloc
....
  /*
   * Init the Key structure
   * 1. rte_dpaa2_device
   * 2. dpaa2_dev_priv
   * 3. fsl_mc_io *dpni_dev
   * 4. dpni_buffer_layout
   */
dpaa2_dev_init()
{
....
  dpaa2_alloc_rx_tx_queues() /* Init dpaa2_queue */
  {
  ....
    struct dpaa2_queue *mc_q, *mcq;

    mc_q = rte_malloc(NULL, sizeof(struct dpaa2_queue) * tot_queues,RTE_CACHE_LINE_SIZE);

    for (i = 0; i < priv->nb_rx_queues; i++) {
    ....
      priv->rx_vq[i] = mc_q++;
    ....
    }
  }
....
  eth_dev->dev_ops = &dpaa2_ethdev_ops;
  eth_dev->data->dev_flags |= RTE_ETH_DEV_INTR_LSC;

  eth_dev->rx_pkt_burst = dpaa2_dev_prefetch_rx;
  eth_dev->tx_pkt_burst = dpaa2_dev_tx;
....
}  

```

```
***
Regarding local_cache:

1. Create the local_cache

rte_mempool_create_empty()  (lib/librte_mempool/rte_mempool.c)
{
....
  mp->local_cache = (struct rte_mempool_cache *)RTE_PTR_ADD(mp, MEMPOOL_HEADER_SIZE(mp, 0));
....
}

2. How to get/put local_cache

static __rte_always_inline struct rte_mempool_cache *
rte_mempool_default_cache(struct rte_mempool *mp, unsigned lcore_id)
{
        if (mp->cache_size == 0)
                return NULL;

        if (lcore_id >= RTE_MAX_LCORE)
                return NULL;

        return &mp->local_cache[lcore_id];
}


rte_mempool_put_bulk()   (lib/librte_mempool/rte_mempool.h)
{
        struct rte_mempool_cache *cache;
        cache = rte_mempool_default_cache(mp, rte_lcore_id());
        rte_mempool_generic_put(mp, obj_table, n, cache);
}

static __rte_always_inline int
rte_mempool_get_bulk(struct rte_mempool *mp, void **obj_table, unsigned int n)
{
        struct rte_mempool_cache *cache;
        cache = rte_mempool_default_cache(mp, rte_lcore_id());
        return rte_mempool_generic_get(mp, obj_table, n, cache);
}

3. How to use local_cache, and when do we use this features, take ixgbe as an example

1) How to Init
------------------------------------------------------------------------
drivers/net/ixgbe/ixgbe_ethdev.c

static const struct eth_dev_ops ixgbe_eth_dev_ops = {
....
        .dev_start            = ixgbe_dev_start,
....
}

drivers/net/ixgbe/ixgbe_rxtx.c

ixgbe_dev_start
  ixgbe_dev_rx_init
    ixgbe_set_rx_function

ixgbe_set_rx_function()
{
....
  dev->rx_pkt_burst = ixgbe_recv_pkts
....  
}

2) How to receive packets via local cache or ring
-------------------------------------------------------------------------
dev->rx_pkt_burst
  ixgbe_recv_pkts
    rte_mbuf_raw_alloc() (lib/librte_mbuf/rte_mbuf.h)
      rte_mempool_get





rte_mempool_get()  (lib/librte_mempool/rte_mempool.h)
  rte_mempool_get_bulk
    rte_mempool_default_cache  /* allocate memeory from local cache */
    rte_mempool_generic_get    /* allocate memory from ring */
-------------------------------------------------------------------------

```

***
```
***
Regarding how to mmap dpaa/dpaa2 memory space, please check the function below:

Go through the c code, the relevent io memory space is get via "ioctl", so does it mean the kernel space dpaa/dpaa2 driver must be Init at a head of time?

(drivers/bus/fslmc/fslmc_vfio.c)
static int64_t vfio_map_mcp_obj(struct fslmc_vfio_group *group, char *mcp_obj)
{
....
        /* getting the mcp object's fd*/
        mc_fd = ioctl(group->fd, VFIO_GROUP_GET_DEVICE_FD, mcp_obj);
        if (mc_fd < 0) {
                FSLMC_VFIO_LOG(ERR, "error in VFIO get dev %s fd from group %d",
                               mcp_obj, group->fd);
                return v_addr;
        }

        /* getting device info*/
        ret = ioctl(mc_fd, VFIO_DEVICE_GET_INFO, &d_info);
        if (ret < 0) {
                FSLMC_VFIO_LOG(ERR, "error in VFIO getting DEVICE_INFO");
                goto MC_FAILURE;
        }

        /* getting device region info*/
        ret = ioctl(mc_fd, VFIO_DEVICE_GET_REGION_INFO, &reg_info);
        if (ret < 0) {
                FSLMC_VFIO_LOG(ERR, "error in VFIO getting REGION_INFO");
                goto MC_FAILURE;
        }

....
        v_addr = (uint64_t)mmap(NULL, reg_info.size,
                PROT_WRITE | PROT_READ, MAP_SHARED,
                mc_fd, reg_info.offset);
....
}

fslmc_process_mcp()   (drivers/bus/fslmc/fslmc_vfio.c)
{
....
  v_addr = vfio_map_mcp_obj(&vfio_group, dev_name);
....
  rte_mcp_ptr_list[0] = (void *)v_addr;
....
}

The dpaa/dpaa2 memory space information saved in global variable "rte_mcp_ptr_list"

-------------------------------------------------------------------------
This is the call chain about the dpaa2 mc io sapce mmap to user space

rte_fslmc_probe
  fslmc_vfio_process_group
    fslmc_process_mcp
      vfio_map_mcp_obj
        mmap

struct rte_fslmc_bus rte_fslmc_bus = {
        .bus = {
                .scan = rte_fslmc_scan,
                .probe = rte_fslmc_probe,
                .find_device = rte_fslmc_find_device,
        },
        .device_list = TAILQ_HEAD_INITIALIZER(rte_fslmc_bus.device_list),
        .driver_list = TAILQ_HEAD_INITIALIZER(rte_fslmc_bus.driver_list),
};

```
***

```
Regarding how to bind/unbind the networking driver to dpdk, it means what's the actions of dpdk_nic_bind.py, please check the informations below:

Example:
root@c0101:/sys/bus/pci/drivers/igb# lspci -nn | grep 350
04:00.0 Ethernet controller [0200]: Intel Corporation I350 Gigabit Network Connection [8086:1521] (rev 01)
04:00.1 Ethernet controller [0200]: Intel Corporation I350 Gigabit Network Connection [8086:1521] (rev 01)
root@c0101:/sys/bus/pci/drivers/igb# ls
0000:04:00.0  0000:04:00.1  bind  module  new_id  remove_id  uevent  unbind
root@c0101:/sys/bus/pci/drivers/igb# ll
total 0
drwxr-xr-x  2 root root    0 Dec  2 22:54 .
drwxr-xr-x 44 root root    0 Dec  2 22:54 ..
lrwxrwxrwx  1 root root    0 Dec 14 00:10 0000:04:00.0 -> ../../../../devices/pci0000:00/0000:00:01.1/0000:04:00.0
lrwxrwxrwx  1 root root    0 Dec 14 00:15 0000:04:00.1 -> ../../../../devices/pci0000:00/0000:00:01.1/0000:04:00.1
--w-------  1 root root 4096 Dec 14 00:15 bind
lrwxrwxrwx  1 root root    0 Dec 14 00:10 module -> ../../../../module/igb
--w-------  1 root root 4096 Dec 14 00:10 new_id
--w-------  1 root root 4096 Dec 14 00:10 remove_id
--w-------  1 root root 4096 Dec  2 22:54 uevent
--w-------  1 root root 4096 Dec 14 00:36 unbind
root@c0101:/sys/bus/pci/drivers/igb# echo 04:00.1 > unbind
-sh: echo: write error: No such device
root@c0101:/sys/bus/pci/drivers/igb# echo 0000:04:00.1 > unbind
root@c0101:/sys/bus/pci/drivers/igb#
root@c0101:/sys/bus/pci/drivers/igb#
root@c0101:/sys/bus/pci/drivers/igb#
root@c0101:/sys/bus/pci/drivers/igb# ifconfig -a
eth0      Link encap:Ethernet  HWaddr 00:25:90:c9:b3:52
          inet addr:128.224.155.26  Bcast:128.224.255.255  Mask:255.255.0.0
          inet6 addr: fe80::225:90ff:fec9:b352/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:1843103 errors:0 dropped:0 overruns:0 frame:0
          TX packets:109774 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:152401478 (145.3 MiB)  TX bytes:12005669 (11.4 MiB)
          Memory:dfd20000-dfd3ffff

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:3 errors:0 dropped:0 overruns:0 frame:0
          TX packets:3 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:784 (784.0 B)  TX bytes:784 (784.0 B)

sit0      Link encap:UNSPEC  HWaddr 00-00-00-00-30-30-30-00-00-00-00-00-00-00-00-00
          NOARP  MTU:1480  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

root@c0101:/sys/bus/pci/drivers/igb# /opt/dpdk/tools/dpdk_nic_bind.py --status

Network devices using DPDK-compatible driver
============================================
0000:07:00.0 '82576 Gigabit Network Connection' drv=igb_uio unused=
0000:07:00.1 '82576 Gigabit Network Connection' drv=igb_uio unused=

Network devices using kernel driver
===================================
0000:04:00.0 'I350 Gigabit Network Connection' if=eth0 drv=igb unused=igb_uio *Active*

Other network devices
=====================
0000:04:00.1 'I350 Gigabit Network Connection' unused=igb_uio
0000:81:00.0 'Device 7004' unused=igb_uio
root@c0101:/sys/bus/pci/drivers/igb# ls
0000:04:00.0  bind  module  new_id  remove_id  uevent  unbind
root@c0101:/sys/bus/pci/drivers/igb# ll
total 0
drwxr-xr-x  2 root root    0 Dec  2 22:54 .
drwxr-xr-x 44 root root    0 Dec  2 22:54 ..
lrwxrwxrwx  1 root root    0 Dec 14 00:10 0000:04:00.0 -> ../../../../devices/pci0000:00/0000:00:01.1/0000:04:00.0
--w-------  1 root root 4096 Dec 14 00:15 bind
lrwxrwxrwx  1 root root    0 Dec 14 00:10 module -> ../../../../module/igb
--w-------  1 root root 4096 Dec 14 00:10 new_id
--w-------  1 root root 4096 Dec 14 00:10 remove_id
--w-------  1 root root 4096 Dec  2 22:54 uevent
--w-------  1 root root 4096 Dec 14 00:41 unbind
root@c0101:/sys/bus/pci/drivers/igb# lspci -nn | grep 350
04:00.0 Ethernet controller [0200]: Intel Corporation I350 Gigabit Network Connection [8086:1521] (rev 01)
04:00.1 Ethernet controller [0200]: Intel Corporation I350 Gigabit Network Connection [8086:1521] (rev 01)
root@c0101:/sys/bus/pci/drivers/igb# echo 8086 1521 > new_id
root@c0101:/sys/bus/pci/drivers/igb# ll
total 0
drwxr-xr-x  2 root root    0 Dec  2 22:54 .
drwxr-xr-x 44 root root    0 Dec  2 22:54 ..
lrwxrwxrwx  1 root root    0 Dec 14 00:10 0000:04:00.0 -> ../../../../devices/pci0000:00/0000:00:01.1/0000:04:00.0
lrwxrwxrwx  1 root root    0 Dec 14 00:41 0000:04:00.1 -> ../../../../devices/pci0000:00/0000:00:01.1/0000:04:00.1
--w-------  1 root root 4096 Dec 14 00:15 bind
lrwxrwxrwx  1 root root    0 Dec 14 00:10 module -> ../../../../module/igb
--w-------  1 root root 4096 Dec 14 00:41 new_id
--w-------  1 root root 4096 Dec 14 00:10 remove_id
--w-------  1 root root 4096 Dec  2 22:54 uevent
--w-------  1 root root 4096 Dec 14 00:41 unbind
root@c0101:/sys/bus/pci/drivers/igb# ifconfig -a
eth0      Link encap:Ethernet  HWaddr 00:25:90:c9:b3:52
          inet addr:128.224.155.26  Bcast:128.224.255.255  Mask:255.255.0.0
          inet6 addr: fe80::225:90ff:fec9:b352/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:1843316 errors:0 dropped:0 overruns:0 frame:0
          TX packets:109874 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:152419152 (145.3 MiB)  TX bytes:12021125 (11.4 MiB)
          Memory:dfd20000-dfd3ffff

eth1      Link encap:Ethernet  HWaddr 00:25:90:c9:b3:53
          BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
          Memory:dfd00000-dfd1ffff

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:3 errors:0 dropped:0 overruns:0 frame:0
          TX packets:3 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:784 (784.0 B)  TX bytes:784 (784.0 B)

sit0      Link encap:UNSPEC  HWaddr 00-00-00-00-30-30-30-00-00-00-00-00-00-00-00-00
          NOARP  MTU:1480  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

root@c0101:/sys/bus/pci/drivers/igb# /opt/dpdk/tools/dpdk_nic_bind.py --status

Network devices using DPDK-compatible driver
============================================
0000:07:00.0 '82576 Gigabit Network Connection' drv=igb_uio unused=
0000:07:00.1 '82576 Gigabit Network Connection' drv=igb_uio unused=

Network devices using kernel driver
===================================
0000:04:00.0 'I350 Gigabit Network Connection' if=eth0 drv=igb unused=igb_uio *Active*
0000:04:00.1 'I350 Gigabit Network Connection' if=eth1 drv=igb unused=igb_uio

Other network devices
=====================
0000:81:00.0 'Device 7004' unused=igb_uio
root@c0101:/sys/bus/pci/drivers/igb# cd ..
root@c0101:/sys/bus/pci/drivers# cd igb_uio
root@c0101:/sys/bus/pci/drivers/igb_uio# ls
0000:07:00.0  0000:07:00.1  bind  module  new_id  remove_id  uevent  unbind
root@c0101:/sys/bus/pci/drivers/igb_uio# ll

=========================================================================
How to deal with the param of sysfs: DEVICE_ATTR

DEVICE_ATTR的使用

使用DEVICE_ATTR，可以在sys fs中添加“文件”，通过修改该文件内容，可以实现在运行过程中动态控制device的目的。
类似的还有DRIVER_ATTR，BUS_ATTR，CLASS_ATTR。
这几个东东的区别就是，DEVICE_ATTR对应的文件在/sys/devices/目录中对应的device下面。
而其他几个分别在driver，bus，class中对应的目录下。
这次主要介绍DEVICE_ATTR，其他几个类似。
在documentation/driver-model/Device.txt中有对DEVICE_ATTR的详细介绍，这儿主要说明使用方法。

先看看DEVICE_ATTR的原型：
DEVICE_ATTR(_name, _mode, _show, _store)
_name：名称，也就是将在sys fs中生成的文件名称。
_mode：上述文件的访问权限，与普通文件相同，UGO的格式。
_show：显示函数，cat该文件时，此函数被调用。
_store：写函数，echo内容到该文件时，此函数被调用。

看看我们怎么填充这些要素：
名称可以随便起一个，便于记忆，并能体现其功能即可。
模式可以为只读0444，只写0222，或者读写都行的0666。当然也可以对User\Group\Other进行区别。
显示和写入函数就需要实现了。

```
***
Regarding IOMMU/VFIO:
1. IOMMU

**IOMMU**
![Alt text](/pic/iommu.png)


**IOMMU Topology**
![Alt text](/pic/IOMMU_Topo.png)

2. VFIO
与KVM一样，用户态通过IOCTL与VFIO交互。可作为操作对象的几种文件描述符有：

   Container文件描述符
       打开/dev/vfio字符设备可得
   IOMMU group文件描述符
       打开/dev/vfio/N文件可得 (详见后文)
   Device文件描述符
       向IOMMU group文件描述符发起相关ioctl可得

逻辑上来说，IOMMU group是IOMMU操作的最小对象。某些IOMMU硬件支持将若干IOMMU group组成更大的单元。VFIO据此做出container的概念，可容纳多个IOMMU group。打开/dev/vfio文件即新建一个空的container。在VFIO中，container是IOMMU操作的最小对象。

要使用VFIO，需先将设备与原驱动拨离，并与VFIO绑定。

用VFIO访问硬件的步骤：

   打开设备所在IOMMU group在/dev/vfio/目录下的文件
   使用VFIO_GROUP_GET_DEVICE_FD得到表示设备的文件描述 (参数为设备名称，一个典型的PCI设备名形如0000:03.00.01)
   对设备进行read/write/mmap等操作

用VFIO配置IOMMU的步骤：

   打开/dev/vfio，得到container文件描述符
   用VFIO_SET_IOMMU绑定一种IOMMU实现层
   打开/dev/vfio/N，得到IOMMU group文件描述符
   用VFIO_GROUP_SET_CONTAINER将IOMMU group加入container
   用VFIO_IOMMU_MAP_DMA将此IOMMU group的DMA地址映射至进程虚拟地址空间



***

```
Regarding how to init per-cpu thread, and how to remote launch

rte_eal_init()
{
....
  pthread_create(eal_thread_loop)
....
}  

eal_thread_loop()
{
....
  eal_thread_set_affinity()
....  
          /* read on our pipe to get commands */
        while (1) {
                void *fct_arg;

                /* wait command */
                do {
                        n = read(m2s, &c, 1);
                } while (n < 0 && errno == EINTR);

                if (n <= 0)
                        rte_panic("cannot read on configuration pipe\n");

                lcore_config[lcore_id].state = RUNNING;

                /* send ack */
                n = 0;
                while (n == 0 || (n < 0 && errno == EINTR))
                        n = write(s2m, &c, 1);
                if (n < 0)
                        rte_panic("cannot write on configuration pipe\n");

                if (lcore_config[lcore_id].f == NULL)
                        rte_panic("NULL function pointer\n");

                /* call the function and store the return value */
                fct_arg = lcore_config[lcore_id].arg;
                ret = lcore_config[lcore_id].f(fct_arg);
                lcore_config[lcore_id].ret = ret;
                rte_wmb();
                lcore_config[lcore_id].state = FINISHED;
        }
....
}


/*
 * Send a message to a slave lcore identified by slave_id to call a
 * function f with argument arg. Once the execution is done, the
 * remote lcore switch in FINISHED state.
 */
int
rte_eal_remote_launch(int (*f)(void *), void *arg, unsigned slave_id)
{
        int n;
        char c = 0;
        int m2s = lcore_config[slave_id].pipe_master2slave[1];
        int s2m = lcore_config[slave_id].pipe_slave2master[0];

        if (lcore_config[slave_id].state != WAIT)
                return -EBUSY;

        lcore_config[slave_id].f = f;
        lcore_config[slave_id].arg = arg;

        /* send message */
        n = 0;
        while (n == 0 || (n < 0 && errno == EINTR))
                n = write(m2s, &c, 1);
        if (n < 0)
                rte_panic("cannot write on configuration pipe\n");

        /* wait ack */
        do {
                n = read(s2m, &c, 1);
        } while (n < 0 && errno == EINTR);

        if (n <= 0)
                rte_panic("cannot read on configuration pipe\n");

        return 0;
}

after set cpu affinity with thread, later, only go through read/write in rte_eal_remote_launch() to complete other function run in specific cpu

```
