Fedora GPU Passthrough Setup Guide
-----

## Hardware:
- Motherboard
    - MSI B450M MORTAR MAX (MS-7b89)
    - BIOS Ver: E7B89AMS.270
    - BIOS Build Date: 04/21/2020
- GPU 1 (PCIe Slot 1) - GIGABYTE AORUS GeForce® GTX 1080 
- GPU 2 (PCIe Slot 4) - NVIDIA Quadro M2000 
- AMD Ryzen 9 3900X
- 64GB RAM
- AMD Matisse USB 3.0 Host Controller (integrated in CPU)
- Intel NVMe SSD (1 TB) (M.2 Slot 1)
- Toshiba SATA SSDs (3x 500 GB) (SATA ports)

## Operating Systems:
- **Host:** Fedora 33
- **Guest 1:** Windows 10 - gaming
- **Guest 2:** Fedora 33 - working

## First time power-on
- Connect only GPU1 to the monitors and leave GPU2 disconnected, otherwise the system will 
- The motherboard may take a while to "warm up" with the newly added hardware or new booting media

## Bios Settings:
Press `DEL` to enter BIOS settings
- enable **Overclocking\CPU Feature\SVM Mode**
- enable **Overclocking\CPU Features\IOMMU** 

## Host preparation:
- install Fedora on the M.2 SSD with the thin LVM partition schema
    - through `custom`
    - add `/boot/efi` partition 
- update and reboot
    - `dnf update`
    - `reboot`
- install virtualization and uefi bios for the VM:
    - `dnf install @virtualization`
    - `dnf install edk2-ovmf` (should be installed already)
- prepare grub with `nano /etc/default/grub` and add to **GRUB_CMDLINE_LINUX=** at the end:
    - `amd_iommu=pt rd.driver.pre=vfio-pci rd.driver.blacklist=nouveau`
- activate modules:
    - `echo "vfio" > /etc/modules-load.d/vfio.conf`
    - `echo "vfio-pci" > /etc/modules-load.d/vfio-pci.conf`
    - `echo "vfio_iommu_type1" > /etc/modules-load.d/vfio_iommu_type1.conf`
    - `echo "vfio_virqfd" > /etc/modules-load.d/vfio_virqfd.conf`
- regenerate initramfs:
    - `dracut -f --kver $(uname -r)`
- perm and generate grub configuration:
    - take a backup `install -m 600 /boot/efi/EFI/fedora/grub.cfg ~/grub.cfg-$(date --iso-8601=minutes).bak`
    - `grub2-mkconfig -o /boot/efi/EFI/fedora/grub.cfg`
    - reboot
- verify if the CMDLINE has been picked up
    - `cat /proc/cmdline`
- check iommu groups and get device IDs:
    - `for g in /sys/kernel/iommu_groups/*; do echo "IOMMU Group ${g##*/}:"; for d in $g/devices/*; do echo -e "\t$(lspci -nns ${d##*/})"; done; done`
    - get all IDs within the group where you devices is member, example output:
        - ```
          IOMMU Group 18:
                  26:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP104 [GeForce GTX 1080] [10de:1b80] (rev a1)
                  26:00.1 Audio device [0403]: NVIDIA Corporation GP104 High Definition Audio Controller [10de:10f0] (rev a1)
          IOMMU Group 22:
                  28:00.3 USB controller [0c03]: Advanced Micro Devices, Inc. [AMD] Matisse USB 3.0 Host Controller [1022:149c]
          ```
        - IDs are: `8086:faf0`, `10de:1b80`, `10de:10f0`, `1022:149c`
- for passthrough I need the IDs from graphic card and USB Controller:
    - `10de:1b80,10de:10f0`
    - `1022:149c`
- add these IDs to **/etc/modprobe.d/vfio.conf**:
    - `options vfio-pci ids=10de:1b80,10de:10f0,1022:149c`
    - reconfigure kernel and grub again:
    - `dracut -f --kver $(uname -r)`
    - `grub2-mkconfig -o /boot/efi/EFI/fedora/grub.cfg`
    - reboot and connect GPU 2 to the monitors. Now GPU 1 will only show the BIOS
      flash screen and Fedora will use GPU 2 as the main output device
- check if **VFIO** driver are loaded:
    - `dmesg | grep -i vfio`
    - also, check every device with: `lspci -nnk -d 10de:1b80`

## install VM
- use **virt-manager**
- add PCI devices to the VM:
    - add hardware > pci host device > ...:
        - add a graphic card, with the audio device
        - add the NVMe M.2
        - add the USB controller
- don't create a virtual hard drive, we use the M.2
- don't create many CPUs, create 1 socket, with cores and threads
- install windows 10
- after installation shutdown VM and edit xml `virsh edit vmname`:
    - after editing `<features></features>` should look like:
    ```
    <features>
        <acpi/>
        <apic/>
        <hyperv>
            <relaxed state='on'/>
            <vapic state='on'/>
            <spinlocks state='on' retries='8191'/>
            <vendor_id state='on' value='1234567890de'/>
        </hyperv>
        <kvm>
            <hidden state='on'/>
        </kvm>
        <vmport state='off'/>
        <ioapic driver='kvm'/>
    </features>
    ```
- boot guest again and install all update

[comment]: # (
## host tuning
- change fedora window manager from wayland to x11 in `/etc/gdm/custom.conf`:
    ```
    ...
    [daemon]
    WaylandEnable=false
    DefaultSession=gnome-xorg.desktop
    ...
    ```
- restart host
- install and configure [barrier](https://github.com/debauchee/barrier)
)

## Known issues:
- When the USB adapter of Dell KM636 keyboard/mouse combo, and a USB flash memory
  stick were both plugged into the front panel, the mouse would stutter.

## References:
- [PCI Passthrough on Fedora 31](https://gist.github.com/jb-alvarado/d6aef18ddb965939442838d7310c5b31)
- [PCI_passthrough_via_OVMF](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF)
- [VFIO-Fedora-Notes](https://qubitrenegade.com/virtualization/kvm/vfio/2019/07/17/VFIO-Fedora-Notes.html)
- [fedora_29_mkinitcpio_doesnt_exist_or](https://www.reddit.com/r/VFIO/comments/a8swkd/fedora_29_mkinitcpio_doesnt_exist_or/)
