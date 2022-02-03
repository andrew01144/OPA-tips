# Scrapbook
This stuff does not have a home yet.

### Example: List all switch ports that appear to be in a bad state
**WITH LinkDowned COUNTER**

While ```opareport -o errors -o slowlinks``` will report any bad links/cables that are active, there may also be links that are so bad that they have not become active,
or will not stay active for long enough to report them.
An example of a port in a bad state is a port that is not-Active but does have a cable attached.
```
# script: opaextractbadports
opareport -o comps -d 6 -A -F portstate:notactive -s -M -x $@ | \
opaxmlextract -d \; -s Focus -e NodeGUID -e PortNum -e NodeDesc \
-e PortState -e PhysState -e OfflineDisabledReason -e LinkDowned \
-s Neighbor -s SMs | \
grep -Ev ';Offline;(No Loc Media|Not installed|Disconnected);'
```
Explanation: ```opareport -o comps -d 6``` provides the information I need.
By default, ```opareport``` only shows links and ports that are Active; ```-A``` selects **all** ports, both Active and not-Active.
I only want to see the not-Active ports, so I focus on them using: ```-F portstate:NotActive```.
```-x``` causes the output to be in xml, and ```opaxmlextract``` extracts the data items I need into a one line-per-port format.

The LinkDowned counter can indicate a bouncing link, so I add statistics with ```-s```.
By default, ```opareport``` gets statistics (that is port counters like LinkDowned) from the PM (Performance Manager).
However, the PM only provides statistics for Active ports, so I use ```-M``` to tell ```opareport``` to read the counters directly from the device.
Note: Directly clearing the counters in hardware (```--clearall``` with ```-M```) is slightly disruptive to the PM and should avoided if possible,
but is required for this procedure.

Lastly, not all not-Active ports are not bad. I use ```grep -v``` to remove ports with good reasons to be not-Active; these are ports with no cable (No Loc Media) and some states encountered in director switches (Not installed and Disconnected).

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
---

## PortState/PhysState/OfflineDisabledReason examples
```
0x00117501026777de;6;SW;em-core01 L103B;Active;LinkUp;;3m;FINISAR CORP    Connected to host
0x00117501026777de;7;SW;em-core01 L103B;Down;Training;;3m;FINISAR CORP    Connected to cable
0x00117501026777de;8;SW;em-core01 L103B;Down;Polling;;10m;FINISAR CORP    Connected to cable
0x00117501026777de;9;SW;em-core01 L103B;Down;Offline;No Loc Media;;       No cable
0x00117501026777de;17;SW;em-core01 L103B;Down;Offline;Disconnected;;      Ports 17-24 of 16-port Leaf
0x0011750102754934;3;SW;em-core01 S201A;Down;Offline;Not installed;;      Spine port to empty Leaf slot.
                                                            ????          Leaf port to empty Spine slot.
```
```
PortState           PhysState               OfflineDisabledReason

Active              LinkUp                  <null>

Down                Offline                 None                      # initial port state
                                            No Loc Media              # no cable in this port
                                            Not installed             # this Spine port goes to an empty leaf slot
                                            Disconnected              # ports 17-24 on a 16-port leaf
                    Polling                 <null>                    # looking for neighbor
                    Training                <null>                    # brining the link up
        <assumed>   Init                    <null>                    # waiting for SM
                    
                    ??Disabled
		    
A bad port might cycle through:
;Down;Offline;None;
;Down;Polling;;
;Down;Training;;
```







