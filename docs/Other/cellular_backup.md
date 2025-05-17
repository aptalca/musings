---
tags:
  - Other
---

# Cellular Backup

I spend quite a bit of time away from home and I do have a few somewhat critical services running in my homelab that I would like to be able to access 24/7 remotely. At a minimum, being able to remote into my main workstation allows me to travel light with just a chromebook. Therefore I try to have as many redundancies as feasibly possible, including the following:

- UPSes for servers, switches, APs, router and modem
- PiKVM for servers and OPNsense router
- ZFS mirror on OPNsense with UEFI bootloader on both drives
- ZFS with parity on main server with ZFSBootMenu bootloader on 2 separate drives (one USB and one SD card)

The above list provides decent protection against power loss and disk failures. However, I needed a redundancy for my home internet connection, which was the weakest link. I wanted to add a way to get into my LAN remotely when my cable connection was down to access devices or data and potentially fix any networking issues to bring the cable internet connection back up.

## Goals

- Have a dedicated device on the LAN that will serve as entry point into LAN
- Use a Google Fi data SIM (metered at $10/GB) (need to minimize usage)
- Failover to LTE only when main WAN is down
- Switch back to main WAN as soon as it's back up
- Run punch out services to better handle IP changes due to interface change

## Plan

- Check for WAN connectivity via `ping -I eth0` (requires the interface to be up, even when not connected to ISP)
  - Can't disable interface when there is no ISP connection, because then the connectivity check cannot be done
  - Rely on `metric` instead. Increase when no connectivity, decrease again when connected
- Perform a connectivity check every minute on cron
- Check against 4 different public servers (Google, Cloudflare and Quad9 dns servers)
- If all 4 pings fail, increase eth0 metric to above cellular, making cellular connection the default
- If any of the 4 pings succeed, decrease eth0 metric to make it active again
- Notify and log on connection changes
- Run Tailscale and Cloudflared to make it easier to enter (both services auto update public IP)

## Hardware

I picked up a USB 4G LTE modem, ZTE MF833V, for about $40. It's a category 4 device, which means it can support speeds up to 150/50Mbps. That's plenty for a backup connection mainly for remoting in when the cable internet is down.

I dedicated an older Raspberry Pi 3 to act as the gateway.

## Software

On the rpi3, I decided to go with Ubuntu Server 24.04 running off of an SD card.

The USB modem should be auto detected by the kernel, but it needs some initial configuration to set up the networking type, NAT and APN settings. Mine came with Windows software to do that and I used a Windows VM. Unfortunately I don't remember the details. But once it was set up, plugging into a linux device lets it connect to the LTE network and get an IP.

Mine is set to run DHCP in the range of `192.168.0.0/24`.

When plugged into the rpi3 with Ubuntu 24.04, the modem is detected and the interface is created, but it is not up. I had to use the following netplan config to accomplish 2 things:
1. Set default metric and a static IP for the LAN interface (eth0)
2. Bring up the cellular interface (usb0) with default metric and a static IP

`/etc/netplan/90-usb-modem.yaml`:
```yaml
network:
    ethernets:
        eth0:
            dhcp4: false
            addresses:
              - 192.168.1.20/24
            routes:
              - to: default
                via: 192.168.1.1
                metric: 100
            nameservers:
                addresses: [192.168.1.1]
        usb0:
            dhcp4: false
            optional: true
            addresses: [192.168.0.184/24]
            routes:
              - to: 0.0.0.0/0
                via: 192.168.0.1
                metric: 1000
    version: 2
```
    
eth0 part overrides the default dhcp config / usb0 part enables the USB LTE modem and creates its interface

The following script checks for online connectivity by pinging 4 dns servers and if they all fail, it switches the metric of eth0 (100 > 2000) to make usb0 the default.
When eth0 connectivity is back, it lowers eth0 metric (2000 > 100) to make it default.

`/home/aptalca/failover.sh`:
```bash
#!/bin/bash

fn_notify () {
  curl -Lfs -H "Title: $1" -H "Authorization: Bearer <my token>" -d "$2"  https://ntfy.<mydomain>/zfshost || \
    curl -Ls -H "Title: $1" -d "$2" https://ntfy.sh/<my custom topic>
}

if ! ping -c 1 -W 5 -I eth0 8.8.4.4; then
  if ! ping -c 1 -W 5 -I eth0 8.8.8.8; then
    if ! ping -c 1 -W 5 -I eth0 1.1.1.1; then
      if ! ping -c 1 -W 5 -I eth0 9.9.9.9; then
        if ip route | grep -q "default via 192.168.1.1 dev eth0 proto static metric 100"; then
          ip route del default via 192.168.1.1 dev eth0 proto static metric 100
          ip route add default via 192.168.1.1 dev eth0 proto static metric 2000
          TZ=America/New_York echo "$(date) connection down, deprioritizing eth0" >> /home/aptalca/eth0-status.log
          chown 1000:1000 /home/aptalca/eth0-status.log
          sleep 3
          fn_notify "Pifi eth0 down" "Pifi connection down, deprioritizing eth0"
        fi
        exit 0
      fi
    fi
  fi
fi

if ip route | grep -q "default via 192.168.1.1 dev eth0 proto static metric 2000"; then
  ip route del default via 192.168.1.1 dev eth0 proto static metric 2000
  ip route add default via 192.168.1.1 dev eth0 proto static metric 100
  TZ=America/New_York echo "$(date) connection back up, prioritizing eth0" >> /home/aptalca/eth0-status.log
  chown 1000:1000 /home/aptalca/eth0-status.log
  sleep 3
  fn_notify "Pifi eth0 back up" "Pifi connection back up, prioritizing eth0"
fi
```

Set up a root crontab to run the script every minute

`sudo crontab -e`

`* * * * * /home/aptalca/failover.sh`

I also installed Tailscale and Cloudflared. Tailscale allows ssh access into my LAN. Cloudflared allows me to serve the PiKVM interface over a Cloudflare tunnel. Both services punch out and automatically update the public IP so I can connect whether WAN is up or down.

When WAN is active, which is the great majority of the time, there is no data transfer over cellular so I keep the metered data cost at an absolute minimum.
