# Running MPI Applications

To run an application on an HPC network such as Cornelis Networks Omni-Path or Nvidia Infiniband, you will often need to tell the MPI library which network type to use.
This is the subject of this document.

This document uses the term 'network'. In HPC, the high performance network is sometimes called a 'fabric'.

Your application will be compiled to run with a particular MPI library.
Firstly, you need to determine which MPI your application uses because the runtime options for each MPI are different.
Examples of MPIs are: OpenMPI, IntelMPI, MVAPICH2.

Next, you need to tell your MPI library which network to use by specifying the correct options in the ```mpirun``` command. Key words for network type are:
- <b>Cornelis Networks Omni-Path</b>
  - <b>PSM2</b>: This is the original and current interface for the Omni-Path HPC fabric.
  - <b>OFI/OPX</b>: Omni-Path Express (OPX) is a new interface which uses the Open Fabrics Interface (OFI) libfabric library
  from the OpenFabrics Alliance (OFA). It provides higher performance and more capabilities than PSM2 in an industry standard API.
  OPX replaces PSM2. Most current usage of OPX is experimental.
- <b>Nvidia/Mellanox Infiniband</b>
  - <b>OpenIB, Verbs</b>: These are the original, depricated interfaces for Nvidia/Mellanox Infiniband networks.
  - <b>UCX</b>: The current interface for Nvidia/Mellanox Infiniband networks.

Examples of ```mpirun``` options for different MPIs and networks are shown below.

Many applications have their own run scripts. You may need to trace the flow of these scripts in order find the point at which
they execute the ```mpirun``` command, because it is here that the options need to be set.
Some run scripts may simplify this by accepting options like ```-fabric PSM2``` to automatically generate the correct mpirun options.

Be aware that some application documentation is written assuming an Nvidia Infiniband network. In these cases, you will need to *change* the recommended mpirun options. Terms like openib, verbs and ucx refer to InfiniBand and should not be used in an Omni-Path environment.

Lastly, you may need to provide additional tuning options to make you application perform at its best.
This will mainly be MPI options (like pinning ranks to cores), but sometimes network options (like message sizes, etc).

## Command line options for Omni-Path networks.


### Omni-Path/PSM2/OpenMPI
```
mpirun --mca btl self,vader --mca mtl psm2 --mca pml cm \
    -np ${NPROCS} --npernode ${PPN} --host ${HOSTLIST} ${APP_EXE}
```
### Omni-Path/PSM2/IntelMPI
```
mpirun -genv I_MPI_FABRICS=shm:ofi -genv I_MPI_OFI_PROVIDER=psm2 \
    -np ${NPROCS} -ppn ${PPN} -hostlist ${HOSTLIST} ${APP_EXE}
```
		
### Omni-Path/OPX/OpenMPI
```
mpirun --mca btl self,vader --mca mtl ofi --mca pml cm \ <needs verification>
    -x LD_LIBRARY_PATH=${OPX_PATH}:${LD_LIBRARY_PATH} -x FI_PROVIDER=opx \
    -np ${NPROCS} --npernode ${PPN} --host ${HOSTLIST} ${APP_EXE}
```
### Omni-Path/OPX/IntelMPI
```
mpirun -genv I_MPI_FABRICS=shm:ofi -genv I_MPI_OFI_PROVIDER=opx \
    -genv LD_LIBRARY_PATH=${OPX_PATH}:${LD_LIBRARY_PATH} -genv FI_PROVIDER_PATH="" \
    -np ${NPROCS} -ppn ${PPN} -hostlist ${HOSTLIST} ${APP_EXE}
```

## Verifying the configuration
There are a number of ways to verify that you are using the correct network interface.
- <b>Query the number of free contexts</b>:
  Each MPI rank will consume a context on the adapter.
  Run ```cat /sys/class/infiniband/hfi1_0/nfreectxts``` on one or all of the compute nodes to see the number of free contexts.
  Record nfreectxts before running the application, then see if nfreectxts decreases during application execution.
  If nfreectxts does not change, then you have not successfully engaged PSM2.<br>
- <b>Diagnostic Messages</b>:
  For PSM2, ```export PSM2_IDENTIFY=1```. If you are using the PSM2 library, each MPI rank will print the version of the PSM2 library to STDERR.
  You may need to use the MPI's options (like -x or -genv) to set or propagate this environment variable to the compute nodes.<br>
  For OPX, ```FI_LOG_LEVEL=info``` is equivalent.
- <b>Measure the performance</b>:
  Use your MPI library with the options you have selected to run a simple MPI latency test program like osu_latency.
  This should show below 1.5us. If you have the wrong configuration, latency will be much higher.
  Some care should be taken with this because other factors, like the CPU speed governor, may also increase the latency, especially for very short-running programs. 
      
## MPIs, Libraries and Layers - diving deeper

### OpenMPI
OpenMPI is the primary open source MPI library. It may be provided pre-built with your network software stack, or with your application. Or, you may need to download the source and build it yourself. OpenMPI allows you to select the network interface at run-time using command-line options. However, OpenMPI must be built with those capabilities in order to have them available for selection. You may also need to build it yourself to include support for site-specific requirements such as job-schedulers. If you have been provided with a pre-built OpenMPI, it may not have been built with all the capabilities you need.

<b>OpenMPI with PSM2</b>: ```--mca mtl psm2``` tells OpenMPI to use the PSM2 library to access the network.

<b>OpenMPI with Omni-Path Express</b>: Omni-Path Express (OPX) has a different structure from PSM2, and includes an extra layer. To use OPX, you tell OpenMPI to use the Open Fabrics Interfaces (OFI) library ```--mca mtl ofi```. Additionally, you tell the OFI library to use the *OPX provider* to access the network ```-x FI_PROVIDER=opx```.

<b>Building OpenMPI</b>: This examples builds OpenMPI with support for both PSM2 and OPX.
```
./configure CC=gcc --prefix=/path/to/libfabric-1.16.1-installed \
    --enable-psm2=yes --enable-opx=yes --enable-psm3=no --enable-verbs=no \
    --enable-sockets=no --enable-tcp=no --enable-udp=no --enable-efa=no --enable-rxm=no --enable-rxd=no
```

### IntelMPI
IntelMPI is the primary commercially supported MPI library and is bundled with many HPC applications. It normally uses the Open Fabrics Interfaces (OFI) library from the OpenFabrics Alliance, so to select a particular network, you specify the *provider* for the OFI library to use.

<b>IntelMPI with PSM2</b>: ```-genv I_MPI_FABRICS=shm:ofi``` tells IntelMPI to use the OFI library. ```-genv I_MPI_OFI_PROVIDER=psm2``` tells the OFI library to use the psm2 provider to access the network. The PSM2 library is not an OFI provider, so we use a thin library (sometimes called a *shim*) to make the PSM2 library appear as a *provider* to the OFI library.

<b>IntelMPI with OPX</b>: ```-genv I_MPI_FABRICS=shm:ofi``` tells IntelMPI to use the OFI library. ```-genv I_MPI_OFI_PROVIDER=opx``` tells the OFI library to use the opx provider to access the network. The OPX provider is a true OFI provider, so no shims are required in this usage.

### MVAPICH2
The MVAPICH2 library is built for a particular network type and so does not have runtime options to select the network.
There will be different builds for each type of network it supports.
Furthermore, it is typically used by software engineers working on their own HPC codes, and rarely used by distributed versions of HPC applications.
Therefore, we will not discuss MVAPICH2 here.
