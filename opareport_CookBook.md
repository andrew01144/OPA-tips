# Exploring the fabric

An Omni-Path fabric is made up of three types of component:
- HFIs (Host Fabric Interfaces, or Adapters), typically installed in Linux hosts.
- Switches
- Links. These are normally cables, and can be:
  - HFI Links; cables from a switch to a host.
  - ISLs (Inter-Switch Links); cables between switches. Or, in the case of a director switch, the internal links between Leafs and Spines.

This document describes some of the commands that can be used to determine how the components in a fabric are interconnected.

The ```opaextract*``` scripts (```opaextractlids```, ```opaextractperf```, etc.) installed with the Omni-Path host software are very useful for exploring the fabric.
These scripts call ```opareport``` in a form that produces xml output from which certain tags (or data fields) are extracted.
They are often more convenient to use than calling ```opareport``` directly because their output is line-oriented and can be filtered and sorted more easily.
You can also use the ```opaextract*``` scripts as the basis for you own customized scripts.

---

### Basic commands

```opafabricinfo``` Fabric summary. A count of the HFIs, Switches and Links in the fabric.<br>
```opaextractlids -q -F nodetype:FI``` List the hosts in the fabric.<br>
```opaextractlids -q -F nodetype:SW``` List the switches in the fabric.<br>
```opaextractsellinks -q``` List the links in the fabric.<br>

The basic commands can be filtered to answer typical questions:

```opaextractsellinks -q | grep cn015``` What switch and port is host cn015 connected to?<br>
```opaextractsellinks -q | grep 'SW.*SW'``` Show all the ISLs (Inter-Switch Links).<br>
```opaextractsellinks -q | grep -v 'SW.*SW' | grep Edge01``` What hosts are connected to Edge01? (All the links except ISLs).<br>

---
### Example: Count the ISLs on each edge switch

This is a very simple (and only partial) topology check of a fat-tree. All the Edge switches should have 24 uplinks to the Spine switches. In this example, I extract all the links from the fabric ```opaextractsellinks -q```, select the Switch-to-Switch links ```grep 'SW.*SW.'```, then select each edge switch in turn ```grep $sw```, and count the links ```wc -l```. The example shows that there are probably two missing ISLs on em-edge03.
```
[root@em101 ~]# for sw in em-edge01 em-edge02 em-edge03 em-edge04; do
> opaextractsellinks -q | grep 'SW.*SW' | grep $sw | wc -l
> done
24
24
22
24
[root@em101 ~]#
```

---
### Example: List all switch ports along with cable details.
This is a script written in the style of ```opaextract*``` that lists all the switch ports in the fabric.

By default, ```opareport``` only shows links and ports that are Active.<br>
However, I want to see all of the ports, both Active and not-Active, so I use ```-A```.
```
# script: opaextractswitchports
opareport -o comps -d 4 -A -F nodetype:SW -x $@ | \
opaxmlextract -H -d \; -s Focus -e NodeGUID -e PortNum -e NodeType \
-e NodeDesc -e PortState -e PhysState -e OfflineDisabledReason \
-e OM4Length -e VendorName -e VendorPN -e VendorSN \
-s Neighbor -s SMs
```
Output
```
[root@headnode ~]# ./opaextractswitchports -Qq
0x00117501020c4d23;0;SW;Edge02;Active;LinkUp;;;;;
0x00117501020c4d23;1;SW;Edge02;Active;LinkUp;;1m;FCI Electronics;10131941-2010LF;CN1539FA102L0145
0x00117501020c4d23;2;SW;Edge02;Active;LinkUp;;1m;FCI Electronics;10131941-2010LF;CN1539FA102L0026
0x00117501020c4d23;3;SW;Edge02;Active;LinkUp;;1m;FCI Electronics;10131941-2010LF;CN1539FA102L0095
0x00117501020c4d23;4;SW;Edge02;Down;Polling;;3m;FCI Electronics;10142057-4030LF;CN2049ZE304L996C
0x00117501020c4d23;5;SW;Edge02;Down;Offline;No Loc Media;;;;
0x00117501020c4d23;6;SW;Edge02;Down;Offline;No Loc Media;;;;
```

---
### Example: Bounce the link to em102

If I have a problem with a link, I might bounce it to determine if the problem is a one-off or if it is persistent. In the case of a link to a host, I might simply log in to the host and run a command to bounce the port. However, if I want to bounce an ISL, then I need to bounce the port at one end or the other, both of which are switch ports.

So, the example below shows both the general case (of bouncing a switch port), and the specific case (of bouncing a host port from the host's command line). I don't log into the switch and use the switch's CLI because that is only available on managed switches. This method can be use with both managed and unmanaged switches.

This method can also be used to take bad links out of service, by using ```opaportconfig disable```.

```
# Which switch and port is em102 connected to?
[root@em101 ~]# opaextractsellinks -q | grep em102
0x0011750101747a3c;1;FI;em102 hfi1_0;0x00117501020d8282;16;SW;em-edge01     # switch em-edge01, port 16

# What is the switch's LID? I will need this for the opaportconfig command.
[root@em101 ~]# opaextractlids -q -F nodetype:SW | grep em-edge01
0x00117501020d8282;0;SW;em-edge01;0x0002                                    # LID = 0x02

# How many LinkDowneds are currently recorded for em102's switch port?
[root@em101 ~]# opaextracterror -Qq -F nodepat:em-edge01:port:16 | cut -d \; -f 1,3,14
NodeDesc;PortNum;LinkDowned
em-edge01;16;9                                                               # LinkDowned = 9

# Bounce em102's switch port
[root@em101 ~]# opaportconfig -l 0x02 -m 16 bounce
Bouncing Port at LID 0x00000002 Port 16 via local port 1 (0x00117501017a99ec) on HFI 1 (hfi1_0)

# Wait 15 seconds, then query LinkDowned again:
[root@em101 ~]# opaextracterror -Qq -F nodepat:em-edge01:port:16 | cut -d \; -f 1,3,14
NodeDesc;PortNum;LinkDowned
em-edge01;16;10                                                              # LinkDowned = 10
[root@em101 ~]#

# For comparison, bounce the link using a local command on the host
[root@em101 ~]# ssh em102 opaportconfig bounce
Bouncing Port 1 (0x0011750101747a3c) on HFI 1 (hfi1_0)

# Wait 15 seconds, then query LinkDowned again:
[root@em101 ~]# opaextracterror -Qq -F nodepat:em-edge01:port:16 | cut -d \; -f 1,3,14
NodeDesc;PortNum;LinkDowned
em-edge01;16;11                                                              # LinkDowned = 11
[root@em101 ~]#

# Now, clear all the error counters across the whole fabric:
[root@em101 ~]# opareport -o none -C
<detail removed>
[root@em101 ~]# opaextracterror -Qq -F nodepat:em-edge01:port:16 | cut -d \; -f 1,3,14
NodeDesc;PortNum;LinkDowned
em-edge01;16;0                                                               # LinkDowned = 0
[root@em101 ~]#

```

---
---
---



## Looking for cabling problems

### Example: Look for links with errors
```
opareport -o none -C; sleep 60; opareport -o slowlinks -o errors
```
This one-liner clears the error counters ```-C``` with no output ```-o none```, then waits 60 seconds to allow for some errors to occur.
It then runs a report that lists any links running slower than they should ```-o slowlinks``` and any links with errors ```-o errors```.
Consider:
- Always remember to include ```-o slowlinks``` because errors on a link may cause the link to be downgraded,
  and the downgraded (slower) link may no longer show any errors.
- Errors are only reported if they exceed the thresholds configured in /etc/opa/opamon.conf.
  If you are accumulating errors over a short time (60 seconds in this example), you may want to lower the thresholds in the file.
  To show every error: change ```Greater``` to ```Equal```, and set the threshold values of the counters you want to see to ```1```.
  Set the value for ```LinkQualityIndicator``` to ```5```.
- Not all errors are bad. Understanding the meaning of the different error types and the context in which they have occurred is key to diagnosing problems.
  A full discussion of the different error counters is beyond the scope of this document, but here are some brief points:
  - The ```LinkDowned``` counter shows how many times the link has gone down. A bad cable could cause a link to go down occasionally, which is a problem;
    but a rebooting node will also cause the link to go down, and that is not a problem.
  - ```LocalLinkIntegrityErrors``` counts each error that has occured on the link. A high number may indicate a bad cable.
    However, errors can also occur when the link comes up.
    So, if ```LocalLinkIntegrityErrors``` are present along with ```LinkDowned```, this may not be a problem.
    Also, note that the default ```opamon.conf``` suppresses reporting ```LocalLinkIntegrityErrors```. 


### Example: Report the LinkQualityIndicator of every active port in the fabric.
```
opaextractperf | cut -d \; -f 1,2,3,24
```
Look inside the ```opaextractperf``` script.
You will see that it runs ```opareport``` with xml output ```-x```, then uses ```opaxmlextract``` to select particular xml tag values to print.
Many custom queries can be achieved by using the various the ```opaextract*``` scripts supplied,
along with ```grep```/```sed```/```cut```/```sort```, or even ```awk``` and ```perl```, for filtering and formatting.

For example, this one-liner shows all links in the fabric with LinkQualityIndicator less than 5, and sorts by quality.
```
opaextractperf | cut -d \; -f 1,2,3,24 | grep -v ';5$' | sort -t \; -k 4n
```


### Example: List all switch ports that appear to be in a bad state.
While ```opareport -o errors -o slowlinks``` will report any bad links/cables that are active, there may also be links that are so bad that they have not become active,
or will not stay active for long enough to report them.
An example of a port in a bad state is a port that is not-Active but does have a cable attached.
```
# script: opaextractbadports
opareport -o comps -d 4 -A -F portstate:notactive -x $@ | \
opaxmlextract -d \; -s Focus -e NodeGUID -e PortNum -e NodeDesc \
-e PortState -e PhysState -e OfflineDisabledReason \
-e OM4Length -e VendorName \
-s Neighbor -s SMs | \
grep -Ev ';Offline;(No Loc Media|Not installed|Disconnected);'
```
Explanation: ```opareport -o comps -d 4``` provides the information I need.
By default, ```opareport``` only shows links and ports that are Active; ```-A``` selects **all** ports, both Active and not-Active.
I only want to see the not-Active ports, so I focus on them using: ```-F portstate:notactive```.
```-x``` causes the output to be in xml, and ```opaxmlextract``` extracts the data items I want into a one-line-per-port format.

Not all not-Active ports are not bad, so I use ```grep -v``` to remove the ports with good reasons to be not-Active. These are ports with no cable (No Loc Media), and unused internal ports in director switches (Not installed and Disconnected).

When run, you might see:
```
[root@headnode ~]# ./opaextractbadports -Qq
NodeGUID;PortNum;NodeDesc;PortState;PhysState;OfflineDisabledReason;OM4Length;VendorName
0x00117501020d8419;11;em-edge02;Down;Polling;;2m;Hitachi Metals
0x00117501020d8419;12;em-edge02;Down;Polling;;2m;Hitachi Metals
0x00117501020d8419;47;em-edge02;Down;Offline;None;3m;FINISAR CORP
0x00117501020d841b;12;em-edge04;Down;Polling;;2m;Hitachi Metals
0x00117501020d841b;33;em-edge04;Down;Polling;;3m;FINISAR CORP
0x00117501020d841b;34;em-edge04;Down;Training;;3m;FINISAR CORP
0x00117501020d841b;48;em-edge04;Down;Training;;3m;FINISAR CORP
[root@headnode ~]# 
```
As a rough guide:
- A port with an unterminated cable will appear as ```Down;Polling```.
- A port in an unstable state will cycle though ```Down;Offline```, ```Down;Polling```, and ```Down;Training```.

