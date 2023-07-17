---
author: "Austin Barnes"
title: "Migrating Proxmox Raw LVM to VMDK for VMWare"
description: "Converting Proxmox virtual machine to a suitable format for VMWare."
tags: ["VMWare","Proxmox"]
categories: ["VMWare","Proxmox"]
date: 2023-07-16
image: proxmoxvm-to-vmwarevm.png
---

# Overview

Converting a Proxmox virtual machine to a vmdk for VMWare. In this, I will be doing so with virtual machines in LVM Raw format in Proxmox, into VMDKs for transfer from the Proxmox Hyper-viser to a ESXI hyper-viser.

## Find critical data of Proxmox VM
First within Proxmox, navigate to the VM in question, and see its location, and data type. Assuming it may be a LVM. Ensure you write down the total CPU count and RAM usage, as it should mirror it on first startup in VMWare. Otherwise it may corrupt.

- Gather data for VM in question:
  - `qm status <vmid>`
  - `qm config <vmid>`
- Find pathing:
  - `pvesm path <storage-identifier>:<disk>`
  - `lvdisplay /dev/<storage>/vm-<vmid>-disk-0`

*If you cannot find it, I have noticed Proxmox likes to deactivate LVMs once the VM is offline. You can change this by running `lvscan`, and then running `lvchange --activate /dev/<storage-pool>/<Disk-ID>`*
  

    
## Convert disk from LVM to VMDK
- Once disk is found, convert it to a vmdk
  ``` bash
  qemu-img convert -f raw /dev/storage/vm-<vmid>-disk-0 -O vmdk <name-of-vmdk-file-to-create> 
  ```

## Move the disk
- Copy disk over to your VMWare host (Or specified storage location)
  ``` bash 
  scp <disk-location> <user>@<ip/fqdn>:<destination-file>
  ```

## Clone to a thin provision
I find this step necessary, as in my experience, the disk corrupts if not forced into a thin provision disk.

- SSH into the VMWare host, and go into the directory the file is copied into. Run the following command, it will clone the disk and not delete the copied .vmdk.

  ```bash
  vmkfstools -i <copied-file-name>.vmdk <new-file-name>.vmdk -d thin
  ```

## Create Virtual Machine in vCenter
In your ESXI/vCenter environment, create a new VM, and ensure you add an existing HDD to it, mapped to the location of the new .vmdk disk file. Addiitonally, ensure your drive controller is set correctly. (Likely will default ISCSI, and short be SATA.)

Again, ensure your RAM/CPU configuration matches that of the Proxmox configuration.

If it fails, just try again, its easy to miss one step and have an issue here.



