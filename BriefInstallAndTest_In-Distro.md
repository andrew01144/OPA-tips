# Setting up an Omni-Path fabric for evaluation<br>(in-distro method)
It is possible to set up an Omni-Path system without downloading and installing any software from Cornelis Networks. Using this method can be more convenient in some situations.

This procedure is suitable for the installation of small clusters and evaluation projects. It would need to be adjusted for use in a production environment using a Cluster Management system.

Cornelis Networks regularly upstreams their code so that it can be realeased with the Linux Distros. This enables you to set up a system using just the packages included in the distro.

| Benefits of using in-distro packages | Benefits of using CornelisOPX software |
| ------------------------------------ | -------------------------------------- |
| The Omni-Path software components have been successfully built on your distro/version of choice. | This configuration has be functionally tested, and is supported by, by Cornelis Networks. |

<br>
<br>

> ## Work in Progress, open questions:
> - Differences between CornelisOPX and in-distro installs.
>   - no hfi1 commands, but they can be installed from rpms in CornelisOPX.
>     - ```rpm -Uvh /tmp/CornelisOPX-OPXS.RHEL*-x86_64.*/repos/OPA_PKGS/RPMS/hfi1-diagtools-sw-0.8-117.x86_64.rpm```
>   - opa admin commands can only be run by root
>   - CornelisOPX: On an idle node, there will be 16 open contexts.
>   - ```memlock``` in ```/etc/security/limits.conf``` is not configured.
>   - AIP - Accelerated IP - available from RHEL 8.4.
>   - Accelerated RDMA - should be available in both CornelisOPX and in-distro.
>   
> - Host stack packages:
>   - is a reboot required between opa-basic-tools and the other packages? Probably not.
>   - is opa-address-resolution useful for anything? Probably not.
> - OpenMPI issues:
>   - OMPI warning: "There was an error initializing an OpenFabrics device".
>     - Fix: Use ```mpirun --mca btl ^openib``` or ```export OMPI_MCA_btl="^openib"```
>     - ```configure --enable-mca-no-build=btl-openib --without-verbs``` does not fix this; ```btl-openib``` still appears in ```ompi_info```.
>   - Is the OpenMPI tree relocatable? Initial look: inconclusive.

<br>
<br>

## Prerequisites
- A cluster of two or more Linux servers running a RHEL-like Linux distro.
- Passwordless ssh from the headnode to all nodes.
- Omni-Path adapters installed in each node, and cabled to an Omni-Path switch.
  - If using a pair of nodes, these can be cabled back-to-back, without a switch.
- Optional/Recommended:
  - There is a non-root user with a shared home directory.
  - pdsh has been installed on the headnode ```yum install pdsh```.

**Omni-Path Switches:** In general, Omni-Path switches can be used in their out-of-box state. Some configuration and firmware updates should be done for a production environment, but are usually unecessary for small evaluation systems. Managing switches will be covered in a separate document.

## Install the host stack
On each node, install the Omni-Path host stack:
```
yum install -y opa-basic-tools rdma-core libpsm2 opa-fastfabric opa-address-resolution opa-fm
```
At this point, you may want to setup the IP-over-Fabric settings (also known as IPoIB). This is usual, but optional, and not required for the following MPI tests. Do this in the same way as for any other network interface by creating a ```/etc/sysconfig/network-scripts/ifcfg-ib0``` file, typically with a static IP address.
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
Build and run OpenMPI and the OSU microbenchmarks in your home directory. This can be done as root, but is best done as a normal user. On this system, the home directory is ```/home/cornelis``` and is shared across all the nodes. If this is not shared on your system, then you can copy the required files to the other systems.

First, download and build OpenMPI.
```
yum install libpsm2-devel
cd /home/cornelis
wget https://download.open-mpi.org/release/open-mpi/v4.1/openmpi-4.1.1.tar.gz
tar xf openmpi-4.1.1.tar.gz
cd openmpi-4.1.1
./configure --prefix=/home/cornelis/openmpi-4.1.1-psm2 --enable-orterun-prefix-by-default --with-psm2
make all install
create /home/cornelis/openmpi-4.1.1-psm2/bin/mpivars.sh:
  export MPI_ROOT=/home/cornelis/openmpi-4.1.1-psm2
  export PATH=$MPI_ROOT/bin:${PATH}
  export LD_LIBRARY_PATH=$MPI_ROOT/lib${LD_LIBRARY_PATH:+:}${LD_LIBRARY_PATH}
  export MANPATH=$MPI_ROOT/share/man:${MANPATH}
  export OMPI_MCA_btl="^openib"
```
*Note: ```--prefix=/home/cornelis/openmpi-4.1.1-psm2``` causes ```make install``` to create this directory and install the OpenMPI components into it.*

Second, download and build the OSU microbenchmarks.
```
cd /home/cornelis
wget https://mvapich.cse.ohio-state.edu/download/mvapich/osu-micro-benchmarks-5.9.tar.gz
tar xf osu-micro-benchmarks-5.9.tar.gz
cd osu-micro-benchmarks-5.9
source /home/cornelis/openmpi-4.1.1-psm2/bin/mpivars.sh
./configure CC=$MPI_ROOT/bin/mpicc CXX=$MPI_ROOT/bin/mpicxx
make
```
If your home directory is not shared, then copy the required directories to the other node(s):
```
cd /home/cornelis
scp -r openmpi-*-psm2 osu-micro-benchmarks-* node02:/home/cornelis
```
Now, measure the bandwidth and latency between a pair of nodes using OpenMPI and the OSU micro-benchmarks.
```
cd /home/cornelis
source /home/cornelis/openmpi-4.1.1-psm2/bin/mpivars.sh
cd osu-micro-benchmarks-5.9/mpi/pt2pt
mpirun --host node01,node02 ./osu_latency
mpirun --host node01,node02 ./osu_bw
```
Measuring bandwidth and latency is a good health check, and particularly recommended during installation.
The program ```deviation.c``` will measure the bandwidth and latency of a list of nodes and report any performance outliers.
```
wget https://github.com/cornelisnetworks/opa-ff/raw/master/MpiApps/apps/deviation/deviation.c
source /home/cornelis/openmpi-4.1.1-psm2/bin/mpivars.sh
mpicc -o deviation deviation.c
pdsh -w node[01-16] -N -R exec echo %h | sort > /tmp/mpi_hosts
mpirun --npernode 1 --hostfile /tmp/mpi_hosts ./deviation
```


### Systems with multiple HFIs
Application traffic can be assigned or spread across multiple adapters according to rules set by the environment variable ```PSM2_MULTIRAIL```.
This is a complex topic fully described in the Omni-Path Multirail Application Note (link to be provided).
For simple testing, it is easiest to measure the performance of one HFI at a time. You can direct PSM2 to use a specifc HFI with the ```HFI_UNIT``` environment variable. Here is the syntax for specifying the environment variable in OpenMPI's mpirun command. ```HFI_UNIT``` numbers from 0 to 3.
```
mpirun --host node01,node02 -x HFI_UNIT=0 ./osu_bw
```

## Diagnostics
Node-based commands

```opainfo``` Check status of the local adapter, link and cable.<br>
```lspci | grep HFI``` Look for Omni-Path adapters.<br>
```lspci -d :24f0 -vv | grep LnkSta:``` Check the PCIe connection status. Should be Speed 8GT/s, Width x16.<br>
```dmidecode | grep -A3 "BIOS Info"``` Check the BIOS version.<br>
```hfi1stats -n 1 | grep Open``` Check for open contexts - each MPI rank that uses PSM2 will open a context.<br>
```cat /proc/cpuinfo | grep MHz``` Check the frequency of the CPUs.<br>
```mpirun -hosts node01,node02 hostname``` Test that passwordless ssh and the MPI infrastructure are working.<br>

Fabric-based commands

```opafabricinfo``` Fabric summary<br>
```opaextractlids -q -F nodetype:FI``` List the hosts on the fabric.<br>
```opaextractlids -q -F nodetype:SW``` List the switches on the fabric.<br>
```opaextractsellinks -q``` List the links in the fabric.<br>
```opareport -o none -C; sleep 60; opareport -o slowlinks -o errors``` Show the quality of the links in the fabric.<br>
