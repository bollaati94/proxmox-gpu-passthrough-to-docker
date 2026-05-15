# NVIDIA GPU Passthrough + Docker GPU Setup  
## Proxmox VE → Ubuntu VM → Docker CUDA

# Architecture

```text
NVIDIA RTX PRO 6000 Blackwell DC
            ↓
      Proxmox Host
      (VFIO/IOMMU)
            ↓
        Ubuntu VM
      (NVIDIA Driver)
            ↓
          Docker
     (CUDA Containers)
```

# Part 1 — Proxmox Host Configuration

## 1. Enable IOMMU

Edit GRUB:

```bash
nano /etc/default/grub
```

Intel CPU:

```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"
```

AMD CPU:

```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on iommu=pt"
```

Apply:

```bash
update-grub
```

## 2. Enable VFIO Modules

```bash
nano /etc/modules
```

Add:

```text
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```

Reboot:

```bash
reboot
```

## 3. Verify IOMMU

```bash
dmesg | grep -e IOMMU -e DMAR
```

## 4. Find NVIDIA GPU

```bash
lspci -nn | grep -i nvidia
```

## 5. Bind GPU to VFIO

```bash
nano /etc/modprobe.d/vfio.conf
```

```text
options vfio-pci ids=10de:xxxx,10de:yyyy
```

## 6. Blacklist NVIDIA Drivers on Host

```bash
nano /etc/modprobe.d/blacklist.conf
```

```text
blacklist nouveau
blacklist nvidia
blacklist nvidiafb
```

## 7. Rebuild Initramfs

```bash
update-initramfs -u -k all
reboot
```

## 8. Verify VFIO Binding

```bash
lspci -nnk | grep -A3 -i nvidia
```

Expected:

```text
Kernel driver in use: vfio-pci
```

# Part 2 — Proxmox VM Configuration

## 9. VM Settings

| Setting | Value |
|---|---|
| Machine | q35 |
| BIOS | OVMF |
| CPU Type | host |

## 10. Add GPU to VM

Proxmox UI:

```text
VM → Hardware → Add → PCI Device
```

Enable:
- All Functions
- PCI-Express
- ROM-Bar

Disable:
- Primary GPU

# Part 3 — Ubuntu VM Setup

## 11. Verify GPU Visibility

```bash
lspci | grep -i nvidia
```

## 12. Install NVIDIA Drivers

```bash
apt update
ubuntu-drivers install
reboot
```

Or:

```bash
apt install -y nvidia-driver-580 nvidia-utils-580
reboot
```

## 13. Verify Driver Installation

```bash
nvidia-smi
```

# Part 4 — Docker Installation

## 14. Install Docker

```bash
apt update

apt install -y     ca-certificates     curl     gnupg
```

```bash
install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
```

```bash
apt update

apt install -y     docker-ce     docker-ce-cli     containerd.io     docker-buildx-plugin     docker-compose-plugin
```

# Part 5 — NVIDIA Container Toolkit

## 15. Install NVIDIA Container Toolkit

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
```

```bash
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

```bash
apt update
apt install -y nvidia-container-toolkit
```

```bash
nvidia-ctk runtime configure --runtime=docker
systemctl restart docker
```

# Part 6 — Test GPU from Docker

## 16. CUDA Test

```bash
docker run --rm --gpus all     nvidia/cuda:12.9.0-base-ubuntu24.04     nvidia-smi
```
