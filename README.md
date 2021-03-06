# Install-ESXI-6.7-on-HP-Elitedesk-800 G4

Internal network card: Intel® Ethernet Connection I219LM GbE LOM integrated network connection

## Intro:
The Intel I219LM is not supported. Since no additional PCI network interface exists a USB3.0 to Ethernet 
adapter needs to be used. In this project the StarTech USB31000S was chosen. It is based on the AX88179 chipset.
VMware engineer William Lam created unoffical drivers which support this chipset. Someone using an Realtek chipset adapter can use the driver created by Jose Gomes. The downloads can be found below.

Downloads: \
ESXi6.X ax88179 driver --> https://s3.amazonaws.com/virtuallyghetto-download/vghetto-ax88179-esxi65-bundle.zip \
ESXi6.X realtek driver --> https://www.devtty.uk/homelab/USB-Ethernet-driver-for-ESXi-6.5 \
ESXi6.7 Offline-Bundle --> https://my.vmware.com/web/vmware/details?downloadGroup=ESXI670&productId=742&rPId=24636 \

## BIOS settings of the HP Elitedesk 800 G4:

Starting to install ESXi via USB or CD in UEFI mode will eventually result in the installation getting stuck stating the following:

*Shutting down firmware services
Relocating modules and starting up the kernel…*

So far I have not found a way getting the UEFI install to finish so I switched to legacy.
For unknown reasons the USB stick was not listed in the legacy bootable devices. Therefore the customized ESXi image was burned on a DVD which was read with a USB/CD directly detected. This might be fixed otherwise.

### Summary of BIOS changes:

* Advanced
  * System Options
    * uncheck "Configure Storage Controller for Intel Optane"
    * uncheck "Configure Storage Controller for RAID"
    * enable VTx
  * Secure Boot Configuration
    * Configure Legacy Support and Secure Boot "Legacy Support Enable and Secure Boot Disable" 
  * Boot Options
    * uncheck Fast Boot
    * check Legacy Boot Order and make sure the USB/Optical drive is first in order
    * uncheck UEFI Boot Order

## Creating the custom ISO
The next step was to create a custom installation bundle, including the AX88179 driver, as well as the PCI.e harddrive driver.
The PowerShell script ESXI-Customzer-PS found on https://www.v-front.de/p/esxi-customizer-ps.html 
simplifies and automates the process of creating fully patched and customized ESXi 5.x and 6.x installation 
ISOs using the VMware PowerCLI ImageBuilder module/snapin.

I tried using the command:

.\ESXi-Customizer-PS-v2.6.0.ps1 -v67 -pkgDir /path/to/ax88179/driver

with no success.

At https://www.virten.net/2017/02/heads-up-esxi-not-working-on-7th-gen-kaby-lake-intel-nuc/ another way was described.
It uses the ESXi6.7 Offline-Bundle instead of the ESXi6.7 standard ISO. This was only obtainable as
trial on vmware.com which needs to be revised if its going to be used in production.

This script is to be used in the VMware PowerCLI (commands should be removed before execution)
```
Add-EsxSoftwareDepot .\update-from-esxi6.7-6.7_update01.zip   //Offline bundle
Add-EsxSoftwareDepot .\vghetto-ax88179-esxi65-bundle.zip      //Custom esxi6.5 (6.7 compatible) ax88179 driver
New-EsxImageProfile -CloneProfile "ESXi-6.7.0-20181001001s-standard" -name "ESXi-6.7-ax88179" -Vendor "virten.net" -AcceptanceLevel "CommunitySupported"
Remove-EsxSoftwarePackage -ImageProfile "ESXi-6.7-ax88179" -SoftwarePackage "vmkusb"  //Remove the vmkusb
Add-EsxSoftwarePackage -ImageProfile "ESXi-6.7-ax88179" -SoftwarePackage "vghetto-ax88179-esxi65" //Add the driver
Export-ESXImageProfile -NoSignatureCheck -ImageProfile "ESXi-6.7-ax88179" -ExportToISO -filepath ESXi-6.7-ax88179.iso //Export image profile to an ISO file
```

Then burn the ISO on a CD/DVD.
When booting the ISO the ESXi installer should not prompt "No network adapter..." anymore.

### Installation
After the installation started it is likely to fail at 85% stating something like "no vnic tagged for management, looking for well-known portgroup
unhalted exception".
The user fgrehl stated in the comments here --> https://www.virten.net/2017/02/heads-up-esxi-not-working-on-7th-gen-kaby-lake-intel-nuc/ the following
"Some adapters refuse to automatically connect back to a vSwitch, or connect to a vSwitch during the installation.
This message is displayed after ESXi has been placed on your disk so you can just ignore it. Reboot the NUC and it should boot into ESXi. You can then configure the physical NIC from the DCUI."

After the reboot it no Hardware OS was detected eventhough we can see it in the BIOS. 
Funny note is that if you switch back to UEFI now and boot into the install it will start loading ESXi but halt exactly as described further up.
And here the magic starts...
When booting and having the CD-Drive plugged in and on boot priority #1 we don't choose the iso but boot form local media.
Then its possible to boot into ESXi. The question remains why we can't boot directly in there but at this point I do not care anymore.

The final things to tweak are to reset the network controller so the ESXi gets a new IP address. After logging in we also need to install the Intel(R) Network controller. This can be done under Host-->Manage-->Hardware-->PCI Devices--> Check Intel(R) Network
Controller and reboot.

At that point I could create a test VM with an CentOS7 image.

GL with your install !
 
