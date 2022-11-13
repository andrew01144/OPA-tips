# XXX work in progress XXX Configuring managed switches

Also known as "internally managed switches".

Managed switches are managed out-of-band, via ```ssh``` over Ethernet.
Therefore, each switch requires an Ethernet connection and an IP address.
(Director switches typically require two Ethernet connections and three IP addresses, see below.)

## Setting the switch's IP address
- Either: Connect to the switch's default IP address.<br>
  This assumes the switch is in its factory configuration. Connect the switch's Ethernet directly to your laptop using the default username, password and IP address: ```admin@192.168.100.9``` password ```adminpass```.
  You will need to configure your laptop's Ethernet port with a compatible IP address, such as 192.168.100.1, and use an ssh client such as putty.
- Or: Connect to the switch's serial port.<br>
  Take a USB A-A cable, plug one end into the switch, and the other end into your laptop.
  (Windows will ususually install drivers if required.) Then use putty or hyperterm to connect to Serial/COMx at 115,200 baud.

Then use the ```setChassisIpAddr``` command to set the IP address of the switch, as shown below.


Overview:
Use opachassisadmin, ssh, sftp, Chassis Viewer. Setting passwords and keys or disabling them. Batch or individual. Config, fw update, monitor.

If you need to discover the IP addresses of your managed switches, use this command:
```
opagenchassis > switchIPs
```

Once you have an ethernet connection to the switch, you should make the following configurations.
If you have multiple switches, it is good practice to configure each switch with a meaningful NodeDesc.
```
swapBsDel
setChassisIpAddr -h <my ip> -m <mask>
setDefaultRoute -h <router>
setNodeDesc edge01
prompt "edge01-> "
time -S <NTPserver> or -T <HHMMSSmmddyyyy>
timeZoneConf 0                             (UK:0, France:1, California:-8)
timeDSTConf 4 1 3 5 1 10                   (Correct for Europe, USA will be different)

loginMode 1   (password not required, this can be useful during setup)
loginMode 0   (normal setting: username and password required)
```

Update the firmware:
```
opachassisadmin -H 192.168.100.9 -P STL1.q7.10.8.4.0.5.spkg upgrade
opachassisadmin -H 192.168.100.9 reboot
```

Useful query commands:
```
hwCheck
hwMonitor [onepass]
logShow
fwVersion
showInventory
fruInfo
capture
```

---
```
opachassisadmin -H 192.168.100.9 configure
Do you wish to adjust syslog configuration settings? [y]: n
Do you wish to configure an NTP server? [y]: n
Do you wish to configure timezone and DST information? [y]: n
Do you wish to configure the chassis link width? [n]: n
Do you wish to configure OPA Node Desc to match ethernet chassis name? [y]: n
Do you wish to configure the Link CRC Mode? [n]: n
```
---

## Director switches
Most director switches are supplied with two Management Modules (MMs) for redundancy, and this adds some additonal complexity to how they are managed.
First, follow the instructions above for configuring Managed Switches. Next, note the following additional features:
- A director switch has two Ethernet ports, one for each MM, and requires two Ethernet cables.
- A director switch needs three IP addresses: one for each MM, and one that is bound to the MM that is currently master.
  The master IP is setup up using the setChassisIpAddr command. Search the installation manual for ‘moduleip’ to see how to setup the other two IP addresses.
  The MM IP addresses will appear as source addresses for syslog messages.
- It is useful to understand all the reboot modes: ```reboot all|-s|-m [slot #]```, and how they causes failovers.


## Pure CLI methods
If you are unable to use ```opachassisadmin```, here are alternative CLI methods based on ```ssh``` and ```sftp```.
```
# Upgrading firmware:
[root@headnode ~] sftp admin@switch
sftp> put STL1.q7.10.8.4.0.5.spkg A:/firmware/STL1.q7.10.8.4.0.5.spkg
sftp> exit
[root@headnode ~] ssh admin@switch
Edge-> showLastScpRetCode -all (Repeat until it shows success – it takes approximately 1 minute)
Edge-> reboot now all

# Enabling the embedded SM:
[root@headnode ~] ssh admin@switch
Edge->
Edge-> smConfig startAtBoot yes
Edge-> smControl restart

# Modifying the embedded SM configuration file:
[root@headnode ~] sftp admin@switch:/firmware/opafm.xml .
[root@headnode ~] vi opafm.xml
[root@headnode ~] sftp admin@switch
sftp> put opafm.xml /firmware
sftp> exit
```

Note: the ```sftp``` ‘put’ syntax can be tricky.
- Use the syntax exactly as show here before trying other variations.
- These ‘put’ examples use ```sftp``` in interactive mode. ```sftp``` has a batch mode, but this cannot be used with normal password authentication.
  If you need to use batch mode, you can configure the switch to use passwordless authentication:
  - Either, disable required passwords: ```Edge-> loginMode 1```
  - Or, provide an ssh key: ```ssh admin@switch sshKey add \"$(cat ~/.ssh/id_rsa.pub)\"```

Further reading: redirect syslog, multiple VLs, ```factory 0|1```.


