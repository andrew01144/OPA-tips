# Setting up an Omni-Path fabric for evaluation
This document is a cheat-sheet for some Cornelis Omni-Path admin commands and procedures; it does not cover switch management.

This procedure is suitable for the installation of small clusters and evaluation projects. It would need to be adjusted for use in a production environment using a Cluster Management system.

In general, Omni-Path switches can be used in their out-of-box state. Some configuration and firmware updates should be done for a production environment, but are usually unecessary for small evaluation systems. Managing switches will be covered in a separate document.

## Installation

Prerequisites
- A cluster of two or more Linux servers running a RHEL-like Linux distro.
- Passwordless ssh from the headnode to all nodes.
- Omni-Path adapters installed in each node, and cabled to an Omni-Path switch.
- The Omni-Path host software bundle (e.g. CornelisOPX-OPXS.RHEL*-x86_64.*.tgz ) has been downloaded.
- Optional/Recommended: pdsh has been installed on the headnode ```yum install pdsh```.

Install the Omni-Path host stack on each node:
```
cd
tar xf /tmp/CornelisOPX-OPXS.RHEL*-x86_64.*.tgz
cd CornelisOPX*
./INSTALL -a
```
At this point, you may want to setup the IP-over-Fabric settings (also known as IPoIB). Do this in the same way as for any other network interface by creating a ```/etc/sysconfig/network-scripts/ifcfg-ib0``` file, typically with a static IP address.
```
# /etc/sysconfig/network-scripts/ifcfg-ib0
DEVICE=ib0
TYPE='InfiniBand'
BOOTPROTO=static
IPADDR=192.168.101.1
BROADCAST=192.168.101.255
NETWORK=192.168.101.0
NETMASK=255.255.255.0
ONBOOT=yes
#CONNECTED_MODE=yes
```
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
Test the bandwidth and latency between a pair of nodes using MVAPICH2 and the OSU micro-benchmarks. These are provided pre-compiled with the CornelisOPX software bundle.
```
source /usr/mpi/gcc/mvapich2-*-hfi/bin/mpivars.sh
cd /usr/mpi/gcc/mvapich2-*-hfi/tests/osu-micro*/mpi/pt2pt
mpirun -hosts node01,node02 ./osu_latency
mpirun -hosts node01,node02 ./osu_bw
```
Test the bandwidth and latency for all nodes using MVAPICH2 and deviation. The deviation program is provided pre-compiled with the CornelisOPX software bundle.
```
source /usr/mpi/gcc/mvapich2-*-hfi/bin/mpivars.sh
cd /usr/mpi/gcc/mvapich2-*-hfi/tests/intel
pdsh -w node[01-16] -N -R exec echo %h | sort > /tmp/mpi_hosts
mpirun -hostfile /tmp/mpi_hosts ./deviation
```
Alternative, using OpenMPI
```
source /usr/mpi/gcc/openmpi-*-hfi/bin/mpivars.sh
cd $MPI_ROOT/tests/osu-*/mpi/pt2pt
mpirun --allow-run-as-root --host node01,node02 ./osu_latency
mpirun --allow-run-as-root --host node01,node02 ./osu_bw
pdsh -w node[01-16] -N -R exec echo %h | sort > /tmp/mpi_hosts
cd $MPI_ROOT/tests/intel
mpirun --allow-run-as-root --npernode 1 --hostfile /tmp/mpi_hosts ./deviation
```

### Systems with multiple HFIs
Application traffic can be assigned or spread across multiple adapters according to rules set by the environment variable ```PSM2_MULTIRAIL```.
This is a complex topic fully described in the Omni-Path Multirail Application Note (link to be provided).
For simple testing, it is easiest to measure the performance of one HFI at a time. You can direct PSM2 to use a specifc HFI with the ```HFI_UNIT``` environment variable. Here is the syntax for specifying the environment variable in the mpirun command. ```HFI_UNIT``` numbers from 0 to 3.
```
# MVAPICH2
mpirun -hosts node01,node02 -genv HFI_UNIT 0 ./osu_bw
# OpenMPI
mpirun --allow-run-as-root --host node01,node02 -x HFI_UNIT=0 ./osu_bw
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


