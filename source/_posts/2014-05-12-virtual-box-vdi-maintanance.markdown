---
layout: post
title: "Virtual Box VDI maintanance"
date: 2014-05-12
categories: [linux, virtual box]
---

Virtual Disk Image (_VDI_) is a Virtual Box container format for guest hard
disk. I found that _VDI_ files on the host system grows over the time. If your
_VDI_ file on the host system is much bigger than used spaces on guest partition
it is time for compaction:

1. Install zerofree tool (`apt-get install zerofree`).
2. Remove unused files (`apt-get autoremove`, `apt-get autoclean`, `orphaner --guess-all`).
3. Reboot the guest system in single user mode (hit `e` during Grub boot and append single option to the Grub boot parameters).
4. Remount filesystems as readonly (`mount -n -o remount,ro /`).
5. Fill unused block with zeros (`zerofree /`). It's time consuming operation.
6. Shutdown the system (`poweroff`).
7. Compact VDI files on the host system (`VBoxManage modifyhd my.vdi compact`). It's time consuming operation.

That's all, after the maintenance VDI file size on the host system should be
very close to the used space on the guest partition.

Oh, I forgot to mention: you shoud have a backup before start ;-)
