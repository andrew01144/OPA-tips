# Configuring externally managed switches
Externally managed switches are managed in-band, over the fabric. They have no Ethernet or serial connection, no http or ssh access. The switches do have an
Ethernet/USB socket, but this is not active. All management is done from special commands run on a fabric node. This is primarily ```opaswitchadmin```,
and is usually run on the cluster’s headnode.

First, generate a file containing the GUIDs of the switches you want to manage. You may know these, or you can use ```opagenswitches``` to discover them
from the live fabric.

```
opagenswitches > switchGuids
```

Test the file, and the opaswitchadmin command, using the simple ping function.
```
opaswitchadmin -L switchGuids ping
```
Try the following query commands. Note that results end up in ~/test.res, and the reports end with a summary of the differences (if any) between
the attributes and configuration of the switches.
```
opaswitchadmin -L switchGuids info
opaswitchadmin -L switchGuids hwvpd
opaswitchadmin -L switchGuids getconfig
```
It is good practice to configure each switch with a meaningful NodeDesc. Do this by (1) editing the switchGuids file to add the name for each switch,
then (2) run ``opaswitchadm -L switchGuids configure``. This alows you to set various parameters within the switch, and will ask which ones you want to change.
```
vi switchGuids
0x00117501020d8f20:0:0,Edge01,3
0x00117501020d8a42:0:0,Edge02,3
0x00117501020d85b3:0:0,Core01,3
etc

opaswitchadmin -L switchGuids configure
Do you wish to configure the switch Link Width Options? [n]:
Do you wish to configure the switch Node Description as it is set in the switches file? [n]: y
Do you wish to configure the switch FM Enabled option? [n]:
Do you wish to configure the switch Link CRC Mode? [n]:

```
To upgrade the firmware:
```
opaswitchadmin -L switchGuids -P Intel_PRREdge_V1_firmware.10.8.4.0.5.emfw upgrade
# After a firmware upgrade, and after certain configuration changes, you will need to reboot the switches.
opaswitchadmin -L switchGuids reboot
```
Because the reboot command is sent to the switches over the fabric, you can potentially ‘paint yourself into a corner’. If you reboot a switch which is close to you,
then you will be unable to reboot a more distant switch. There are a number of ways to avoid this problem:
1.	The ```opagenswitches``` command puts a ‘distance’ parameter against each switch in the switchGuids file so that ```opaswitchadmin``` can reboot them in a safe order.
3.	Alternatively, you can achieve a similar effect by grouping the switches into separate switchGuids files and rebooting each batch in the correct sequence.
4.	Lastly, you can use the lower level command ```opaswreset -t <guid> -i 30```. This command returns immediately, but the switch waits an interval of 30 seconds
  before rebooting. If you put this command in a loop to reboot all the switches, 30 sec should be enough time to instruct them all before any of them start to
  disrupt the fabric by rebooting. In the following example, we measure how long it takes to ping each switch before issuing the ```opaswreset``` command.
```
export PATH=/usr/lib/opa/tools:$PATH
cat switchGuids | sed -n 's/^\(0x[0-9A-Fa-f]*\).*/\1/p' > guidList
time for guid in $(cat guidList); do opaswping -t $guid; done
for guid in $(cat guidList); do opaswreset -t $guid -i 30; done
```
For building your own scripts to ping, query and reboot switches, look at the ```opaswping```, ```opaswquery```, ```opaswreset``` commands in /usr/lib/opa/tools.
I recommend using ```opaswitchadmin``` for configuration and firmware upgrades.
