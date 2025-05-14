---
tags:
  - Other
---

# Zfshost
Details of my new mini homelab server

## Hardware
- Jonsbo N2 case
- Purple CWWK/Topton board with i3-N305 processor and Jonsbo CPU fan (2 eth instead of 4, pci-e port has its own lane so it can be used alongside both M2 ports)
- Corsair 48GB mem stick (4800 version)
- 2 1TB WD Blue Nvme on the board M2 slots (1X each unfortunately so not full speed) in a ZFS mirrored pool for the OS
- 5 8TB WD (mostly red) HDD in drive bays in a ZFS RAIDZ2 pool (24TB usable) for long term storage
- 1 256GB SSD (haven't decided what to use it for yet, probably temp files like `/var/lib/docker` content to increase the life of NVMEs)
- 750W Thermaltake SFX PSU
- Coral dual tpu in a pci-e adapter
- PiKVM ATX controller (hooked up to motherboard header pins)
- USB header splitter and adapter to connect the case's USB A and USB C ports to the motherboard's single USB 3 header port (I'm honestly surprised this system worked because plenty of folks online said it wouldn't)
  - [USB 3 header splitter](https://www.amazon.com/dp/B0CH4HYYT8)
  - [USB 3 to type C header adapter](https://www.amazon.com/dp/B0CPFRZL6Y)
- [6 in 1 SATA cable](https://www.amazon.com/dp/B0B1CZHXZ1) (case is very cramped and full SATA cables make ot more crowded in there)
- 1 32GB sdcard as UEFI #1 with ZfsBootMenu
- 1 32GB USB flash drive (in mobo USB slot) as UEFI #2 with ZfsBootMenu (for redundancy)
- APC UPS connected to a USB2 port in the back

## Software

### UEFI
Custom build of [ZfsBootMenu](https://docs.zfsbootmenu.org/en/latest/) on 2 separate devices (1 sdcard and 1 USB stick for redundancy). Used the [container build](https://docs.zfsbootmenu.org/en/latest/general/container-building/example.html) method but added [Tailscale](https://docs.zfsbootmenu.org/en/latest/general/tailscale.html) and [NTFY](https://github.com/aptalca/mkinitcpio-ntfy).

ZfsBootMenu waits for passphrase and once entered, unlocks all zfs pools, searches for bootable datasets, provides them as options (for optional multi-boot). It also allows for zfs recovery and snapshot management. It also caches the ZFS keys when launching the OS so pools/datasets can be mounted automatically.

Custom build includes:
- Tailscale (for remote access)
- Dropbear (for remote access)
- Notifications via Discord and NTFY (so that when it boots after a power failure, I'm notified to ssh in and enter the ZFS unlock passphrase to minimize down time)

_Tip: When setting up your network access in ZfsBootMenu with a static IP, use the interface name `eth0` or `eth1` because that's what the VoidLinux based ZfsBootMenu uses instead of the more modern `enpXsX` scheme. For instance if your Debian install lists two network interfaces `enp4s0` and `enp5s0`, ZBM will identify them as `eth0` and `eth1` respectively._

Here's my tree for the zfsbootmenu build folder:
```
/etc/zfsbootmenu
├── build                               - Output folder for built EFIs. Always keeps one older version named backup by default.
│   ├── zfsbootmenu-backup.EFI            I do "cp -a /etc/zfsbootmenu/build/zfsbootmenu.EFI /boot/efi/EFI/BOOT/BOOTX64.EFI" to
│   └── zfsbootmenu.EFI                   update the EFI on the mounted sd card (and USB flash drive mounted to "/boot/efi2")
├── cleanup.d
│   └── hostfiles                       - Cleans up temp files at the end of build
├── config.yaml                         - Enables initcpio and other settings for the build
├── dropbear                            - Contains the dropbear config and keys
│   ├── dropbear.conf                   - Contains the listen port argument
│   ├── dropbear_ecdsa_host_key         - Host keys don't need to exist as they would be created on first run, then they would be reused
│   ├── dropbear_ecdsa_host_key.pub
│   ├── dropbear_ed25519_host_key
│   ├── dropbear_ed25519_host_key.pub
│   ├── dropbear_rsa_host_key
│   ├── dropbear_rsa_host_key.pub
│   └── root_key                        - Contains my public ssh key for key based ssh login
├── initcpio
│   ├── hooks                           - Contains runtime hooks for when ZfsBootMenu loads
│   │   ├── ntfy                        - Sends notification with a 10 sec delay (also has a discord notification added)
│   │   ├── dropbear                    - Starts the dropbear service
│   │   ├── rclocal                     - Sets up networking
│   │   └── tailscale                   - Starts the tailscale service
│   ├── install                         - Contains build time hooks for building ZfsBootMenu
│   │   ├── ntfy                        - Copies the ca-certificates into UEFI for curl ssl and enables the runtime hook
│   │   ├── dropbear                    - Creates host keys if needed, copies the keys and dropbear binary into UEFI, and enables the runtime hook
│   │   ├── rclocal                     - Enables the runtime hooks that enable networking for ZfsBootMenu
│   │   └── tailscale                   - Copies the tailscale and dep binaries and the state file into UEFI, and enables the runtime hook
│   └── rc.local                        - Contains the networking config (interface, routes and dns)
├── mkinitcpio.conf.d                   - Contains the conf files for initcpio modules (hooks and binaries) needed to be installed/enabled in ZfsBootMenu
│   ├── ntfy.conf                       - Adds the curl binary and enables ntfy hooks
│   ├── dropbear.conf                   - Enables dropbear hooks
│   ├── network.conf                    - Enables the rclocal hooks for networking
│   └── tailscale.conf                  - Enables the tailscale hooks and adds the ip and dhclient binaries
├── rc.d                                - Contains scripts to prepare the build container environment
│   ├── dropbear                        - Copies the dropbear config files into the build container filesystem
│   └── tailscale                       - Copies the tailscale config files into the build container filesystem
├── tailscale                           - Contains the tailscale config and state files
│   ├── tailscaled.conf
│   └── tailscaled.state
├── zbm-builder.conf                    - Config for the zbm builder
└── zbm-builder.sh                      - Script that builds the ZfsBootMenu UEFI in a docker container and outputs to build folder

10 directories, 31 files
```
With all config files in place, simply cd to the folder and run `zbm-builder.sh` from inside that folder. It creates a docker build container with the current folder mounted, and it will use the various rc.d and mkinitcpio init files to prepare the build environment, build the UEFI image, and output to `build/`. You can then copy it to your boot drive at `/EFI/BOOT/BOOTX64.EFI`.


### OS
Debian (Bookworm) server installed on the dual NVME mirrored ZFS pool. Protected by ZFS passphrase (key also stored in the pool for caching and automounting).
Install followed ZfsBootMenu instructions: https://docs.zfsbootmenu.org/en/latest/guides/debian/bookworm-uefi.html with a few changes:
- Followed the `Separate Boot Device` instructions but skipped the parts about creating a partition on the NVME for the ZFS pool (because I'm using a separate device for the UEFI). I instead created the ZFS pool using the full disk.
- Created an encrypted pool.
- Instructions make you use a single disk for the pool. Afterwards I converted it to a dual mirror by adding the second NVME to the pool.

_Tip: Stock Debian server does not come with any ntp client/service installed so eventually time shifts and operations involving time based cryptography start failing. Don't forget to install the ntp package (or alternative) to keep time synced. I found out through Authelia crashing on start._

When I first set up stock Debian, I noticed some startup log errors about the GPU. I also noticed that hw transcode wasn't working in containers when passing the DRI device. Researching the issue, I became aware of two potential causes. Unfortunately, I applied both fixes at once so I can't tell for sure now if both are necessary or if one would suffice. But I personally would have eventually implemented both of these for other reasons anyway:
- Upgraded the kernel to the backports version because I thought my gpu was too new for the stock Debian kernel for full support. That may or may not be true, but I needed the newer kernel for other reasons as well so I'm keeping it.
- Added `non-free-firmware` to Debian apt sources (`/etc/apt/sources.list`) so the Intel drivers/firmware are upgraded.

Upgrading the kernel lead to another issue, this time with the Coral pci-e driver for the Edge TPU. Apparently, Google sucks at keeping the driver up to date. Following their [official directions](https://coral.ai/docs/m2/get-started/#2a-on-linux) results in an error during compilation. After some rabbit hole exploring, I came across an unmerged PR that solves the issue with compiling on the 6.12 kernel: https://github.com/google/gasket-driver/pull/35 I had to locally build the DKMS deb with that patch and install it.

### ZFS

#### ZFS Pools

OS pool was created above. I then created an encrypted ZFS pool made up of the 5 8TB HDDs in RAIDZ2 (2 parity) for a usable space of 24TB. When creating, make sure to use the devices `/dev/disk/by-id` instead of `/dev/sdX`. `lsblk` is your friend.

Resources:
- https://docs.zfsbootmenu.org/en/latest/guides/debian/bookworm-uefi.html#create-the-zpool
- https://virtualize.link/Other/zfs/#creation

#### ZFS Dataset Structure

My home folder is on its own dataset `zroot/ROOT/home` and it contains all my docker app data (persistent data) at `/home/aptalca/appdata`.

`/var/lib/docker` is on its own dataset `zroot/docker` along with my openvscode-server's dind docker folder and the `modcache` as all of that is ephemeral and I can set the least aggressive snapshot plan for it.

My long term storage pool (HDD based) `zarray` has datasets for `Media`, `Books`, `Pictures` and `Misc` (contains backups and other miscellanous data).

#### ZFS Auto Mount

Unfortunately Debian's ZFS package no longer auto mounts pools and datasets. Apparently it was disabled due to a bug that affected some people a while back.

To enable, follow the instructions [here](https://www.reddit.com/r/zfs/comments/lway2o/comment/gpk9c66/) (essentially do `sudo systemctl edit zfs-mount` and add the missing `-l` to the `zfs mount` command so it loads the cached keys when attempting the mount).

To make sure the key for the HDD pool is cached by ZfsBootMenu, make sure to add the property `org.zfsbootmenu:keysource` to the pool. If using the same passphrase as the OS pool (highly recommended), you'd set it to `org.zfsbootmenu:keysource="zroot/ROOT/debian"` along with `keylocation=file:///etc/zfs/zroot.key` and `keyformat=passphrase`. That way ZfsBootMenu will know to unlock and mount the OS pool first, and cache the key in `/etc/zfs/zroot.key` for both pools.

#### ZFS Snapshots

The auto-snapshot package is highly recommended. By default it does frequent (every 15 minutes), hourly, daily, weekly and monthly snapshots on all pools and datasets. The behavior can be customized by setting properties in pools and datasets.

I have disabled daily, weekly and monthly on the OS dataset (`zroot/ROOT/debian`). I also moved `/var/lib/docker` out of the OS dataset into a newly created `zroot/docker` dataset as it's all ephemeral data and did not need to mingle it with the rest of the OS.

#### ZFS Alerts

By default the zfs package on Debian enables monthly ZFS scrubs. The alerts are managed by ZED, which can be customized to send notifications via various methods.

I enabled NTFY via [instructions here](https://virtualize.link/Other/zfs/#alerts) and emails via msmtp (no mta package installed, just the `/etc/mail.rc` edited with the content `set sendmail=/usr/bin/msmtp` and `ZED_EMAIL_PROG="mail"` set in `zed.rc`.

I enabled both for redundancy. I use my own self hosted NTFY server and there are brief instances where it may not be accessible. In that case, email notification is a good backup.

### APC UPS

Set up apcupsd with [instructions here](https://wiki.debian.org/apcupsd).

I created a custom scripts folder at `/etc/apcupsd/customscripts` and copied all the scripts into that folder to prevent the scripts from getting overwritten during a package update.

In `/etc/apcupsd/apcupsd.conf`, I set `SCRIPTDIR /etc/apcupsd/customscripts`.

In `/etc/apcupsd/customscripts/apccontrol` I set `SCRIPTDIR=/etc/apcupsd/customscripts`

And I modified the scripts `onbattery`, `offbattery` and `doshutdown` to add email and NTFY notifications. As an example, my `onbattery` script contains the following:
```sh
printf "To: myemail@gmail.com\nFrom: Zfshost <myemail@gmail.com>\nSubject: Power loss at home\n\nPower went out at home on %s\n\n%s" "$(/usr/bin/date)" "$(/sbin/apcaccess status)" | /usr/bin/msmtp --auth=on --tls=on --host smtp.gmail.com --port 587 --user myemail@gmail.com --read-envelope-from --read-recipients --password 'echo <gmail app password>' --logfile /home/aptalca/msmtp.log
chown 1000:1000 /home/aptalca/msmtp.log
curl -fs \
  -H "Authorization: Bearer <my_token>" \
  -d "Power went out at home on $(date)" \
  https://ntfy.<mydomain>/zfshost || \
curl \
  -d "Power went out at home on $(date)" \
  https://ntfy.sh/<mycustomtopic>
exit 0
```
So it first sends me an email, then sends me an NTFY notification through my self hosted instance. If that fails, it notifies me through the public instance of NTFY (which is less reliable as its notifications aren't totally instant, but it has better uptime than my self hosted instance, therefore it's a decent backup).

I get notified when the power goes out, when the power comes back (if within the DELAY period set) and when the server decides to shut down.

### SMART Alerts

Installed the smartmontools package and modified the `/etc/smartmontools/smartd.conf` to enable both email alerts (via msmtp) and [via NTFY](https://virtualize.link/Other/smartd/).

It performs automatic tests and monitors disk tempreature. Notifies if there are any issues.

### Docker

Installed docker from the official repos: https://docs.docker.com/engine/install/debian/

Put the `/var/lib/docker` contents into the ZFS dataset `zroot/docker` mounted to `/mnt/docker` by adding `"data-root": "/mnt/docker/docker"` into `/etc/docker/daemon.json` so I could disable the auto snapshots. Currently debating whether to move that to the SSD instead (the one connected to the 6th SATA port on the mobo and not added to any ZFS pools).

I also put the [modmanager](https://github.com/linuxserver/docker-modmanager) data (`/mnt/docker/modmanager`) as well as the `/var/lib/docker` content of my `openvscode-server`'s DIND (`/mnt/docker/dind-openvscode-server`) in that same dataset. None of those 3 sources of data need to be retained and can easily be recreated by running the containers again.

The persistent data of all containers reside under `/home/aptalca/appdata`. The home folder is on its own ZFS dataset with an aggressive snapshot schedule as that data is crucial. Compose yamls (one for Immich specifically and one for all others) are at `/home/aptalca`.

Keeping most container arguments in a single yaml results in a very long file. I took advantage of yaml anchors [described here](https://virtualize.link/Containers/yaml-anchors/) to make it more manageable.

#### Docker Networking and Macvlan

We generally recommend against using macvlan for docker containers however there are a few use cases where macvlans are necessary. One use case is when we want to manage container traffic via source IP based firewall rules on the router (Opnsense). I have a few containers that I do that for, such as bypassing VPN to upload backups to Backblaze B2, or putting certain containers on a different VPN that supports port forwarding (for . . . reasons).

One major drawback of macvlan is that devices or containers on a macvlan can't access or be accessed by the host that is not on the macvlan. That is a kernel restriction not specific to docker, but an annoying issue. To get around it, you can create a virtual interface and route the connections through there as the kernel restriction affects the main interface only.

What I did is, create the following custom interface config and place it in `/etc/network/interfaces.d/`, which not only sets up my server ethernet with a static IP, but it also creates a virtual interface along with a route for it that has a lower metric (higher priority) than the main interface so all lan connections go over the virtual interface. That way, macvlan containers can connect the host and vice versa.

```
    auto lan1
    iface lan1 inet static
        address 192.168.1.50/24
        gateway 192.168.1.1

    auto lan1.10
    iface lan1.10 inet manual

    post-up ip link add vhost0 link lan1 type macvlan mode bridge
    post-up ip link set vhost0 up
    post-up ip route add 192.168.1.0/24 dev lan1 proto kernel scope link src 192.168.1.50 metric 1
    post-up ip route del 192.168.1.0/24 dev lan1 proto kernel scope link src 192.168.1.50
    post-up ip route add 192.168.1.0/24 dev vhost0 proto kernel scope link src 192.168.1.50
    post-up ip link add vhost0.10 link lan1.10 type macvlan mode bridge
    post-up ip link set vhost0.10 up
```

`lan1` is my main ethernet interface and `lan1.10` is the `vlan 10` tagged version. `vhost` is the virtual interface and `vhost0.10` is the `vlan 10` tagged version. The post up arguments delete the default lan route for my `lan1` interface and replace it with one that has `metric 1`, so the `vhost` route for the lan is preferred.

You'll notice that I'm using `lan1` for my main physical interface instead of the detected `enp4s0`. That's because the auto generated name is based on the pci bus and slot ids and unfortunately not static. When I plug and unplug my coral into the pci-e slot, the interface name flip flops between `enp3s0` and `enp4s0`, which is far from ideal when linking virtual interfaces, using static IP rules, and customizing routes.

So I had to resort to creating a permanent name for my main physical interface via a udev rule that matches the name to the interface's MAC address. I created a file at `/etc/udev/rules.d/10-custom-lan1.rules` with the following content:
```
SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*",
ATTR{address}=="<my MAC address>", NAME="lan1"
```
So now my main physical interface is listed as `lan1`, with the non-permanent name listed as `altname enp4s0`.

When I create a macvlan network in docker, I use the parent `vhost0` or `vhost0.10` as appropriate.

```yaml
networks:
  macvlan:
    name: macvlan
    driver: macvlan
    driver_opts:
      parent: vhost0
    ipam:
      driver: default
      config:
       - subnet: 192.168.1.0/24
         gateway: 192.168.1.1
  macvlan.10:
    name: macvlan.10
    driver: macvlan
    driver_opts:
      parent: vhost0.10
    ipam:
      driver: default
      config:
       - subnet: 192.168.2.0/24
         gateway: 192.168.2.1
```

#### Immich Stack

Immich set up involves a stack of 4 containers. I elected to keep it in a separate yaml because updates require comparing the yaml contents to the upstream's recommended yaml (that gets updated with each upstream release) and carefully editing/updating pinned image tags and other arguments.

_Disclaimer: I'm pretty anal retentive about my family photos. I need to keep original copies of photos backed up both locally and remotely and can't stand the idea of losing photos. Perhaps my descendants (if any are into history or are sentimental) will thank me for it, who knows. . ._

Prior to setting up Immich, I would manually transfer photos from the family cell phones to my server via smb. That way I kept a master record of all family photos that would get backed up to Backblaze B2 weekly. But as with any manual operation, the frequency of transfers was not great and I risked losing data between transfers. In fact, I noticed that some photos got corrupted on the device some days or weeks after they were taken (I know they werent't initially corrupt because they were uploaded to Google Photos properly shortly after they were taken). Somehow between the photos being taken (and subsequently uploaded Google) and my manual transfer, something accessed the photos on the device and corrupted them. No idea what, but I wanted to prevent that in the future.

When I set up Immich (as a backup to Google Photos because there is always the possibility of Google making a change that would make it unfeasible for me to continue using it), I wanted to make sure Immich only had read only access to my photos (since it's still considered alpha software and has breaking changes). I added my photos library to Immich as an external library and mounted the path in the Immich container as read-only. For uploads, I set up Nextcloud, which the phones would upload to Nextcloud automatically, and a nightly cron script would copy the photos to my photos library, which Immich would scan and detect shortly after.

Unfortunately Nextcloud's photo sync function turned out to be unreliable. First I noticed it was stripping geotags from photos when uploading because it needed a special Android permission. After assigning that permission, it worked for a couple of months but then broke again. Apparently Google blocked Nextcloud from accessing that permission. That became a deal breaker for me and I had to migrate. It also had other bugs like trying to upload temp files (hidden, starting with a period in their name) and sometimes succeeding but often failing with errors (I don't know which is worse), even though the option in settings to ignore hidden files was checked. [The bug report](https://github.com/nextcloud/android/issues/13943) pending 5 months with no team response (other than adding a couple of tags) and reddit posts about the issue going back much longer.

Hesitantly, I switched to using Immich's Android app to upload photos, which meant that Immich would have write access to all my photos going forward. The compromise I came up with is letting Immich upload to its internal library, then a nightly cron script copies the newly added photos to a separate folder, which I use for long term storage and back up to Backblaze from. So if Immich does something unexpected or unwanted to the photos in the future, I would still have untouched copies in that other folder. Sure, if Immich messes with the photos right after upload and before the nightly cron copy, I'm out of luck, but it's a fairly small risk. I'm more worried about a potential Immich bug that would mass modify older photos, in which case I would be protected.

To sum up, I have the following folders on the server:
- `/mnt/user/Pictures/All`: contains all original photos taken priot to starting with Immich. Backed up weekly to Backblaze B2
- `/mnt/user/Pictures/All-Immich`: contains all original photos uploaded by Immich and copied here nightly. Backed up weekly to Backblaze B2
- `/mnt/user/Immich`: Immich's internal library. All mobile uploaded photos go here.

As you can see there is duplication between the last two paths as the nightly cron script copies photos from one folder to the other. To prevent using double the storage space, I could use hardlinks, but that would mean Immich making changes to one copy would also modify the other copy and that would defeat the purpose. I'm currently looking into ZFS deduplication feature. In the past it was highly recommended against due to really high resource utilization. But ZFS 2.3 apparently introduced a `fast dedup` feature (rather a collection of features and improvements) that reduces the resource utilization and makes the feature more useful. ZFS dedup works at a block level and would prevent double storage fo copied files in different folders. It would also simply duplicate the blocks if Immich ever made a change to its own copy so the copied version remains untouched. I'll have to do some tests before jumping onto it. Debian backports just got support for ZFS 2.3.0 (as of 2025-03-20) so it's brand new.

### VMs

I was quite spoiled by Unraid's VM management interface, which was pretty good. I mainly used two VMs:
- Win11 VM for a dual purpose:
  - Adobe Suite (I don't want to install any Adobe stuff on my main computer because Creative Cloud spreads like a virus with its million background processes that are impossible to disable as it turns into a game of whack-a-mole)
  - Bypassing VPN (my entire internet connection goes over Torguard via Wireguard tunnels for privacy but some websites block me. In those crucial cases, I use the browser of the VM, which bypasses the VPN via firewall rules, which is much easier than modifying firewall rules for my main computer whenever I need to access such a website).
- Hackintosh VM specifically for tracking Airtags that I put in various bags. Apparently you can only track Airtags via an Apple device and not on the web (through iCloud). Thankfully a hackintosh VM counts as an Apple device.

I was really dreading migrating them and using cli kvm tools to manage them. But I found out about [Cockpit](https://cockpit-project.org/) which made it pretty easy to move the VMs over and manage them. Its gui even has a built-in VNC viewer. For the Hackintosh VM I had to manually edit the xml for a couple of things but most other things I could do from the gui. It was as simple as moving the VM disk files over to the new server and importing them while selecting the necessary info like cpu and network devices.

When I need to fire up one of the VMs I just do it from the Cockpit interface. Bonus: It also has a pretty good ZFS module for viewing various stats about storage and snapshots and all that.
