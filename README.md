.. SPDX-License-Identifier: GPL-2.0

Kernel driver amd_energy
==========================

Supported chips:

* AMD Family 17h Processors: Model 30h

* AMD Family 19h Processors: Model 01h and 30h

* AMD Family 19h Processors: Model 01h and 10h

  Prefix: 'amd_energy'

  Addresses used:  RAPL MSRs

  Datasheets:

  - Processor Programming Reference (PPR) for AMD Family 17h Model 01h, Revision B1 Processors

	https://developer.amd.com/wp-content/resources/55570-B1_PUB.zip

  - Preliminary Processor Programming Reference (PPR) for AMD Family 17h Model 31h, Revision B0 Processors

	https://developer.amd.com/wp-content/resources/56176_ppr_Family_17h_Model_71h_B0_pub_Rev_3.06.zip

  - Preliminary Processor Programming Reference (PPR) for AMD Family 19h Model 1h, Revision B1 Processors

	https://www.amd.com/system/files/TechDocs/55898_pub.zip

Author: Naveen Krishna Chatradhi <nchatrad@amd.com>

Security: CVE-2020-12912
------------------------------

Refer 2020 tab in https://www.amd.com/en/corporate/product-security#paragraph-313561 for details

Description
-----------

The Energy driver exposes the energy counters that are
reported via the Running Average Power Limit (RAPL)
Model-specific Registers (MSRs) via the hardware monitor
(HWMON) sysfs interface.

1. Power, Energy and Time Units
   MSR_RAPL_POWER_UNIT/ C001_0299:
   shared with all cores in the socket

2. Energy consumed by each Core
   MSR_CORE_ENERGY_STATUS/ C001_029A:
   32-bitRO, Accumulator, core-level power reporting

3. Energy consumed by Socket
   MSR_PACKAGE_ENERGY_STATUS/ C001_029B:
   32-bitRO, Accumulator, socket-level power reporting,
   shared with all cores in socket

These registers are updated every 1ms and cleared on
reset of the system.

Note: If SMT is enabled, Linux enumerates all threads as cpus.
Since, the energy status registers are accessed at core level,
reading those registers from the sibling threads would result
in duplicate values. Hence, energy counter entries are not
populated for the siblings.

Energy Caluclation
------------------

Energy information (in Joules) is based on the multiplier,
1/2^ESU; where ESU is an unsigned integer read from
MSR_RAPL_POWER_UNIT register. Default value is 10000b,
indicating energy status unit is 15.3 micro-Joules increment.

Reported values are scaled as per the formula

scaled value = ((1/2^ESU) * (Raw value) * 1000000UL) in uJoules

Users calculate power for a given domain by calculating
	dEnergy/dTime for that domain.

Energy accumulation
--------------------------

Current, Socket energy status register is 32bit, assuming a 240W
2P system, the register would wrap around in

	2^32*15.3 e-6/240 * 2 = 547.60833024 secs to wrap(~9 mins)

The Core energy register may wrap around after several days.

To improve the wrap around time, a kernel thread is implemented
to accumulate the socket energy counters and one core energy counter
per run to a respective 64-bit counter. The kernel thread starts
running during probe, wakes up every 100secs and stops running
when driver is removed.

Frequency of the accumulator thread is set during the probe
based on the chosen energy unit resolution. For example
A. fine grain (1.625 micro J)
B. course grain (0.125 milli J)

A socket and core energy read would return the current register
value added to the respective energy accumulator.

On newer EPYC CPUs with 64bit RAPL energy MSRs, software accumulation
of energy counters is not required. Hence, accumulation is enabled
only on select EPYC CPUs with 32bit RAPL MSRs.

Sysfs attributes
----------------

=============== ========  =====================================
Attribute	Label	  Description
===============	========  =====================================

* For index N between [1] and [nr_cpus]

===============	========  ======================================
energy[N]_input EcoreX	  Core Energy   X = [0] to [nr_cpus - 1]
			  Measured input core energy
===============	========  ======================================

* For N between [nr_cpus] and [nr_cpus + nr_socks]

===============	========  ======================================
energy[N]_input EsocketX  Socket Energy X = [0] to [nr_socks -1]
			  Measured input socket energy
=============== ========  ======================================

Note: To address CVE-2020-12912, the visibility of the energy[N]_input
attributes is restricted to owner and groups only.

Build and Install
-----------------

Kernel development packages for the running kernel need to be installed
prior to building the Energy module. A Makefile is provided which should
work with most kernel source trees.

To build the kernel module:

#> make

To install the kernel module:

#> sudo make modules_install

To clean the kernel module build directory:

#> make clean


Loading
-------

If the Energy module was installed you should use the modprobe command to
load the module.

#> sudo modprobe amd_energy

The Energy module can also be loaded using insmod if the module was not
installed:

The Energy module can also be loaded using insmod if the module was not
installed:

#> sudo insmod ./amd_energy.ko


DKMS support
------------

Building Module with running version of kernel

Add the module to DKMS tree:
#> sudo dkms add ../amd_energy

Build the module using DKMS:
#> sudo dkms build -m amd_energy/1.0

Install the module using DKMS:
#> sudo dkms install --force amd_energy/1.0

Load the module:
#> sudo modprobe amd_energy

Building Module with specific version of kernel

Add the module to DKMS tree:
#> sudo dkms add ../amd_energy

Build the module using DKMS:
#> sudo dkms build amd_energy/1.0 -k linux_version

Install the module using DKMS:
#> sudo dkms install --force amd_energy/1.0 -k linux_version
Module is built: /lib/modules/linux_version/updates/dkms/

Notes: It is required to have specific linux verion header in /usr/src

To remove module from dkms tree
#> sudo dkms remove -m amd_energy/1.0 --all
#> sudo rm -rf /usr/src/amd_energy-1.0/
