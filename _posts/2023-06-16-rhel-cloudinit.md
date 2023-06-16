---
title: "Cloud-Init for RHEL/CentOS/Rocky on Proxmox"
date: 2023-06-16 17:15:00 0200
layout: post
categories: [proxmox]
tags: [proxmox,rhel,cloudinit,rockylinux]
---

# Cloud-Init for RHEL/CentOS/Rocky on Proxmox

I have noticed that when provisioning RHEL-based templates, the hostname is always set to localhost. However, there is a workaround that allows cloud-init to set the hostname in the OS as it is in Proxmox.

Note: This solution may also work with other platforms that use cloud-init, such as VMware and OpenStack.

## How to Fix the Problem

To resolve this issue, follow the steps below:

**Step 1: Download the Cloud Image**

Download the cloud image for RHEL/CentOS/RockyLinux. In this guide, we will use RockyLinux.

**Image:** Rocky-9-GenericCloud-Base-9.2-20230513.0.x86_64.qcow2
**URL:** [Download RockyLinux Cloud Image](https://dl.rockylinux.org/pub/rocky/9/images/x86_64/)

**Step 2: Enable NBD on the Proxmox Host**

Start by enabling the Network Block Device (NBD) on the Proxmox host. Run the following command:

```bash
modprobe nbd max_part=8
```

**Step 3: Connect the QCOW2 as a Network Block Device**

Connect the downloaded QCOW2 file as a network block device (NBD) using the following command:

```bash
qemu-nbd --connect=/dev/nbd0 Rocky-9-GenericCloud-Base-9.2-20230513.0.x86_64.qcow2
```

**Step 4: Find the Virtual Machine Partitions**

Use the following command to identify the virtual machine partitions:

```bash
fdisk /dev/nbd0 -l
```

Make a note of the partition, e.g., **/dev/nbd0p5**.

**Step 5: Mount the VM Partition**

Mount the virtual machine partition using the following command:

```bash
mount /dev/nbd0p5 /mnt/somepoint/
```

**Step 6: Edit /etc/cloud/cfg**

Open the `/etc/cloud/cfg` file and add the following line:

```bash
prefer_fqdn_over_hostname: false
```

The updated file should look like this:

```bash
---
# This will cause the set+update hostname module to not operate (if true)
preserve_hostname: false
prefer_fqdn_over_hostname: false
---
```

**Step 7: Unmount and Disconnect**

After making the necessary changes, unmount the partition and disconnect from the network block device:

```bash
umount /mnt/somepoint/
qemu-nbd --disconnect /dev/nbd0
rmmod nbd
```

**Step 8: Create a New Virtual Machine**

Create a new virtual machine with the following command:

```bash
qm create 8000 --memory 2048 --core 2 --name rocky-9-cloud --net0 virtio,bridge=vmbr0
```

**Step 9: Import the Modified RockyLinux Disk**

Import the modified RockyLinux disk into the local-lvm storage:

```bash
qm importdisk 8000 Rocky-9-GenericCloud-Base-9.2-20230513.0.x86_64.qcow2 local-lvm
```

**Step 10: Attach the New Disk to the VM**

Attach the new disk to the virtual machine as a SCSI drive on the SCSI controller:

```bash
qm set 8000 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-8000-disk-0
```

**Step 11: Add Cloud-

Init Drive**

Add the cloud-init drive to the virtual machine:

```bash
qm set 8000 --ide2 local-lvm:cloudinit
```

**Step 12: Make the Cloud-Init Drive Bootable**

Make the cloud-init drive bootable and restrict the BIOS to boot from the disk only:

```bash
qm set 8000 --boot c --bootdisk scsi0
```

**Step 13: Add Serial Console**

Add a serial console to the virtual machine:

```bash
qm set 8000 --serial0 socket --vga serial0
```

**Step 14: Create Template**

Create a template for the virtual machine:

```bash
qm template 8000
```

### Goodbye, localhost!

After following these steps, your virtual machine's hostname should match the one you set in Proxmox. Note that you may need to set the CPU type to "Host" for the image to boot. For more information, refer to this [RHEL 9.0 Proxmox installation troubleshooting thread](https://access.redhat.com/discussions/6959360).

References:
- [How to Mount a QCOW2 Disk Image](https://gist.github.com/shamil/62935d9b456a6f9877b5)
- [Perfect Proxmox Template with Cloud Image and Cloud Init](https://docs.technotim.live/posts/cloud-init-cloud-image/)
