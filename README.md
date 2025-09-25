# Setup Virtual Machine Template on Proxmox
How to setup Ubuntu cloud VM template on proxmox using a script.

## Create Script
```bash
# create a shell script with content provided below, from the proxmox node command shell
$ nano create-cloud-vm-template.sh
```

The script below downloads Ubuntu noble version 24. Other Ubuntu versions can be found [here](https://cloud-images.ubuntu.com/)  
_Default username and password is **`ubuntu`** and **`Cool123`**. `sshkey` parameter is optional._

```bash
#!/bin/bash
# Usage: ./create-cloud-vm-template.sh <VMID> [USERNAME] [PASSWORD] [SSH_KEY]
# Example: ./create-cloud-vm-template.sh 5000 ubuntu Cool123 "ssh-ed25519 AAAAC3NzaC1l test"

VMID="$1"
CI_USER="${2:-ubuntu}"
CI_PASSWORD="${3:-Cool123}"
SSH_KEY="${4:-}"

if [ -z "$VMID" ]; then
  echo "Usage: $0 <VMID> [USERNAME] [PASSWORD] [SSH_KEY]"
  exit 1
fi

IMG_FILE="noble-server-cloudimg-amd64.img"
IMG_URL="https://cloud-images.ubuntu.com/noble/current/${IMG_FILE}"
VM_NAME="ubuntu-24-cloud"

# Download the cloud image if not already present
if [ ! -f "$IMG_FILE" ]; then
  wget "$IMG_URL"
fi

# Create VM (host for nest virtualization)
qm create "$VMID" --memory 8196 --core 2 --cpu host --name "$VM_NAME" --net0 virtio,bridge=vmbr0

# qm set "$VMID" --cpu host

# Import disk into ZFS
qm disk import "$VMID" "$IMG_FILE" local-zfs

# Attach disk with proper size
qm set "$VMID" --scsihw virtio-scsi-pci --scsi0 local-zfs:vm-"$VMID"-disk-0

# Add cloud-init disk
qm set "$VMID" --ide2 local-zfs:cloudinit

# Set cloud-init username and password
qm set "$VMID" --ciuser "$CI_USER"
qm set "$VMID" --cipassword "$CI_PASSWORD"

# Set SSH key only if provided
if [ -n "$SSH_KEY" ]; then
  qm set "$VMID" --sshkey <(echo "$SSH_KEY")
fi

# Set boot options
qm set "$VMID" --boot c --bootdisk scsi0

# Resize disk (extra space beyond base image - rounding off to 32G)
qm resize "$VMID" scsi0 +28.5G

# Convert VM to template
qm template "$VMID"

```

## Usage
The following will create a template with id 5000, which you can clone to create your virtual machines.
```bash
# Use it like this - ssh key is optional
$ ./create-cloud-vm-template.sh 5000 ubuntu Cool123 "ssh-ed25519 AAAAC3NzaC1l test"
```

## Test
```bash
# You can create a virtual machine using the template like this. 
# You can right click and clone from the web ui as well.
$ qm clone 5000 101 --name test-vm1 --full
```
