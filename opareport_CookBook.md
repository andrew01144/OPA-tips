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
- Always remember to include ```-o slowlinks``` because a bad cable may cause the link to be downgraded,
  and the downgraded (slower) link may no longer show any errors.
- Think about the error thresholds which are configured in /etc/opa/opamon.conf (XXX more words needed).
- Not all errors are bad. Understanding the meaning of the different error types is key to diagnosing problems.
  A full discussion of the different error counters is beyond the scope of this document, but for some brief examples:
  - The LinkDowned counter shows how many times the link has gone down. A bad cable could cause a link to go down occasionally, which is a problem.
    A rebooting node will also cause the link to go down, which is not a problem.
  - The XXX shows that errors have occured on the link. That may be a bad cable. But, can also occur when the link comes up.
    So, if XXX is present along with LinkDowned, this may not be a problem.


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
or will not stay active for long enough to see them.
This is mainly an issue with ISL cables, because if a host cable is down, then you will be aware of it from other indications.
I decided that a port in a ‘bad state’ is a port that has a cable attached, but is not Active, and/or is generating LinkDowns.
```
# script: opaextractbadports
opareport -o comps -d 5 -A -F portstate:notactive -s -M -x $@ | \
opaxmlextract -d \; -s Focus -e NodeGUID -e PortNum -e NodeDesc \
-e PortState -e PhysState -e OfflineDisabledReason -e LinkDowned \
-s Neighbor -s SMs | \
grep -v ';Offline;No Loc Media;'
```
Explanation: ```opareport -o comps -d 5``` provides the information I need.
By default, ```opareport``` only shows links and ports that are Active; ```-A``` selects **all** ports, both Active and not-Active.
I only want to see the not-Active ports, so I focus on them using: ```-F portstate:NotActive```.
The LinkDowned counter can be interesting, so I add statistics with ```-s```.
By default, ```opareport``` gets statistics (that is port counter data like LinkDowned) from the PM (performance Manager).
However, the PM only holds statistics for Active ports, so I use ```-M``` to tell ```opareport``` to read the counters directly from the device.
Lastly, ```-x``` causes the output to be in xml. ```opaxmlextract``` extracts the data items I need into a one line-per-port format.
```grep -v ';Offline;No Loc Media;'``` removes the ports with no cables. (It’s fine to be not-Active if there is no cable).
I am not interested in host ports that are not-Active, but I don’t need to filter them out, because they can’t been seen from the fabric.
Note: Directly clearing the counters in hardware (```--clearall``` with ```-M```) is slightly disruptive to the PM and should avoided if possible,
but is required for this procedure. Remove reporting LinkDowns from the script if you don’t want to disrupt the PM.

When run, you might see:
```
[root@headnode ~]# opareport -o none -M --clearall
[root@headnode ~]# ./opaextractbadports -Qq
NodeGUID;PortNum;NodeDesc;PortState;PhysState;OfflineDisabledReason;LinkDowned
0x00117501ff536c5f;1;Edge01;Down;Offline;None;284018
0x00117501ff536c5f;2;Edge01;Down;Offline;None;284017
0x00117501ff536c5f;6;Edge01;Down;Polling;;0
0x00117501ff536c5f;8;Edge01;Down;Offline;Transient;6
[root@headnode ~]# 
```

