#**Intel IOMMU**

##**Agenda** 

**Why**  
**Hardware Perspective**  
**Software Perspective**  


##**Why** 

According to the [Intel VT-d SPEC] [1], Section 2.5, IOMMU provides several
advantages:  
  
1. Device protection and isolation  
2. DMA remapping  

##**Hardware Perspective**

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

###**Domain**

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


##**Software Perspective**


[1]: http://www.intel.com/content/www/us/en/embedded/technology/virtualization/vt-directed-io-spec.html
