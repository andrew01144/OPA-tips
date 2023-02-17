# Running MPI Applications

To run an application on an HPC network such as Cornelis Omni-Path or Nvidia Infiniband, you will need to tell the MPI library which network type to use.
This is the subject of this document.

This document uses the work 'network'. In HPC, the high performance network is sometimes called a 'fabric'.

Your application will be compiled to run with a particular MPI library.
Firstly, you need to determine which MPI your application uses because the runtime options for each MPI are different.
Examples of MPIs are: OpenMPI, IntelMPI, MVAPICH2, PlatformMPI.

Next, you need to tell your MPI library which network to use by specifying the correct options to the ```mpirun``` command.
Examples of ```mpirun``` options for different MPIs and networks are shown below.
In reality, many applications have their own run scripts. You may need to work out how these scripts work in order find the point at which
they construct the ```mpirun``` command, because it is here that the options need to be set correctly.
Furthermore, some run scripts try to simplify this process: the run script may accept an option like ```-fabric PSM2``` to automatically generate
the correct mpirun options for the selected network.

Lastly, you may need to provide additional tuning options to make you application perform at its best.
This will mainly be MPI options (like pinning cores to ranks), but sometimes network options (like message sizes, etc).


### Types of MPI
- <b>OpenMPI</b> and <b>IntelMPI</b>: These are the most commonly used MPI libraries and the focus of this document.
- <b>MVAPICH2</b>: This MPI library is built for a particular network type and so does not have runtime options to select the network.
There will be different builds for each type of network it supports.
Furthermore, it typically used by software engineers working on their own HPC codes, and rarely used by distributed versions of HPC applications.
Therefore, we will not discuss MVAPICH2 here.
- <b>PlatformMPI</b>: Though distributed with some applications, it is not discussed here. PlatformMPI has previously been known as HP-MPI.
### Types of Network
- <b>PSM2</b>: This is the long-established interface for the Omni-Path HPC fabric. To run an application on Omni-Path, you will need to configure your MPI to use PSM2.
- <b>OPX, OFI</b>: Omni-Path Express (OPX) is a new interface which is a provider for the Open Fabrics Interface (OFI) libfabric library
  from the OpenFabrics Alliance (OFA). It provides higher performance and more capabilities than psm2 in an industry standard API.
  OPX replaces PSM2. Most application usage of OPX is currently experimental.
- <b>OpenIB, Verbs, UCX</b>: These are the interfaces for Nvidia/Mellanox Infiniband networks.

### PSM2/OpenMPI
```
mpirun --mca btl self,vader -mca mtl psm2 -mca pml cm \
    -np ${NPROCS} --npernode ${PPN} --host ${HOSTLIST} ${APP_EXE}
```
### PSM2/IntelMPI
```
mpirun -genv I_MPI_FABRICS=shm:ofi -genv I_MPI_OFI_PROVIDER=psm2 \
    -np ${NPROCS} -ppn ${PPN} -hostlist ${HOSTLIST} ${APP_EXE}
```
		
### OPX/OpenMPI
```
mpirun --mca btl self,vader -mca mtl ofi -mca pml cm \ ???????
    -x LD_LIBRARY_PATH=${OPX_PATH}:${LD_LIBRARY_PATH} -x FI_PROVIDER=opx \
    -np ${NPROCS} --npernode ${PPN} --host ${HOSTLIST} ${APP_EXE}
```
### OPX/IntelMPI
```
mpirun -env I_MPI_FABRICS=shm:ofi -env I_MPI_OFI_PROVIDER=opx \
    -genv LD_LIBRARY_PATH=${OPX_PATH}:${LD_LIBRARY_PATH} -genv FI_PROVIDER_PATH="" \
	  -np ${NPROCS} -ppn ${PPN} -hostlist ${HOSTLIST} ${APP_EXE}
```

## Verifying the configuration
There are a number of ways to verify that you are using the correct network interface.
- <b>Count the Open Contexts</b>:
  Run ```hfi1stats -n 1 | grep Open``` on one or all of the compute nodes while the application is running.
  You should see one open context for each MPI rank. Note that sometimes some contexts are opened at boot time to improve storage performance,
  so you should record the CtxtsOpen before running the application, then see if this number increases when the application is run.
- <b>Diagnostic Messages</b>:
  For PSM2, ```export PSM2_IDENTIFY=1```. If you are using the PSM2 library, each MPI rank will print the version of the PSM2 library to STDERR.
  You may need to use the MPI's options (like -x or -genv) to set or propagate this environment variable to the compute nodes.<br>
  For OPX - is there an equivalent?
- <b>Measure the performance</b>:
  Use your MPI library with the options you have selected to run a simple MPI latency test program like osu_latency.
  This should show around 1.5us. If you have the wrong configuration, latency will be much higher.
  Some care should be taken with this because other factors, like the CPU speed governor, may also increase the latency for very short-running programs. 
      