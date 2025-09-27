## VMware VMDK VM migration to Proxmox VE

This guide provides a step-by-step process for migrating a **VMware virtual machine (VMDK)** to **Proxmox VE**.  
It covers disk transfer, format conversion, import into Proxmox, and guest optimization.

<img width="1024" height="386" alt="image" src="https://github.com/user-attachments/assets/45e0556a-daa1-4f4f-96a7-9a48f9a952db" />

---

### ✅ Pre-Migration Checklist
- VM must be **powered off** in VMware.
- Ensure all snapshots are deleted/committed (Proxmox import doesn’t handle VMware snapshots well)
- Check source VM disk size.  
- Ensure Proxmox node has enough **free space** on target storage.  
- Note down the VM’s **hardware resources** (CPU, RAM, NICs, disk sizes).  
- Verify **network connectivity** between ESXi and Proxmox.  
- Use `screen` or `tmux` for long-running operations.  

---

### 🔹 Step 1: Access VMware ESXi and Locate the VM Directory
```bash
ssh root@10.192.192.50
cd /vmfs/volumes/datastore1/xyz-frontend-vm
```

---

### 🔹 Step 2: Copy VMDK Files to Proxmox
- Confirm the following exist:
  - `xyz-frontend-vm.vmdk` → small descriptor file.  
  - `xyz-frontend-vm-flat.vmdk` → large data file.  

- Check current rulesets & Enable SSH client ruleset
```bash
esxcli network firewall ruleset list
esxcli network firewall ruleset set -e true -r sshClient
```
- Copy both to the Proxmox node:

```bash
scp xyz-frontend-vm.vmdk root@10.192.192.100:/root/
scp xyz-frontend-vm-flat.vmdk root@10.192.192.100:/root/
```

---

### 🔹 Step 3: Create a New VM Shell in Proxmox or In the Proxmox GUI, create a new VM with the desired ID (VMID).

- Choose do not create/attach a disk (since you’ll import one).
- Configure CPU, RAM, and network similar to the VMware VM.

```bash
ssh root@10.192.192.100
qm create 105 --name app1-vm --memory 2048 --cores 2 --scsihw pvscsi
```

---

### 🔹 Step 4: Convert VMDK to QCOW2
```bash
cd /root/
qemu-img convert -f vmdk -O qcow2 xyz-frontend-vm.vmdk vm-disk.qcow2
ls -lh vm-disk.qcow2
du -sh vm-disk.qcow2
```

---

### 🔹 Step 5: Import Disk into Proxmox Storage
```bash
qm importdisk 105 vm-disk.qcow2 local-lvm
lvs
```

---

### 🔹 Step 6: Attach Disk to VM
```bash
qm set 105 --scsi0 local-lvm:vm-105-disk-0
qm set 105 --boot order=scsi0
```

---

### 🔹 Step 7: Configure Networking
Use the Proxmox Web GUI:

1. **VM → Hardware → Add → Network Device**  
2. Choose `virtio` (recommended) or `E1000` for compatibility  
3. Attach to correct bridge (e.g., `vmbr0`)  

---

### 🔹 Step 8: Optimize Guest OS

#### Remove VMware Tools
- **Legacy VMware Tools:**
```bash
sudo vmware-uninstall-tools.pl
```

- **open-vm-tools:**
```bash
# Debian/Ubuntu
sudo apt purge open-vm-tools open-vm-tools-desktop -y

# RHEL/CentOS
sudo yum remove open-vm-tools -y
```

#### Install QEMU Guest Agent
```bash
# Debian/Ubuntu
sudo apt install qemu-guest-agent -y
sudo systemctl enable --now qemu-guest-agent

# RHEL/CentOS
sudo yum install qemu-guest-agent -y
sudo systemctl enable --now qemu-guest-agent
```

Enable it in Proxmox GUI: **VM → Options → QEMU Guest Agent → Enable**

---

### 🔹 Step 9: Start the VM
```bash
qm start 105
```

---

### ⚠️ Troubleshooting
- If VM fails to boot:
  - Switch disk controller to `lsi` or `sata`.  
  - Boot with OS install ISO → Rescue mode → rebuild initramfs:  
    ```bash
    dracut -f -v        # CentOS/RHEL
    update-initramfs -u # Debian/Ubuntu
    ```
- For Windows VMs: attach VirtIO ISO and install drivers.

---


