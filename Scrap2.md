```



########## GIGABTYE Servers - access BIOS from Linux CLI

With Dell, we would use iDRAC/racadm. This is how to do it on a Gigabyte server.

##### List the BIOS parameter values

Determine the model number
# dmidecode | grep -EA3 '(BIOS|System) Information'

Google the model number "G262-ZR0-00"
On the product page, click Support, Click Utility. You should see:
https://www.gigabyte.com/us/Enterprise/High-Density-Server/H262-Z63-rev-A00#Support-Utility

Get the Download Link for "GSM CLI"
# mkdir AndrewCornelis
# cd AndrewCornelis
# wget https://download.gigabyte.com/FileList/Utility/server_utility_cli_2.1.xx.zip
# unzip server*.zip

The form of the command is:
java -jar GbtUtility-2.1.21.jar -H {bmc IP} -U {bmc account} -P {bmc password} -RU {Redfish account} -RP {Redfish password} redfish bios info

Note: On the system I used, I could not ping my own BMC, but I could ping others.

Run the Util, request bios info
# java -jar 2.1.50/GbtUtility-2.1.50.jar -H 10.1.27.56 -U admin -P Password -RU admin -RP Password \
    redfish bios info
	
the results are saved in:	
# cat results/2022-02-16/redfish/10.1.27.56_redfish_bios.log
These are in the form parameter number: parameter value. eg: "Milan0179": "Auto"

To get a list of parameter names for the parameter numbers, do this:
# java -jar 2.1.50/GbtUtility-2.1.50.jar -H 10.1.27.56 -U admin -P Password -RU admin -RP Password \
    redfish registries get BiosAttributeRegistry
# cat results/2022-02-16/redfish/10.1.27.56_redfish_registries.log
This will contain info like:
        "AttributeName" : "Milan0179",
        "DisplayName" : "Determinism Slider",
		
		
		
		







Workload Profile
	Milan0653
	Koi: Auto
	Suggest: HPC Optimized  or "CPU Intensive"
	
		
Core Performance Boost
	Milan0031
	Koi: Auto
	Options: Auto, Disabled
	Suggest: ??
		
		
Determinism Slider		
	Milan0179
	Koi: Auto
	Suggest: Performance
		
		
System Profile
	Can't find this in Gigabyte BIOS.
	Suggest: Performance
	
	
# cat changeBios.json
{
	"Attributes":{
		"Milan0653": "HPC Optimized",
		"Milan0179": "Performance",
	}
}

Then apply the changes in the file:
# java -jar 2.1.50/GbtUtility-2.1.50.jar -H 10.1.27.65 -U admin -P Password -RU admin -RP Password \
	redfish bios sd post changeBios.json
	
Then reboot
# reboot



```
