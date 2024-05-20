# Setting up an Omni-Path fabric for evaluation with Ubuntu<br>

This procedure is suitable for the installation of small clusters and evaluation projects. It would need to be adjusted for use in a production environment using a Cluster Management system.
<br>

## Prerequisites
- A cluster of two or more Linux servers running Ubuntu 22.04 LTS or 24.04 LTS.
- Passwordless ssh from the headnode to all nodes.
- Omni-Path adapters installed in each node, and cabled to an Omni-Path switch.
  - If using a pair of nodes, these can be cabled back-to-back, without a switch.
- Optional/Recommended:
  - There is a non-root user with a shared home directory.
  - Install the following packages ```sudo apt install -y build-essential wget pdsh```.

**Omni-Path Switches:** In general, Omni-Path switches can be used in their out-of-box state. Some configuration and firmware updates should be done for a production environment, but are usually unecessary for small evaluation systems. Managing switches will be covered in a separate document.

> ## Unknowns
> - Replicate configurations that ./INSTALL applies, but I don't have a list of what they are. Probably best not to do this automatically (as ./INSTALL does); simply describe the steps required to implement them. They inlcude:
>   - Optionally allow non-root users to run opa admin commands. Should be handled by export OPA_UDEV_RULES=1 below.
>   - Add memlock lines to /etc/security/limits.conf
>     - Description in limits.conf is *User space Infiniband verbs require memlock permissions. If desired you can limit these permissions to the users permitted to use OPA and/or reduce the limits.  Keep in mind this limit is per user (not per process)*.
>     - When do you need user space verbs? Maybe for Lustre/BeeGFS/GPFS?
>   - Reserve some contexts (16?) for I/O - I don't know how to do that, or when that is needed.
>   - Anything else?

## Install the host stack
On each node, install the Omni-Path host stack:<br>
```
export OPA_UDEV_RULES=1 # Allows non-root users to execute opa admin commands
sudo apt install -y opa-basic-tools libpsm2-2 opa-fastfabric opa-fm
```
At this point, you may want to setup an IP-over-Fabric interface (also known as IPoIB). This is usual, but optional, and not required for the following MPI tests.
Do this in the same way as for any other network interface by editing a ```netplan``` file, such as ```/etc/netplan/99_config.yaml```, typically using a static IP address.<br>
*Do I need to run ```sudo netplan apply```, or will the reboot take care of that?*
```
# /etc/netplan/99_config.yaml or similar file
network:
  version: 2
  ethernets:
    ib0:
      addresses:
        - 192.168.101.1/24
```
```
reboot
```
Check that the Omni-Path adapters are named in the the traditional way (such as hfi1_0), and not the *predictable* way (such as opap129s). The names can be seen by running ```ls /sys/class/infiniband```.
To configure the traditional name, edit or create the file ```/lib/udev/rules.d/60-rdma-persistent-naming.rules```, and add the line ```ACTION=="add", SUBSYSTEM=="infiniband", PROGRAM="rdma_rename %k NAME_KERNEL"```.
Then ```reboot```, or maybe reload the module with ```modprobe -r hfi1; modprobe hfi1``` (*needs testing*).

On the headnode, start the Fabric Manager. This assumes there is no SM/FM running on the switch.
```
sudo systemctl enable opafm
sudo systemctl start opafm
```
Check that the fabric is up and that the correct number of hosts and switches are present.
```
opafabricinfo
```

## Measure the MPI latency and bandwidth
Build and run OpenMPI and the OSU microbenchmarks in your home directory. This can be done as root, but is best done as a normal user. On this system, the home directory is ```/home/cornelis``` and is shared across all the nodes. If this is not shared on your system, then you can copy the required files to the other systems.

First, download and build OpenMPI.
```
sudo apt install -y libpsm2-devel
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
If the full set of OpenMPI options is required, use this command line:
```
mpirun --mca btl self,vader --mca mtl psm2 --mca pml cm -np 2 --npernode 1 --host node01,node02 ./osu_latency
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
```sudo lspci -d :24f0 -vv | grep LnkSta:``` Check the PCIe connection status. Should be Speed 8GT/s, Width x16.<br>
```sudo dmidecode | grep -A3 "BIOS Info"``` Check the BIOS version.<br>
```apt list --installed | grep -E 'opa|psm2'``` List packages installed and their versions.<br>
```cat /sys/class/infiniband/hfi1_0/nfreectxts``` Check that each running PSM2 MPI rank consumes a context.<br>
```cat /proc/cpuinfo | grep MHz``` Check the frequency of the CPUs.<br>
```mpirun -hosts node01,node02 hostname``` Test that passwordless ssh and the MPI infrastructure are working.<br>

Fabric-based commands

```opafabricinfo``` Fabric summary<br>
```opaextractlids -q -F nodetype:FI``` List the hosts on the fabric.<br>
```opaextractlids -q -F nodetype:SW``` List the switches on the fabric.<br>
```opaextractsellinks -q``` List the links in the fabric.<br>
```opareport -o none -C; sleep 60; opareport -o slowlinks -o errors``` Show the quality of the links in the fabric.<br>
