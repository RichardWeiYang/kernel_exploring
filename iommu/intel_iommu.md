#**Intel IOMMU**

##**Agenda** 

* [**Why**](#1)  
* [**Hardware Perspective**](#2)  
    * [**Domain**](#2.1)
* [**Software Perspective**](#3)  
    * [**Initialization**](#3.1)
        * [Parse DMAR Table from ACPI](#3.1.1)
        * [Configure DRHD Entry](#3.1.2)
    * [**Domains**](#3.1)

##**Why** <a id="1">

According to the [Intel VT-d SPEC] [1], Section 2.5, IOMMU provides several
advantages:  
  
1. Device protection and isolation  
2. DMA remapping  

##**Hardware Perspective** <a id="2">

This note focus on Device protection & isolation and DMA remapping.  

The general idea is simple: give each device a "Page Table" to map from the
device address space to the physical memory space. In order to achieve this,
Intel provides some hardware called VT-d, which could be accessed through
ACPI.  

The following chart gives a general idea:  
1. One system could have several IOMMU device, which is called DMA Remapping
Hardware Unit Definition  
2. Root Table Address stores the physical memory address to the in memory
configuration of "Page Table" for devices  
3. Each Root Entry represents a PCI bus  
4. Each Context Entry stores the physical memory address of the "Page Table"
for that device  


                          DMA Remapping Reporting Structure
                          +----------------------------------------+
                          |                                        |
                          .                                        .
                          .                                        .
                          .                                        .
                          |                                        |
                          +----------------------------------------+
                          |DMA Remapping Hardware Unit Definition  | is represented by dmar_drhd_unit
                          |                                        |
                          |            +---------------------------+
                          |            |Register Base Address      | dmar_drhd_unit->reg_base_addr
                          |            +---------------------------+
                          |                      |                 |
                          |                      |                 |
                          +----------------------------------------+
                                                 |
                                                 |
                                                 |
                                                 v
                          +----------------------------------------+
                          |                                        |
                          .                                        .
                          .                                        .
                          .                                        .
                          |                                        |
                     020h +----------------------------------------+
                          | Root Table Address     ----+           | intel_iommu->root_entry
                          +----------------------------------------+
                          |                            |           |
                          +----------------------------------------+
                                                       |
                                                       |ACPI / Hardware
                          --------------------------------------------------
                                                       |Memory
             +-----------------------------------------+
             |
             |                                                        +---------------------------------+          
             |                                             +--------->|Dev 31, Func 7     Context Entry |   ---->  IOMMU Table for each PCI device
             +-->+---------------------------------+       |          +---------------------------------+
                 |Bus #255              Root Entry |-------+          |Dev 30, Func 7     Context Entry |
                 +---------------------------------+                  +---------------------------------+
                 |Bus #254              Root Entry |                  |Dev 29, Func 7     Context Entry |
                 +---------------------------------+                  +---------------------------------+
                 |Bus #253                         |                  |                                 |
                 +---------------------------------+                  |                                 |
                 |                                 |                  |                                 |
                 |                                 |                  .                                 .
                 |                                 |                  .                                 .
                 .                                 .                  .                                 .
                 .                                 .                  |                                 |
                 .                                 .                  |                                 |
                 |                                 |                  +---------------------------------+
                 |                                 |                  |Dev 0, Func 2      Context Entry |
                 +---------------------------------+                  +---------------------------------+
                 |Bus #2                Root Entry |                  |Dev 0, Func 1      Context Entry |
                 +---------------------------------+                  +---------------------------------+
                 |Bus #1                Root Entry |                  |Dev 0, Func 0      Context Entry |
                 +---------------------------------+                  +---------------------------------+
                 |Bus #0                Root Entry |
                 +---------------------------------+

###**Domain** <a id="2.1">

For memory efficiency, several device could share the same device "Page
Table", eg. they are assigned to the save Virtual Machine. This is called
"Domain".  

Each domain is assigned an ID, which is written to the Context Entry. Each
device which has the same "Page Table" must has the same Domain ID, while each
device has the same "Page Table" prefer to have the same Domain ID. See [Intel
VT-d SPEC] [1], Section 3.4.3.  

       127                           87           72
       +----------------------------+---------------+--------------+
       |                            |DID            |              |
       +----------------------------+---------------+--------------+

Context Entry format form [Intel VT-d SPEC] [1], Section 9.3



       +----------------------------+----------+--------+          
       |Dev 31, Func 7              |DID#2     |        |   -------+---->  IOMMU Table
       +----------------------------+----------+--------+          |
       |Dev 30, Func 7                                  |          |
       +----------------------------+----------+--------+          |
       |Dev 29, Func 7              |DID#2     |        |   -------+
       +----------------------------+----------+--------+
       |                                                |
       |                                                |
       |                                                |
       .                                                .
       .                                                .
       .                                                .
       |                                                |
       |                                                |
       +------------------------------------------------+
       |Dev 0, Func 2                                   |
       +------------------------------------------------+
       |Dev 0, Func 1                                   |
       +------------------------------------------------+
       |Dev 0, Func 0                                   |
       +------------------------------------------------+

       Context Entry with the same DID could points to the same "Page Table"


##**Software Perspective** <a id="3">  

###**Initialization** <a id="3.1">  

Brief initialization sequence:  

```
    intel_iommu_init()  
        dmar_table_init()  
            parse_dmar_table()  
                dmar_table_detect()  
                dmar_parse_one_drhd()  
        init_dmars()  
            iommu_init_domains()  
            iommu_alloc_root_entry()  
            iommu_set_root_entry()  
            iommu_enable_translation()  
            dma_ops = &intel_dma_ops  
	    bus_set_iommu(&pci_bus_type, &intel_iommu_ops)  
```

It can be divided into two stages:  
1. Parse DMAR Table from ACPI  
2. Configure DRHD Entry  

####**Parse DMAR Table from ACPI** <a id="3.1.1">  

DMAR Table is accessed with ACPI interface.

```c
	#define ACPI_SIG_DMAR           "DMAR"	/* DMA Remapping table */
	status = acpi_get_table_with_size(ACPI_SIG_DMAR, 0,
				(struct acpi_table_header **)&dmar_tbl,
				&dmar_tbl_size);
```

The DMAR table in ACPI looks like this:

       +-------------------------------------------+
       |signature                                  |   "DMAR"
       |                                           |
       +-------------------------------------------+
       |length                                     |
       |                                           |
       +-------------------------------------------+
       .                                           .
       .                                           .
       .                                           .
       +-------------------------------------------+
       |                                           |
       |                                           |
       +-------------------------------------------+
       |Remapping Structure[]                      |
       |                                           |
       +-------------------------------------------+

           DMAR Table Format

For more detailed information, see [Intel VT-d SPEC] [1], Section 8.1.  

From the code and structure defined in SPEC, I guess the ACPI table is  
searched with the "signature". Then when it hits a "DMAR" table,  
dmar_walk_dmar_table() will walk through the "Remapping Structure", which  
contains the important information/configuration of the Intel IOMMU.  

```c
	ret = dmar_walk_dmar_table(dmar, &cb);
```

There are different kinds of Remapping Structure, see [Intel VT-d SPEC] [1], Section 8.2.  

     +-----------+------------------------------------------------+
     |Value      |Description                                     |
     +-----------+------------------------------------------------+
     |0          |DMA Remapping Hardware Unit Definition Structure| DRHD
     +-----------+------------------------------------------------+
     |1          |                                                |
     +-----------+------------------------------------------------+
     |2          |                                                |
     +-----------+------------------------------------------------+
     |3          |                                                |
     +-----------+------------------------------------------------+
     |4          |                                                |
     +-----------+------------------------------------------------+
     |>4         |                                                |
     +-----------+------------------------------------------------+

           Remapping Structure Type

In all of them, Type 0 DRHD structure is what this article focus. And its  
format is defined in [Intel VT-d SPEC] [1], Section 8.3.  


       +-------------------------------------------+
       |Tyep                                       |   "0"
       |                                           |
       +-------------------------------------------+
       |length                                     |
       |                                           |
       +-------------------------------------------+
       |Flags                                      |
       |                                           |
       +-------------------------------------------+
       |Reserved                                   |
       |                                           |
       +-------------------------------------------+
       |Segment Number                             |
       |                                           |
       +-------------------------------------------+
       |Register Base Address                      |   Familiar?
       |                                           |
       +-------------------------------------------+
       |Device Scope[]                             |
       |                                           |
       +-------------------------------------------+

        DRHD Format

Encounter the "Register Base Address" again~ This points to the configuration  
space of this DRHD unit. All the secrets live here. For more detail in this  
region, see [Intel VT-d SPEC] [1], Section 10.  

####**Configure DRHD Entry** <a id="3.1.2">  

After kernel gets basic information of the IOMMU device, it will configure DMA  
mapping through "Register Base Address". One word, the DMA mapping Page Table.  
The progress of configuring the Page Table happens in the whole life time of a  
device.  

At boot time, it does two basic things:  
+ Allocate Root Entry Table and set it  

```c
	iommu_alloc_root_entry()
	iommu_set_root_entry()
```

+ Enable the Translation  

```c
	iommu_enable_translation()
```

###**Domains** <a id="3.2">  


[1]: http://www.intel.com/content/www/us/en/embedded/technology/virtualization/vt-directed-io-spec.html
