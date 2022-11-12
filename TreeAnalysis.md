
# 2-tier fat-tree topology check

Omni-Path switches are connected together in a *fat-tree* topology, also known as a Clos topology.
It's important that the tree is built correctly, because errors in the topology can cause uneven bandwidth within the fabric.
This can be a silent problem, with jobs occasionaly taking longer than at other times, depending on placement and other jobs in the fabric.

Therefore, it is important to check that your topology is correct. In an ideal world, you would be confident that the original cable map
(often an Excel file used to print the labels for the cables) has been designed correctly,
then use a tool such as ```opareport -o verifylinks -T topologyfile.xml``` to check that the cables have all been installed according to the original design.

However, sometimes that orginal design file is not available, or is not trusted, so we have to check the topology without reference to any other information.
This document describes some methods for doing this.

Rules of a fat-tree:
- A 2-tier fat-tree has two tiers of switches, often referred to as Spines and Leafs.
- All the hosts are connected to Leaf switches
- Each Spine switch connects to all of the Leaf switches using the same number of cables to each Leaf.
- The cables connecting the Leafs to the Spines are called ISLs - Inter-Switch Links.
- Leaf switches are not connected to each other, and Spine switches are not connected to each other.
- (Some minor deviations from these rules can sometimes be made, but will not be discussed here.)

By looking at the number of Spines connected to each Leaf, and the number of Leafs connected to each Spine,
you can usually spot any missing or misconnected cables the fabric.
For a more thorough check, list the individual ISLs from each Leaf and ensure they are connected to the Spines in a consistent pattern.

Large fabrics (>1000 nodes) may use a 3-tier fat-tree. Similar techniques can be used for 3-tier trees, but are not described here.

## Get a list of links from the fabric
To start, get a list of links from the fabric using ```opaextractsellinks``` and save it to a file. Each line of the file represents a cable in the fabric.
However, links from ```opaextractsellinks``` can appear either way around (spine-leaf or leaf-spine) which makes grep'ing difficult,
so make a file with each link shown both ways around. For each line, print it, then print it again with the ends reversed.
```
opaextractsellinks -Qq > allLinks.txt
cat allLinks.txt | perl -F\; -lane 'print $_; print "$F[4];$F[5];$F[6];$F[7];$F[0];$F[1];$F[2];$F[3]"' > allLinksX2.txt
```
You should see something like this:
```
0x00117501020e5bb6;25;SW;cm1-Spine15;0x00117501020e5b4a;42;SW;cm1-Leaf24
0x00117501020e5b4a;42;SW;cm1-Leaf24;0x00117501020e5bb6;25;SW;cm1-Spine15
0x00117501020e5bb9;6;SW;cm1-Leaf21;0x00117501020e5b50;22;SW;cm1-Spine01
0x00117501020e5b50;22;SW;cm1-Spine01;0x00117501020e5bb9;6;SW;cm1-Leaf21
0x00117501020e5db7;32;SW;cm1-Leaf05;0x0011750901fe94af;1;FI;cn0226 hfi1_0
0x0011750901fe94af;1;FI;cn0226 hfi1_0;0x00117501020e5db7;32;SW;cm1-Leaf05
0x00117501020e5db7;33;SW;cm1-Leaf05;0x0011750901de5663;1;FI;cn0279 hfi1_0
0x0011750901de5663;1;FI;cn0279 hfi1_0;0x00117501020e5db7;33;SW;cm1-Leaf05
```
## Where switches have been named 'leaf' and 'spine'.
During installation, the NodeDesc of the switches may have been configured to show the role of the switch.
For example: ```Leaf01```, ```Leaf02```, ```Spine01```, ```Spine02```.
When this has been done, the links can be grep'ed and sorted quite easily.
Names maybe different in your fabric, perhaps Leaf/Spine, or Edge/Core; adjust the commands below to match your environment.
If the switches have not been named, skip to the section below.

### Switches and FIs
```
# List/count Leafs
cat allLinksX2.txt | grep 'Leaf.*Spine' | cut -d \; -f 4 | sort -u
# List/count Spines
cat allLinksX2.txt | grep 'Leaf.*Spine' | cut -d \; -f 8 | sort -u
# List/count FIs
cat allLinksX2.txt | grep 'Leaf.*FI'    | cut -d \; -f 8 | sort
# These figures should agree with opafabricinfo.
```
### Links/Cables
```
# Look at the ISL cables:
# Count the ISLs on each leaf. The final sort is optional and highlights any variance in the number.
cat allLinksX2.txt | grep 'Leaf.*Spine' | cut -d \; -f 4 | sort | uniq -c | sort
# Count the ISLs on each spine
cat allLinksX2.txt | grep 'Leaf.*Spine' | cut -d \; -f 8 | sort | uniq -c | sort

# Show the pattern of port usage for the ISLs. (Print ISLs, sorted by leaf, then by leaf-port)
cat allLinksX2.txt | grep 'Leaf.*Spine' | sort -t \; -k4,4 -k2,2n
```
```
# Look at FI cables:
# Count the FIs on each leaf
cat allLinksX2.txt | grep 'Leaf.*FI' | cut -d \; -f 4 | sort | uniq -c
# Show the pattern of port usage for the FIs. (Print FIs, sorted by leaf, then by leaf-port)
cat allLinksX2.txt | grep 'Leaf.*FI' | sort -t \; -k4,4 -k2,2n
# ... and again, but sorted by leaf, then by hostname.
cat allLinksX2.txt | grep 'Leaf.*FI' | sort -t \; -k4,4 -k8
```

## Where switches have *not* been named.
Where switches have *not* been named, you will need to work out which switches are Leafs and which are Spines using one or both of the methods below.
The objective is to make two lists: ```guidsLeafs.txt``` and ```guidsSpines.txt``` (though guidsSpines.txt is not used in this procedure).

### Switches and FIs
```
# Method #1: Sort switches by the number of ISLs they have.
# You can probably spot the leafs and spines from this. Spines will generally have many more ISLs than Leafs have.
cat allLinksX2.txt.txt | grep 'SW.*SW' | cut -d \; -f 1 | sort | uniq -c | sort
# Using Alt-LeftMouse on putty, you can select the column of GUIDs and paste them to a file (or use an editor).
cat > guidsLeafs.txt  [paste, ^D]
cat > guidsSpines.txt [paste, ^D]

# Method #2: Find the leafs by listing the switches that have FIs attached.
# Leafs will have many FIs connected (if your servers are up), and Spines will generally have no FIs connected.
# Count the FIs on each switch
cat allLinksX2.txt | grep 'SW.*FI' | cut -d \; -f 1 | sort | uniq -c
# Does this look correct? If so, make a list of the GUIDs.
cat allLinksX2.txt | grep 'SW.*FI' | cut -d \; -f 1 | sort -u > guidsLeafs.txt
# Make a list of all of the switches
cat allLinksX2.txt | grep -v FI | cut -d \; -f 1 | sort -u > guidsAll.txt
# Remove the leafs, leaving the spines.
cat guidsAll.txt | grep -vFf guidsLeafs.txt > guidsSpines.txt
```
### Links/Cables
```
# Show the pattern of port usage for the ISLs. (Print ISLs, sorted by leaf, then by leaf-port)
for leaf in $(cat guidsLeafs.txt); do cat allLinksX2.txt | grep "$leaf.*SW.*.SW" | sort -t \; -k 2,2n; done
# Show the pattern of port usage for the FIs. (Print FIs, sorted by leaf, then by leaf-port)
for leaf in $(cat guidsLeafs.txt); do cat allLinksX2.txt | grep "$leaf.*SW.*.FI" | sort -t \; -k 2,2n; done
# ... and again, but sorted by leaf, then by hostname.
for leaf in $(cat guidsLeafs.txt); do cat allLinksX2.txt | grep "$leaf.*SW.*.FI" | sort -t \; -k 8; done
```
