# To be added
Omni-Path tips to be added to existing or new docs in this project.

- How to download the software:<br>
Go to www.cornelisnetworks.com, hover over 'Support', and select 'Customer Center'.<br>
You may need to create an account.<br>
Click on 'Release Library'. In the menus, select 'Release: *Latest Release*', 'Products: *blank*', 'Operating System: *RHEL 8.4*'. Note: Go all the way to the bottom of the menu to find it.<br>
Now select "*Cornelis Omni-Path Express OPXS (Formerly IFS) Software - RHEL 8.4 - Release 10.11.1.1*" and click the download button.<br>
The file will be: CornelisOPX-OPXS.RHEL84-x86_64.10.11.1.1.1.tgz.<br>
- How to build and run OpenMPI
- How to run IMB and IntelMPI<br>
```
source /somewhere/intel-cluster-runtime/2019.6/mpi/intel64/bin/mpivars.sh
cd 
mpirun -ppn 32 -np 64 -f mpi_hosts -genv I_MPI_FABRICS=shm:ofi -genv FI_PROVIDER=psm2 \
/somewhere/intel-cluster-runtime/2019.6/mpi/intel64/bin/IMB-MPI1 uniband -npmin 64 -warm_up on -msglog 3:7
```
- How to setup and test Advanced IP: Instructions are in Section 8 and 8.2 of the Performance Tuning Guide
- Mention that most systems have 16 open contexts when idle
- How to use taskset and hwloc-ls (yum install hwloc)
- How to see core usage: ps -A -o psr,comm | grep <app name> | sort -n

