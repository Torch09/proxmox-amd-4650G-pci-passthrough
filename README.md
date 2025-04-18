# Proxmox AMD iGPU Passthrough + M.2. Google Coral
Most of this works only because isc30 and matt22207 have published their findings, so I'll do the same.  
Links to the Repos are below.

# Info
These are the steps I used to get GPU passthrough for AMD APUs inside Proxmox working.  
We'll only use a VM, this is no LXC guide! (This isn't really a guide at all)

Some guides and Discussions I followed:  
- https://github.com/isc30/ryzen-gpu-passthrough-proxmox
- https://gist.github.com/matt22207/bb1ba1811a08a715e32f106450b0418a
- https://forum.proxmox.com/threads/amd-ryzen-7-renoir-4750g-apu-and-igpu-pass-thru-to-windows-10-guest.84849/
- https://forum.proxmox.com/threads/gpu-passthrough-ryzen-4600g-apu.120151/

# Host Configuration
Make sure you have enabled SR-IOV, IOMMU, iGPU, etc. There is an example for the M12-LE0 near the bottom of this "guide".   
Updating the CPU Microcode is also a good idea.  
I'm using Proxmox 8.4.1 for this guide.  
The CPU is a 4650G and the Mainboard I'm using is a Gigabyte M12-LE0.  

## Modproble
Get the IDs for the APU
``` bash
lspci -nnk
```
this returns a long list, look for the ```0b:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Renoir [1002:1636] (rev d9)``` entry, if there isn't any, double check that you've enabled the iGPU inside the BIOS.  
The Numbers will be different in your case.  
Copy the ID ```1002:1636``` and look in the next lines for the next ID (as long as the numbers at the beginning match, example: ```0b:00.0; 0b:00.1; 0b:00.2```)

Create/Edit a modprobe file and add the IDs you copied in there, example: ```/etc/modprobe.d/vfio.conf```
``` /etc/modprobe.d/vfio.conf
options vfio-pci  ids=1002:1636,1002:1637,1022:15df,1022:1639,1022:15e2,1022:15e3"
```
also add the softdep lines, so the config should look like this:
```/etc/modprobe.d/vfio.conf
options vfio-pci ids=1002:1636,1002:1637,1022:15df,1022:1639,1022:15e2,1022:15e3
softdep radeon pre: vfio-pci
softdep amdgpu pre: vfio-pci
softdep snd_hda_intel pre: vfio-pci
```
this will load the vfio-pci drivers before the original ones

Next we'll blacklist AMD entries:
The file is: ```/etc/modprobe.d/amd-blacklist.conf```
```
blacklist radeon
blacklist amdgpu
blacklist snd_hda_intel
blacklist ccp
```

## GRUB < Not needed if Proxmox uses UEFI
Edit the ```GRUB_CMDLINE_LINUX_DEFAULT=``` line inside the ```/etc/default/grub```file
We'll blacklist, add iommu, disable framebuffers
My current config looks like this:
```/etc/default/grub
GRUB_DEFAULT=0
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`
GRUB_CMDLINE_LINUX_DEFAULT="quiet initcall_blacklist=acpi_cpufreq_init amd_pstate=passive amd_pstate.shared_mem=1 amd_iommu=on iommu=pt nomodeset initcall_blacklist=sysfb_init video=efifb:off video=vesafb:off video=simplefb:off"
GRUB_CMDLINE_LINUX=""
```

## Modules
Now we'll add kernel modules so we can use vfio
The file is: ```/etc/modules```
```/etc/modules
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```

## Apply and restart
Update initramfs:
```bash
update-initramfs -u -k all
```
after that, reboot proxmox

## Check Settings
Run:
```bash
lspci -nnk
```
Now under ```Kernel driver in use:``` the drive should be: ```vfio-pci```

Thats all for the Host Machine!


#  VM
**!! MACHINE MUST BE Q35-7 !!**
## Create and Configure a VM
### Hardware: 
- BIOS: UEFI
- Machine: Q35-7

### Install OS
Boot the OS without PCI Devices and install the OS like you would normally do.  
After the installation is finished, reboot, install updates and shutdown.

### APU vBIOS File
Use this as GPU ROM (Renoir):  
Link: https://forum.proxmox.com/threads/amd-ryzen-7-renoir-4750g-apu-and-igpu-pass-thru-to-windows-10-guest.84849/post-418165

There is also one for Cezanne, haven't tested it:  
https://forum.proxmox.com/threads/amd-ryzen-7-renoir-4750g-apu-and-igpu-pass-thru-to-windows-10-guest.84849/post-549222

Move the file to: ```/usr/share/kvm/```   
If you want to extract the vBIOS yourself, there is a small roadmap at the bottom of this page.

### Configure VM
We'll edit the qemu-server config for the VM we created, in my example the ID of my VM is "101"  
My Config has two PCI Devices:  
- hostpci0: 0000:0b:00.0: GPU  
- hostpci1: 0000:01:00: Google Coral, all Functions passthrough  

We will make the following changes:
- replace the "cpu" line with: ```cpu: host,hidden=1```
- append ```pcie=1,romfile=vbios_renoir.bin,x-vga=1"```to the GPU PCI line

The name ```vbios_renoir.bin```must match the name of the file you pasted into ```/usr/share/kvm/```

The Config File after the changes:  ```/etc/pve/qemu-server/101.conf```
``` /etc/pve/qemu-server/101.conf
agent: 1
bios: ovmf
boot: order=scsi0;ide2;net0
cores: 6
cpu: host,hidden=1
efidisk0: local-nvme:vm-101-disk-0,efitype=4m,pre-enrolled-keys=1,size=4M
hostpci0: 0000:0b:00.0,pcie=1,romfile=vbios_renoir.bin
hostpci1: 0000:01:00
ide2: none,media=cdrom
machine: pc-q35-7.2
memory: 8192
meta: creation-qemu=9.2.0,ctime=1744835205
name: frigateNVR
net0: virtio=BC:24:11:87:B0:FB,bridge=vmbr1,firewall=1,tag=40
net1: virtio=BC:24:11:EB:CD:CB,bridge=vmbr1,tag=89
numa: 0
onboot: 1
ostype: l26
scsi0: local-nvme:vm-101-disk-1,iothread=1,size=120G,ssd=1
scsi2: /dev/disk/by-id/ata-WDC_WD30PURX-64P6ZY0_WD-WCC4N4UL72CR,backup=0,size=2930266584K
scsihw: virtio-scsi-single
smbios1: uuid=32943f07-d1b5-452a-aa74-5f5ef94cd4ae
sockets: 1
startup: order=31
tags: docker-host
vmgenid: e9a8a581-0b37-4db8-83ba-6e1240174bbd
```


## Configure Guest
There can be a PSP error, we'll add two boot parameters to grub to fix this  
**INFO: WE ARE EDITING ON THE GUEST VM, NOT THE HOST**  
Add the following to: ```/etc/default/grub```
```/etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="amdgpu.fw_load_type=0 amdgpu.tmz=0"
```
After that is inserted, update grub:
```bash
update-grub
```
and reboot

## FINISHED
After the reboot, you should have a ```renderD128```and a ```card1```inside ```/dev/dri```
you can also check if the correct drivers are used with the following command:
```bash
lspci -nnk
```
the value for the AMD GPU should now be:
```
1:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Renoir [Radeon RX Vega 6 (Ryzen 4000/5000 Mobile Series)] [1002:1636] (rev d9)
Subsystem: Gigabyte Technology Co., Ltd Renoir [Radeon RX Vega 6 (Ryzen 4000/5000 Mobile Series)] [1458:1000]
Kernel driver in use: amdgpu
Kernel modules: amdgpu
```
If the ```Kernel driver in use: amdgpu``` is there, everything should work fine.

# Troubleshooting
## The iGPU can't be resetted form the guest
Run this on the host to reset it:
```
echo "1" > /sys/bus/pci/devices/0000\:0b\:00.0/remove 

delay

echo "1" > /sys/bus/pci/rescan
```
From: https://forum.proxmox.com/threads/pcie-card-falls-off-the-bus-after-vm-reboot-how-can-i-automatically-reset-it.122917/

Haven't tested, but here is also workaround for the reset:
https://forum.proxmox.com/threads/amd-ryzen-5600g-igpu-code-43-error.138665/#post-726791
# Specific Hardware Configuration
## Gigabyte MC12-LE0
- Update at least to BIOS F18
- Boot OS, go into BIOS -> Key: F2, BMC BIOS Viewer won't work
	- Advanced
		- PCI Subsystem
			- Above 4G Decoding: Enabled
			- SR-IOV Support: Enabled
		- AMD CBS
			- NBIO
				- IOMMU: Enabled
				- GFX: Don't use "AUTO" choose "UMA_AUTO"
	- Security
		- Secure Boot: Disabled < Haven't tested if needed
	- Boot
		- Boot mode select: UEFI
- Look over the settings and configure for your needs

# Extract vBIOS from BIOS updates
The vBIOS is, not always, included inside the BIOS update, you just have to find one, for the correct APU you are using.  
I've used the BIOS update for the "Lenovo ThinkStation P358".  
Download the .exe and run "VBiosFinder" (https://github.com/binboum/VBiosFinder/tree/implement-docker-image), this should extract 2 Files, one is the correct "Renoir" one.  
You can add it to your GPU on the Hardware -> PCI Device tab in Proxmox.  
If you run "dmesg" inside the guest, it tells you what kind of ROM it initializes.  
If its the wrong on, use the other file.
  
   
# Google Coral
Link: https://coral.ai  

**Everything is done on the GUEST, not the Host!**

# Install M.2 or Mini PCIe Accelrator
```
echo "deb https://packages.cloud.google.com/apt coral-edgetpu-stable main" | sudo tee /etc/apt/sources.list.d/coral-edgetpu.list
```
```
curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```
```
sudo apt-get update
```
if your system is old enough, you can use the gasket-dkms from Google:
```
sudo apt-get install gasket-dkms libedgetpu1-std
```
else, you install only the libegetpu1-std:
```
sudo apt-get install libedgetpu1-std
```
and build the gasket-dkms
```
apt remove gasket-dkms
```
```
apt-get install -q -y --no-install-recommends git curl devscripts dkms dh-dkms build-essential debhelper
```
```
cd /opt/ && git clone https://github.com/google/gasket-driver.git && cd gasket-driver
```
```
debuild -us -uc -tc -b
```
```
cd .. &&  dpkg -i gasket-dkms_1.0-18_all.deb
```
```
reboot
```
Now if you run:
```
ls /dev/apex*
```
the returned value should be:
```/dev/apex_0```
