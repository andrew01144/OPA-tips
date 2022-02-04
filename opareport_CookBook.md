# Omni-Path opareport Cook Book

```opareport``` is a very powerful command for gathering fabric-wide information.
This document shows some examples of how ```opareport``` can be used on the command line and within scripts.

The ```opaextract*``` scripts (e.g. ```opaextractlids```, ```opaextractperf```) are provided to extract particular data from the fabric.
These scripts call ```opareport``` in a form that produces xml output from which certain tags (or data fields) are extracted.
They are often more convenient to use than ```opareport``` because their output is line-oriented and can be sorted and filtered more easily.
You can also use the ```opaextract*``` scripts as the basis for you own customized scripts.

## Exploring the fabric

```opafabricinfo``` Fabric summary<br>
```opaextractlids -q -F nodetype:FI``` List the hosts on the fabric.<br>
```opaextractlids -q -F nodetype:SW``` List the switches on the fabric.<br>
```opaextractsellinks -q``` List the links in the fabric.<br>

## Looking for cabling problems

### Example: Look for links with errors
```
opareport -o none -C; sleep 60; opareport -o slowlinks -o errors
```
This one-liner clears the error counters (```-C```) with no output (```-o none```), then waits 60 seconds to allow for some errors to occur.
It then runs a report that lists any links running slower than they should (```-o slowlinks```) and any links with errors (```-o errors```).
Consider:
- Always remember to include ```-o slowlinks``` because errors on a link may cause the link to be downgraded,
  and the downgraded (slower) link may no longer show any errors.
- Errors are only reported if they exceed the thresholds configured in /etc/opa/opamon.conf.
  If you are accumulating errors over a short time (60 seconds in this example), you may want to lower the thresholds in the file.
  To show every error: change ```Greater``` to ```Equal```, and set the threshold values of the counters you want to see to ```1```.
  Set the value for ```LinkQualityIndicator``` to ```5```.
- Not all errors are bad. Understanding the meaning of the different error types is key to diagnosing problems.
  A full discussion of the different error counters is beyond the scope of this document, but are some brief points:
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
You will see that it runs ```opareport``` with xml output (```-x```), then uses ```opaxmlextract``` to select particular xml tag values to print.
Many custom queries can be achieved by using the various the ```opaextract*``` scripts supplied,
along with ```grep```/```sed```/```cut```/```sort```, or even ```awk``` and ```perl```, for filtering and formatting.

For example, this one-liner shows all links in the fabric with LinkQualityIndicator less than 5, and sorts by quality.
```
opaextractperf | cut -d \; -f 1,2,3,24 | grep -v ';5$' | sort -t \; -k 4n
```

### Example: List all switch ports along with cable details.
This is a script written in the style of ```opaextract*``` that lists all the switch ports in the fabric.
By default, ```opareport``` only shows links and ports that are Active.
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

