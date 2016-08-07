**Intel IOMMU**
===============

**Agenda** 
-------------

Why
Hardware Perspective
Software Perspective


**Why** 
-------------

According to the [Intel VT-d SPEC] [1], Section 2.5, IOMMU provides several
advantages:
1. Device protection and isolation
2. DMA remapping

**Hardware Perspective**
-------------


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
ACPI / Hardware (?)                                    |
----------------------------------------------------------------------------
Memory                                                 |
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


**Software Perspective**
-------------


[1]: http://www.intel.com/content/www/us/en/embedded/technology/virtualization/vt-directed-io-spec.html
