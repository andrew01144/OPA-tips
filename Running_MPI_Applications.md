# Running MPI Applications

To run an application on an HPC network such as Cornelis Networks Omni-Path or Nvidia Infiniband, you will need to tell the MPI library which network type to use.
This is the subject of this document.

This document uses the term 'network'. In HPC, the high performance network is sometimes called a 'fabric'.

Your application will be compiled to run with a particular MPI library.
Firstly, you need to determine which MPI your application uses because the runtime options for each MPI are different.
Examples of MPIs are: OpenMPI, IntelMPI, MVAPICH2, PlatformMPI.

Next, you need to tell your MPI library which network to use by specifying the correct options in the ```mpirun``` command.
Examples of ```mpirun``` options for different MPIs and networks are shown below.

Many applications have their own run scripts. You may need to trace the flow of these scripts in order find the point at which
they execute the ```mpirun``` command, because it is here that the options need to be set.
Some run scripts may simplify this by accepting options like ```-fabric PSM2``` to automatically generate the correct mpirun options.

Be aware that some application documentation is written assuming an Nvidia Infiniband network. In these cases, you will need to *change* the recommended mpirun options.

Lastly, you may need to provide additional tuning options to make you application perform at its best.
This will mainly be MPI options (like pinning ranks to cores), but sometimes network options (like message sizes, etc).


### Types of MPI
- <b>OpenMPI</b> and <b>IntelMPI</b>: These are the most commonly used MPI libraries and the focus of this document.
- <b>MVAPICH2</b>: This MPI library is built for a particular network type and so does not have runtime options to select the network.
There will be different builds for each type of network it supports.
Furthermore, it is typically used by software engineers working on their own HPC codes, and rarely used by distributed versions of HPC applications.
Therefore, we will not discuss MVAPICH2 here.
- <b>PlatformMPI</b>: Though distributed with some applications, it is not discussed here. PlatformMPI has previously been known as HP-MPI.
### Types of network and their interfaces
- <b>Cornelis Networks Omni-Path</b>
  - <b>PSM2</b>: This is the original and current interface for the Omni-Path HPC fabric. To run an application on Omni-Path, you will need to configure your MPI to use PSM2.
  - <b>OFI/OPX</b>: Omni-Path Express (OPX) is a new interface which is a provider for the Open Fabrics Interface (OFI) libfabric library
  from the OpenFabrics Alliance (OFA). It provides higher performance and more capabilities than PSM2 in an industry standard API.
  OPX replaces PSM2. Most current usage of OPX is experimental.
- <b>Nvidia Infiniband</b>
  - <b>OpenIB, Verbs</b>: These are the original, depricated interfaces for Nvidia/Mellanox Infiniband networks.
  - <b>UCX</b>: The current interface for Nvidia/Mellanox Infiniband networks.

### Omni-Path/PSM2/OpenMPI
```
mpirun --mca btl self,vader -mca mtl psm2 -mca pml cm \
    -np ${NPROCS} --npernode ${PPN} --host ${HOSTLIST} ${APP_EXE}
```
### Omni-Path/PSM2/IntelMPI
```
mpirun -genv I_MPI_FABRICS=shm:ofi -genv I_MPI_OFI_PROVIDER=psm2 \  # should these be -env to be consitent with the OPX example?
    -np ${NPROCS} -ppn ${PPN} -hostlist ${HOSTLIST} ${APP_EXE}
```
		
### Omni-Path/OPX/OpenMPI
```
mpirun --mca btl self,vader -mca mtl ofi -mca pml cm \ ???????
    -x LD_LIBRARY_PATH=${OPX_PATH}:${LD_LIBRARY_PATH} -x FI_PROVIDER=opx \
    -np ${NPROCS} --npernode ${PPN} --host ${HOSTLIST} ${APP_EXE}
```
### Omni-Path/OPX/IntelMPI
```
mpirun -env I_MPI_FABRICS=shm:ofi -env I_MPI_OFI_PROVIDER=opx \
    -genv LD_LIBRARY_PATH=${OPX_PATH}:${LD_LIBRARY_PATH} -genv FI_PROVIDER_PATH="" \
    -np ${NPROCS} -ppn ${PPN} -hostlist ${HOSTLIST} ${APP_EXE}
```

## Verifying the configuration
There are a number of ways to verify that you are using the correct network interface.
- <b>Query the number of free contexts</b>:
  Each PSM2 MPI rank will consume a context on the adapter.
  Run ```cat /sys/class/infiniband/hfi1_0/nfreectxts``` on one or all of the compute nodes to see the number of free contexts.
  Record nfreectxts before running the application, then see if nfreectxts decreases during application execution.
  If nfreectxts does not change, then you have not successfully engaged PSM2.<br>
  For OPX - same behavior?
- <b>Diagnostic Messages</b>:
  For PSM2, ```export PSM2_IDENTIFY=1```. If you are using the PSM2 library, each MPI rank will print the version of the PSM2 library to STDERR.
  You may need to use the MPI's options (like -x or -genv) to set or propagate this environment variable to the compute nodes.<br>
  For OPX - is there an equivalent?
- <b>Measure the performance</b>:
  Use your MPI library with the options you have selected to run a simple MPI latency test program like osu_latency.
  This should show around 1.5us. If you have the wrong configuration, latency will be much higher.
  Some care should be taken with this because other factors, like the CPU speed governor, may also increase the latency, especially for very short-running programs. 
      
