# OPX Provider Installation and Setup

Libraries like MPI must use the PSM2 or OPX interfaces to access the Cornelis Omni-Path fabric. See also [Running MPI programs](Running_MPI_Applications.md).

Omni-Path Express (OPX) is the new interface which uses the Open Fabrics Interface (OFI) libfabric library from the OpenFabrics Alliance (OFA). It is foundational to all future Omni-Path development, providing higher performance and more capabilities than PSM2 in an industry standard API. If you want to use OPX, read on.

If you have chosen to install Omni-Path support using the *in-distro* opa packages, you will need to download libfabric and the OPX provider from https://www.github.com/ofiwg/libfabric or https://www.github.com/cornelisnetworks/libfabric *(explained in the application note, below)*.

If you have chosen to install Omni-Path support using the *CornelisOPXS software bundle*: From Release 10.12.0, libfabric and the OPX provider are included in the OPXS Software bundle.

Now follow the *OPX Provider Installation and Setup Application Note*.

The Cornelis Networks website does not provide links to software. Instead, follow this procedure:
- Go https://www.cornelisnetworks.com, hover over 'Support', and select 'Customer Center'.
  - You may need to create an account.
- Click on *Release Library*.
- In the menus, select
  - Release: *Latest Release*
  - Products: *blank*
  - Operating System: *blank*
  - Search box: *OPX*
- Now select "*Cornelis Omni-Path Express OPX Provider Installation and Setup Application Note*" and click the download button.<br>
The file will be: Omni-Path_Express_OPX_Provider_Install_&_Setup_AN_A00033_v5.0.pdf.<br>
