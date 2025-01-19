---
title: "Infrastructure as Code"
date: 2025-01-19
tags: [Homelab, Terraform, Ansible]
---

From the very beginning, I used an Infrastructure as Code (IaaC) approach in my homelab. However, due to privacy concerns, I couldn't publish it as open source. Recently, I spent a lot of time separating sensitive information so that I could publish the rest as open source 😊

Check it out here: <https://github.com/mkuthan/homelab-public>

## Why IaaC?

What is a main challenge in a homelab? For me it's the same as in a production environment - keeping everything up to date, secure, and reliable while minimizing manual work.

I'm a software engineer, so I'm used to writing beatiful, testable code. In the infrastructure world, it's not that easy.
Fortunately, IaaC tools like Terraform and Ansible help me to write infrastructure code in a way I'm used to.

## Terraform

Terraform defines the following resources in my homelab:

🖥️ Linux containers (LXC) using the [Telmate Proxmox](https://registry.terraform.io/providers/Telmate/proxmox/latest/docs) provider. It covers most of the container resource definitions: CPU, memory, root disk, mount points, networking, SSH keys, and nested virtualization. In the Proxmox UI, I only define replication and high availability settings.

☁️ Virtual Private Server (VPS) with required networking resources in Google Cloud Platform (GCP). I use this VPS for hosting Uptime Kuma to monitor my homelab services.

📦 Bucket on Google Cloud Storage (GCS) for storing offsite backups.

🔒 Tailscale access control lists (ACLs). Thanks to data providers like `tailscale_devices` or `tailscale_users` I'm able to generate ACLs on the fly.

## Ansible

Ansible roles define almost all the software I use in my homelab. I couldn't imagine to maintain all that stuff manually.
Here are some examples:

🛡️ Adguard DNS

📦 Apt Cacher NG

🛠️ Backup Ninja

🐳 Docker

📹 Frigate

📊 Grafana

📈 Grafana Agent

👴 Gramps

🌈 Hyperion NG

📸 Immich

🎥 Kodi

📂 Loki

📧 Mailrise

🐝 Mosqquitto

🔋 NUT

🌐 Omada Software Controller

📄 Paperless NGX

💾 Proxmox Backup Server

📈 Prometheus

🎵 Raspotify

🔄 RClone

🖥️ Samba

🔍 SearXNG

🎶 Shairport

📄 Stirling PDF

🔒 Tailscale

🚀 Traefik

📡 Transmission

📊 Uptime Kuma

🔐 Vaultwarden

🔍 Whoogle

📡 Zigbee2MQT

If you're interested in how these services are set up in my homelab, you can explore the playbooks. Here are some examples: [Proxmox hosts](https://github.com/mkuthan/homelab-public/blob/main/ansible/playbooks/pve.yml),
[Raspberry Pi](https://github.com/mkuthan/homelab-public/blob/main/ansible/playbooks/pi.yml),
[VPS](https://github.com/mkuthan/homelab-public/blob/main/ansible/playbooks/vps.yml).

Please note that I use a dynamic Ansible inventory for all my Linux containers. You can find more details in the [inventory.proxmox.yml](https://github.com/mkuthan/homelab-public/blob/main/ansible/inventory.proxmox.yml) file. The static inventory includes only non-virtualized hosts such as Proxmox VE, Raspberry Pi, and VPS.

## Conclusion

I hope you find my homelab setup useful and inspiring. If you have any questions, feel free to ask me on [GitHub Discussions](https://github.com/mkuthan/homelab-public/discussions).
