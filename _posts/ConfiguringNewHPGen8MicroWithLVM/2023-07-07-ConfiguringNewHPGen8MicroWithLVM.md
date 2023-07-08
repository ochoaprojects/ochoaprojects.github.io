---
title: Configuring New HP Gen 8 Micro with LVM 
date: 2023-07-07 12:00:00 -500
categories: [Homelab, ProxMox]
tags: [lvm,proxmox,fdisk,cdisk,storage,linux]
---

A comprehensive guide on configuring RAID 5 and Logical Volume Management (LVM) on an HP server, specifically tailored for the powerful Proxmox virtualization platform. In this post, I will take you through each step of the process, from utilizing HP Smart Storage Admin to creating a RAID 5 logical drive. We'll then cover booting the HP server, listing and manipulating disk partitions, and finally, creating an LVM in Proxmox.

## Table of Contents
- [HP Smart Storage Admin](#hp-smart-storage-admin)
- [Create Logical Drive Raid 5](#create-logical-drive-raid-5)
- [Boot HP Server](#boot-hp-server)
- [List Disk Partition](#list-disk-partition)
- [Manipulate Disk Partition Table](#manipulate-disk-partition-table)
- [Create LVM in ProxMox](#create-lvm-in-proxmox)
- [Closing Notes](#closing-notes)

### HP Smart Storage Admin

![HP Smart Storage Administrator](/project-assets/ConfiguringNewHPGen8MicroWithLVM/hp-smart-storage-administrator.png)

### Create Logical Drive Raid 5

![Create Logical Device](/project-assets/ConfiguringNewHPGen8MicroWithLVM/create-logical-device.png)

### Boot HP Server

![Boot HP Server](/project-assets/ConfiguringNewHPGen8MicroWithLVM/boot-hp-server.png)

### List Disk Partition

```bash
fdisk -l
```

![List Disk Partition](/project-assets/ConfiguringNewHPGen8MicroWithLVM/list-disk-partition.png)

### Manipulate Disk Partition Table

Take note of the output from `fdisk -l` when you find the logical device create by the HP Smart Storage Admin // Raid 5 -- In this case it was `/dev/sda` at 3.7 TB

next command is:
```bash
cdisk /dev/sda
```

Choose `gpt` as a label type
![cdisk Command](/project-assets/ConfiguringNewHPGen8MicroWithLVM/cdisk-command.png)

Choose `New` at the bottom
![Create New](/project-assets/ConfiguringNewHPGen8MicroWithLVM/create-new.png)

Complete!
![Complete](/project-assets/ConfiguringNewHPGen8MicroWithLVM/complete.png)

### Create LVM in ProxMox

1. Create a ***physical volume*** in the `/dev/sda` partition with the following command

```bash
pvcreate /dev/sda1
```

![Create Physical Volume](/project-assets/ConfiguringNewHPGen8MicroWithLVM/create-physical-volume.png)

2. Create a ***volume group*** in the newly created physical volume `/dev/sda1` with the following command

```bash
vgcreate WD-Red-4TB-Bay1 /dev/sda1
```

![Create Logical Volume](/project-assets/ConfiguringNewHPGen8MicroWithLVM/create-logical-volume.png)

3. To create a LVM-Thin disk, we use the command

```bash
lvcreate -L 100G -T -n vmstore WD-Red-4TB-Bay
```

breaking down the command we have following:
- `lvcreate`: this command is to create a logical volume in an existing volume group. then we proceed to add the options in directly afterwards
- `-L`: the first option `-L` is where we define the size of our new logical volume. I have chosen 100Gigs
- `-T`: the second option is `-T` where we define to the "lvcreate program" that we are creating not only a new logical volume, but it is a **THIN** volume. Thus, LVM-Thin! (Logical Volume Management - Thin) Thin is for the Thin-Provisioning which enables features like snapshots
- `-n`: last option is `-n` where we define the name of our new logical volume. In this case i have chosen vmstore. This "vmstore" will be what i see in the web-interface under the "thin-pool" drop down of the "WD-Red-4TB-Bay1" volume group i am creating this logical volume in

After defining our logical volume name, we are telling ProxMox which of our _already_ created "Volume Groups" that we are creating this in. In this case we are creating this in the "WD-Red-4TB-Bay1" volume group
all these things are happening in this one command line

as a result we are presented with these options in the web interface of ProxMox
![Proxmox Storage Result](/project-assets/ConfiguringNewHPGen8MicroWithLVM/proxmox-storage-result.png)

### Closing Notes

This is the command used to resize the logical volume in a Volume group. I did not have space for this resize. It is simply a note for a working command line

```bash
lvresize --size +3.5T --poolmetadatasize +3.5T WD-Red-4TB-Bay1/vmstore
```

![Resize Logical Volume](/project-assets/ConfiguringNewHPGen8MicroWithLVM/resize-logical-volume.png)

This command is used to remove a Logical Volume from a Volume Group

```bash
lvremove -f WD-Red-4TB-Bay1/vmstore
```

![Remove Logical Volume](/project-assets/ConfiguringNewHPGen8MicroWithLVM/remove-logical-volume.png)

In my case I did not want the 100G LVM-Thin. I wanted to use the entire disk. So I removed the logical volume and recreated with the same command with newly desired size

```bash
lvcreate -L 3.63T -T -n vmstore WD-Red-4TB-Bay1
```

![Resize LVM Thin to Desired State](/project-assets/ConfiguringNewHPGen8MicroWithLVM/resize-lvm-thin-to-desired-state.png)

To view the LVM-Thin storage devices in ProxMox, visit the following page in the WebUI
![LVM Thin WebUI](/project-assets/ConfiguringNewHPGen8MicroWithLVM/lvm-thin-webui.png)