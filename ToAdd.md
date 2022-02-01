# To be added
Omni-Path tips to be added to existing or new docs in this project.

- How to download the software: Go to www.cornelisnetworks.com, hover over 'Support', and select 'Customer Center'. You may need to create an account. Go to the 'Release Library' and select 'Release: Latest Release', 'Product: Omni-Path Software (Including Omni-Path Host Fabric Interface Driver', 'Operating System: RHEL 8.3'. Now select "Cornelis Omni-Path IFS Software - RHEL 8.3 - Release 10.11.0.0" and it will download the file: IntelOPA-IFS.RHEL83-x86_64.10.11.0.0.577.tgz.
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
- Mention error count thresholds for opareport -o errors
- How to handle multiple HFIs, HFI_UNIT=0, hwloc-ls (yum install hwloc)
- Mention using taskset
- Mention see core usage: ps -A -o psr,comm | grep <app name> | sort -n

