# Notes Proxmox PCI-Passthrough

## Setup for GPU passthrough

Following the tutorial: [Tutorial](http://blog.quindorian.org/2018/03/building-a-2u-amd-ryzen-server-proxmox-gpu-passthrough.html/)

### Change grub parameters to load IOMMU

Ensure this line in `/etc/default/grub`:

```shell
GRUB_CMDLINE_LINUX_DEFAULT="quiet iommu=pt amd_iommu=on video=efifb:off"
```
For intel, it is `intel_iommu=on`.

The run `update grub`. **Reboot** the system and check:

    dmesg | grep -e DMAR -e IOMMU

### Blacklisting drivers

Prevent GPU from initialisation when the proxmox host starts.

```shell
# For NVIDIA GPU
echo "blacklist nouveau" >> /etc/modprobe.d/blacklist.conf
echo "blacklist nvidia*" >> /etc/modprobe.d/blacklist.conf
# For AMD GPU
echo "blacklist amdgpu" >> /etc/modprobe.d/blacklist.conf
echo "blacklist radeon" >> /etc/modprobe.d/blacklist.conf
```

Ensure drivers are loaded during kernel init:

```shell
echo vfio >> /etc/modules
echo vfio_iommu_type1 >> /etc/modules
echo vfio_pci >> /etc/modules
echo vfio_virqfd >> /etc/modules
```
then run

    update-initramfs -u

### Discovery of pci devices

    lspci -v
    lspci -v | grep -i vga

Show the kernel driver loaded for a pci device. In this case, the Nvidia card was used for the proxmox host and the nouveau driver not yet blacklisted.

```shell
root@pvfat:~# lspci -nnk -s 09:00.0
09:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 21 [Radeon RX 6800/6800 XT / 6900 XT] [1002:73bf] (rev c0)
	Subsystem: Advanced Micro Devices, Inc. [AMD/ATI] Radeon RX 6900 XT [1002:0e3a]
	Kernel driver in use: vfio-pci
	Kernel modules: amdgpu
root@pvfat:~# lspci -nnk -s 04:00.0
04:00.0 VGA compatible controller [0300]: NVIDIA Corporation TU106 [GeForce RTX 2070] [10de:1f02] (rev a1)
	Subsystem: Gigabyte Technology Co., Ltd TU106 [GeForce RTX 2070] [1458:37c2]
	Kernel driver in use: nouveau
	Kernel modules: nvidiafb, nouveau
```

Finding vendor ids for each GPU:

    lspci -n -s 09:00
    lspci -n -s 04:00

Edit or create the file `/etc/modprobe.d/vfio.conf` and include all vendor ids to use the 
vfio kernel driver.

```shell
options vfio-pci ids=1002:73bf,1002:ab28,1002:73a6,1002:73a4,1022:1487,10de:1f02,10de:10f9,10de:1ada,10de:1adb disable_vga=1
```

## Resources

- https://pve.proxmox.com/wiki/PCI_Passthrough
- [Tutorial](http://blog.quindorian.org/2018/03/building-a-2u-amd-ryzen-server-proxmox-gpu-passthrough.html/)

## Ansible Role

- Copy the code over to your ansible repos roles directory


## Playbook Example

```yaml
- hosts: desktop_workstation
  tags: ollama
  roles:
   - role: ollama
  vars:
    ollama_remove: false
    # Required if ollama runs on a server or accessed by other machines other than localhost
    ollama_host: 0.0.0.0:11434
    ollama_allowed_origins: "http://{{ ollama_host }}"
    ollama_use_amd_gpu: true
    ollama_update: false
    ollama_service_env: {}
    # Example service env for systemd service
    # OLLAMA_DEBUG: "true"
    # AMD_LOG_LEVEL: 3
    # CUDA_VISIBLE_DEVICES: -1
    # HIP_VISIBLE_DEVICES: 0
    # ROCR_VISIBLE_DEVICES: -1
    # OLLAMA_CONTEXT_LENGTH: 4096
    # HSA_OVERRIDE_GFX_VERSION: gfx1030
```

## Checking ollama logs

- Ensure that the GPU is discovered using `journalctl -fu ollama.service`
- Check if a loaded model uses CPU and/or GPU with `ollama ps`
