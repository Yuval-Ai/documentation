# How To Pass a physical NIC Into a Kata Container
![](https://github.com/Yuval-Ai/documentation/blob/doc-create-NIC-passthrough-and-Kata/use-cases/images/NIC%20passthrough%20Diagram.png)


In this guide we walk through the process of passing a physical NIC into a Kata Container. The "usual" way that a KC (Kata Container) is wired for communication is through a series of bridges and virtual NICs as can be seen in the Image above. 

For some use cases, the container demands a direct link to the physical port of the host device, for example in a situation were the container is required to route high BW traffic without having support for acceleration such as SR-IOV.

#### In this guide:

* [Before you start (Restrictions, Requirements and Assumptions)](#before-you-start-restrictions-requirements-and-assumptions)
* [Pass a physical NIC into a Kata Container](#Pass-a-physical-NIC-into-a-Kata-Container)
  * [Part 1 – The Host](#part-1--the-host)
  * [Part 2 – The Container](#part-2--the-container)
* [Test Your Setup](#Test-Your-Setup)

## Before you start (Restrictions, Requirements and Assumptions)  

The method described in this guide relays on your system [supporting  IOMMU.]( https://en.wikipedia.org/wiki/Input–output_memory_management_unit#Published_specifications) before you start, make sure that your system has this attribute. If it does not, you should look for a different solution then the one presented here. 

If you are uncertain about your system’s support, the first two steps in [Part 1 – The Host](# Part-1-–-The-Host) should help you figure this out. 

 In my setup, I used a server with *Intel(R) Xeon(R) CPU D-1548 @ 2.00GHz* and [support for VT-d](https://en.wikipedia.org/wiki/X86_virtualization#Intel-VT-d), Operated by Ubuntu 18.04.3 LTS. The general steps proposed is this guide should match other hardware and OS with some changes.  

This guide’s starting point is a system with an installed and working *Docker* and *Kata Containers*.

If that is not the case for you:

\-    [How to install Kata Containers]( https://github.com/kata-containers/documentation/tree/master/install)

\-    [How to install Docker]( https://github.com/kata-containers/documentation/blob/master/install/docker/ubuntu-docker-install.md) 



## Pass a physical NIC into a Kata Container 

### Part 1 – The Host

**Note:** all the next commands are run as a super-user. Use `$ sudo -i` to become a SU.



#### 1. Configure system to load with IOMMU support.

   Edit grub file ( */etc/default/grub* )

```bash
vi /etc/default/grub
```

And add ` intel_iommu=on iommu=pt` To the **GRUB_CMDLINE_LINUX** line 

> GRUB_CMDLINE_LINUX=" … quiet intel_iommu=on iommu=pt"



If using UEFI: (like I do) 

/TODO – Change to Ubuntu

```bash
grub2-mkconfig -o /boot/efi/EFI/centos/grub.cfg
```

Else (not using UEFI):

```bash
grub2-mkconfig -o /boot/grub2/grub.cfg
```

 > Expected result
 >
 > Generating grub configuration file ...
 >
 > Found linux image: /boot/vmlinuz-3.10.0-957.el7.x86_64
 >
 > Found initrd image: /boot/initramfs-3.10.0-957.el7.x86_64.img
 >
 > Found linux image: /boot/vmlinuz-0-rescue-dcd5b66ae0534f86a683928888d3f7aa
 >
 > Found initrd image: /boot/initramfs-0-rescue-dcd5b66ae0534f86a683928888d3f7aa.img
 >
 > done



#### 2. Test that the system loads with IOMMU:

```bash
dmesg | head 
```
/TODO – Change to Ubuntu 

> Expected results 
>
> [    0.000000] Command line: BOOT_IMAGE=/vmlinuz-3.10.0-957.el7.x86_64 root=/dev/mapper/centos-root ro crashkernel=auto [rd.lvm.lv](http://rd.lvm.lv)=centos/root [rd.lvm.lv](http://rd.lvm.lv)=centos/swap rhgb quiet LANG=en_US.UTF-8 intel_iommu=on iommu=pt


```bash
dmesg | grep -ie iommu
```

> Expected results
>
> ...
>
> [    1.081716] iommu: Adding device 0000:08:00.0 to group 33 
>
> [    1.081736] iommu: Adding device 0000:09:00.0 to group 34 
>
> [    1.081755] iommu: Adding device 0000:0a:00.0 to group 35 
>
> [    1.081776] iommu: Adding device 0000:0b:00.0 to group 36 
>
> [    1.081795] iommu: Adding device 0000:0c:00.0 to group 37 
>
> ...



#### 3. Enable the *hotplug_vfio_on_root_bus* configuration in the *kata configuration.toml* file:

```bash
vi /usr/share/defaults/kata-containers/configuration.toml
```
a. Set hotplug_vfio_on_root_bus = true

b. Make sure you are using the PC machine type by verifying machine_type = "pc"



#### 4. Install *lshw* and *pciutils* (Ubuntu)

```bash
apt install lshw pciutils
```



#### 5. Find PCI address for the interface you wish to pass to the container

Say I want to pass **enp0s20f1** to the container (see what interfaces you have by running `# ip link` ). 

Get address: 

```bash
lshw -class network -businfo -numeric | grep enp0s20f1
```

> Expected results 
> (something like)
> 
> pci@0000:00:14.1 enp0s20f1 network Ethernet Connection I354 1.0 GbE Backplane



#### 6. Insert the address into a variable

```bash
BDF="0000:00:14.1"
```



#### 7. Load *vfio-pci*  kernel module /TODO maybe set this to permanent? 

```bash
modprobe -i vfio-pci
```
Test that the module loaded correctly: 

```bash
lsmod | grep vfio
```

> Expected results
>
> vfio_pci                      41268     0
>
> vfio_iommu_type1  22300      0
>
> vfio                            32656      2     *vfio_iommu_type1*,vfio_pci
>
> irqbypass                 13503      2      kvm,vfio_pci



#### 8. Unbind the Interface from its current driver. 

```bash
echo $BDF | sudo tee /sys/bus/pci/devices/$BDF/driver/unbind
```

> Expected results
>
> 0000:00:14.1



#### 9.     Find vendor & device ID

```bash
lspci -n -s $BDF
```

> Expected results (something like)
>
> 00:14.1 0200: 8086:1f40 (rev 03)



#### 10.  Bind the device to vfio-pci using the result of the previous step.

```bash
echo 8086 1f40 | sudo tee /sys/bus/pci/drivers/vfio-pci/new_id
```

**Note:** the ` xxxx xxxx ` format is of the correct form and not a mistake.

> Expected results (something like)
> 
> 8086 1f40

#### 11. Check what the new device name is, now under vfio

```bash
ls /dev/vfio
```

> Expected results 
> (something like)
>
> 39  vfio

**Note:** that the requested outcome here is the newly created “39” 

#### All set here. Time to move to the Container’s side of the deal.


### Part 2 – The Container

#### 12.   Create the container you wish to pass the NIC into

**Note:** once running the next command you will be automatically transferred into the container’s bash shell. 
* To disconnect from it whilst keeping the container alive press *Ctrl+p, Ctrl+q.* 
* To create the container without been transferred into it add ‘*d*’ (for detached) to the command line flags (` # docker run -itd `)

```bash
docker run -it --runtime=kata-runtime --name <choose a name> --device /dev/vfio/<device name from previous step> -v /dev:/dev <image> <command>
```
   For example:

```bash
docker run -it --runtime=kata-runtime --name vfio_con --device /dev/vfio/39 -v /dev:/dev ubuntu bash
```
If you aren’t sure what just happened back there, here’s a breakdown of the command:
- `docker run` - creates and runs the container. 
- ` -it ` - docker command flags for “interactive” and “tty”.
- ` --runtime=kata-runtime ` - tells docker to run the container as a kata container (if you set kata-runtime as the default runtime, this option is redundant).
- ` --name vfio_con ` – name the container. It can be any name you choose. 
- ` --device /dev/vfio/39 ` – this one is the important part. With this command, you tell docker to pass the NIC to the container. 
- `-v /dev:/dev `– here you tell docker to map the devices folder of the host to the devices folder of the container.
- ` ubuntu `– choose the container image you want to run. 
- ` bash ` - the command you what the container to run. 

 

#### 13.  Find forwarded device
Update and install pciutils (Ubuntu)

```bash
apt update ; apt install pciutils
```

List devices:
```bash
lspci –nn -D
```

> Expected results 
>
> (something like this between the results)
>
> 0000:00:08.0 Ethernet controller [0200]: Intel Corporation I211 Gigabit Network Connection [8086:1539] (rev 03)

**Note:** the [XXXX:XXXX] address you are looking for is the same one you used in *step 10*

#### 14.   Bind the passed device with a driver in the container.
/ TODO

## Test Your Setup
/TODO
