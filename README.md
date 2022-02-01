# Setting up an Omni-Path fabric for evaluation
This document is a cheat-sheet for some Cornelis Omni-Path admin commands and procedures; it does not cover switch management.

This procedure is suitable for the installation of small clusters and evaluation projects. It would need to be adjusted for use in a production environment using a Cluster Management system.

In general, Omni-Path switches can be used in their out-of-box state. Some configuration and firmware updates should be done for a production environment, but are usually unecessary for small evaluation systems. Managing switches will be covered in a separate document.

## Installation

Prerequisites
- A cluster of two or more Linux servers running a RHEL-like Linux distro.
- Passwordless ssh from the headnode to all nodes.
- Omni-Path adapters installed in each node, and cabled to an Omni-Path switch.
- The Omni-Path host software bundle (e.g. IntelOPA-IFS.RHEL*-x86_64.\*.tgz) has been downloaded.
- Optional/Recommended: pdsh has been installed on the headnode ```yum install pdsh```.

Install the Omni-Path host stack on each node:
```
cd
tar xf /tmp/IntelOPA-IFS.RHEL*-x86_64.*.tgz
cd IntelOPA-IFS*
./INSTALL -a
```
At this point, you may want to setup the IP-over-Fabric settings (also known as IPoIB). Do this in the same way as for any other network interface by creating a ```/etc/sysconfig/network-scripts/ifcfg-ib0``` file, typically with a static IP address.
```
reboot
```
On the headnode, start the Fabric Manager. This assumes there is no SM/FM running on the switch.
```
systemctl enable opafm
systemctl start opafm
```
Check that the fabric is up and that the correct number of hosts and switches are present.
```
opafabricinfo
```
## Measure the MPI latency and bandwidth
Test the bandwidth and latency between a pair of nodes using MVAPICH2 and the OSU micro-benchmarks. These are provided pre-compiled with the IntelOPA software bundle.
```
source /usr/mpi/gcc/mvapich2-*-hfi/bin/mpivars.sh
cd /usr/mpi/gcc/mvapich2-*-hfi/tests/osu-micro*/mpi/pt2pt
mpirun -hosts node01,node02 ./osu_latency
mpirun -hosts node01,node02 ./osu_bw
```
Test the bandwidth and latency for all nodes using MVAPICH2 and deviation. The deviation program is provided pre-compiled with the IntelOPA software bundle.
```
source /usr/mpi/gcc/mvapich2-*-hfi/bin/mpivars.sh
cd /usr/mpi/gcc/mvapich2-*-hfi/tests/intel
pdsh -w node[01-16] -N -R exec echo %h | sort > /tmp/mpi_hosts
mpirun -hostfile /tmp/mpi_hosts ./deviation
```
Alternative, using OpenMPI
```
source /usr/mpi/gcc/openmpi-*-hfi/bin/mpivars.sh
cd /usr/mpi/gcc/openmpi-*-hfi/tests/osu-micro-benchmarks-*/mpi/pt2pt
mpirun --allow-run-as-root --host node01,node02 ./osu_latency
mpirun --allow-run-as-root --host node01,node02 ./osu_bw
pdsh -w node[01-16] -N -R exec echo %h | sort > /tmp/mpi_hosts
mpirun --allow-run-as-root --npernode 1 --hostfile /tmp/mpi_hosts ./deviation
```

## Diagnostics
Node-based commands

```opainfo``` Check status of the local adapter, link and cable.<br>
```lspci | grep HFI``` Look for Omni-Path adapters.<br>
```lspci -d :24f0 -vv | grep LnkSta:``` Check the PCIe connection status. Should be Speed 8GT/s, Width x16.<br>
```dmidecode | grep -A3 "BIOS Info"``` Check the BIOS version (Dell only?).<br>
```hfi1stats -n 1 | grep Open``` Check for open contexts - each MPI rank that uses PSM2 will open a context.<br>
```cat /proc/cpuinfo | grep MHz``` Check the frequency of the CPUs.<br>
```mpirun -hosts node01,node02 hostname``` Test that passwordless ssh and the MPI infrastructure are working.<br>

Fabric-based commands

```opafabricinfo``` Fabric summary<br>
```opareport -o none -C; sleep 60; opareport -o slowlinks -o errors``` Show the quality of the links in the fabric.<br>
```opaextractlids -q -F nodetype:FI``` List the hosts on the fabric.<br>
```opaextractlids -q -F nodetype:SW``` List the switches on the fabric.<br>
```opaextractsellinks -q``` List the links in the fabric.<br>







