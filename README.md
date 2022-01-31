# OPA-tips
A cheat-sheet for some Cornelis Omni-Path admin commands and procedures.

## Installation and testing
This procedure is suitable for the installation of small clusters and evaluation projects. It would need to be adjusted for use in a production environment using a Cluster Management system.

Prerequisites
- A cluster of two or more Linux servers running a RHEL-like Linux distro.
- Passwordless ssh from the headnode to all nodes.
- Optional/Recommended: pdsh on the headnode ```yum install pdsh```.
 
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
On the headnode, start the Fabric Manager
```
systemctl enable opafm
systemctl start opafm
```
Check that the fabric is up
```
opafabricinfo
```
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

## Useful diagnostic commands
Node-based commands

```opainfo``` Check status of local adapter, link and cable.<br>
```lspci | grep HFI``` Look for Omni-Path adapters.<br>
```lspci -d :24f0 -vvv | grep LnkSta:``` Check PCIe connection status. Should be Speed 8GT/s, Width x16.<br>
```dmidecode | grep -A3 "BIOS Info"``` Check BIOS version (Dell only?).<br>
```hfi1stats -n 1 | grep Open``` Check for open contexts - this indicates an MPI rank using PSM2.<br>
```cat /proc/cpuinfo | grep MHz``` Check frequency of CPUs.<br>
```mpirun -hosts node01,node02 hostname``` Test that passwordless ssh and the MPI infrastructure is working.<br>

Fabric-based commands

```opafabricinfo``` Fabric summary<br>
```opareport -o none -C; sleep 60; opareport -o slowlinks -o errors``` Show quality of links in the fabric.<br>
```opaextractlids -q -F nodetype:FI``` List hosts on the fabric.<br>
```opaextractlids -q -F nodetype:SW``` List switches on the fabric.<br>
```opaextractsellinks -q``` List links on the fabric.<br>






## Other Stuff
OpenMPI procedure for running osu_latency, osu_bw and deviation.
```
source /usr/mpi/gcc/openmpi-*-hfi/bin/mpivars.sh
cd /usr/mpi/gcc/openmpi-*-hfi/tests/osu-micro-benchmarks-*/mpi/pt2pt
mpirun --allow-run-as-root --host node01,node02 ./osu_latency
mpirun --allow-run-as-root --host node01,node02 ./osu_bw
pdsh -w node[01-16] -N -R exec echo %h | sort > /tmp/mpi_hosts
mpirun --allow-run-as-root --npernode 1 --hostfile /tmp/mpi_hosts ./deviation
```
## TBD
- Download the software: From www.cornelisnetworks.com, hover over 'Support', select 'Customer Center'. You may need to create an account. Go to the 'Release Library' and select 'Relase: Latest Release', 'Product: Omni-Path Software (Including Omni-Path Host Fabric Interface Driver', 'Operating System: RHEL 8.3'. Now select "Cornelis Omni-Path IFS Software - RHEL 8.3 - Release 10.11.0.0" and it will download the file: IntelOPA-IFS.RHEL82-x86_64.10.11.0.0.577.tgz.
- Build and run OpenMPI
- Run IMB and IntelMPI
- Setup and test Advanced IP
- Mention that most systems have 16 open contexts when idle
- Mention error count thresholds for opareport -o errors






