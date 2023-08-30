# In-distro or Cornelis OPXS software bundle - which to use?

Cornelis Networks regularly upstreams their code so that it can be realeased with the Linux distros. This enables you to set up a system using just the packages included in the distro.

| Using the in-distro packages | Using the Cornelis OPXS software bundle |
| ------------------------------------ | -------------------------------------- |
| Should be considered for RHEL 8.4 and later.<br>May be easier to install and maintain. | Recommended for RHEL earlier than 8.4.<br>The OPXS software bundle contains some features and capabilities that are not available in the pre-8.4 packages. |
| Benefits:<br>The Omni-Path software components have been successfully built on your distro/version of choice. | Benefits:<br>This configuration has been functionally tested, and is supported by, Cornelis Networks. |

<br>
<br>

## Differences between OPXS and in-distro installations
### Features and configuration
- OPXS: On an idle node, there will be 16 open hfi contexts. This can improve verbs performance, particularly for storage operations. I don't know how to replicate this.
- OPXS: ```/etc/security/limits.conf``` contains ```* hard memlock unlimited``` ```* soft memlock unlimited```.
- AIP - Accelerated IP - is available in in-distro from RHEL 8.4.
- Accelerated RDMA - should be available in both OPXS and in-distro.
### Commands
- opa admin commands can only be run by root. To enable non-root users, create a udev rule to ```chmod 666 /dev/infiniband/umad*```.
- The following commands are not included in the in-distro Omni-Path packages.
  - hfi1 commands ```hfi1_control``` ```hfi1_eprom``` ```hfi1_pkt_send``` ```hfi1_pkt_test``` ```hfi1stats```
    -  These can be installed from rpms in OPXS. ```rpm -Uvh /tmp/CornelisOPX-OPXS.RHEL*-x86_64.*/repos/OPA_PKGS/RPMS/hfi1-diagtools-sw-0.8-117.x86_64.rpm```
  - ```opa-arptbl-tuneup``` Adjusts kernel ARP/neighbor table sizes for very large subnets based on configured IPv4/IPv6 network interface netmask values.
Normally executes once on boot by opa.service ; however, opa-arptbl-tuneup can be invoked with user discretion for a changed subnet configuration.
  - ```opaautostartconfig``` Provides a command line interface to configure autostart options for various OPA utilities.
  - ```opaconfig``` (Switch and Host) Configures the Omni-Path Software through command line interface or TUI menus.
### Patching
- Patching in-distro Omni-Path software is accomplished with pulls from GitHub repository and manually applied.
The Cornelis GitHub repository:  https://github.com/cornelisnetworks
### HFI parameters
  - There is only 1 parameter not available with the inbox release: ```ipoib_accel``` This parameter enables or disables the AIP feature.  The same can be accomplished inbox with the parameter ‘cap_mask’;  specifically bit 19 or 0x80000, which is set by default.

<br>
<br>

> ## Work in Progress, open questions:
> - How to pre-open the 16 contexts
