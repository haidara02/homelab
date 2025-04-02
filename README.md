# homelab
(This project was forked from trentz's [homelab](https://github.com/trentzz/homelab), with modifications to my personal needs and available hardware)
Homelab Adventure! This is intended as a blog or guide, as well as a way to document this project!

If you would like to follow this as a guide, it's structured more or less in the order of what you should do first, but feel free to skip around to the sections you're most interested in.

WIP at the moment. Stay tuned!

- [homelab](#homelab)
  - [Server hardware, VMs and NAS](#server-hardware-vms-and-nas)
    - [Proxmox](#proxmox)
    - [Hardware](#hardware)
    - [Docker](#docker)
    - [Jellyfin](#jellyin)
    - [Openmediavault](#openmediavault)
  - [Initial networking setup](#initial-networking-setup)
    - [Domain registration](#domain-registration)
    - [Cloudflare](#cloudflare)
    - [Cloudflare tunnel](#cloudflare-tunnel)
  - [Services](#services)
    - [Nextcloud](#nextcloud)

## Server hardware, VMs and NAS

### Hardware

- CPU: [Intel Core i5-4590](https://www.intel.com/content/www/us/en/products/sku/80815/intel-core-i54590-processor-6m-cache-up-to-3-70-ghz/specifications.html)
- Motherboard: [Asus H81M-A Micro ATX LGA1150](https://www.asus.com/us/supportonly/h81m-a/helpdesk_knowledge/)
- Memory: 16 GB (2 x 8 GB) DDR3-1600 CL10
- Storage: Patriot P210 256 GB SSD | Seagate Ironwolf 10 TB NAS SATA HDD
- Case: OEM Micro-ATX case ([specs](https://phoenixtechnologies.es/en/products/caja-de-ordenador-phoenix-micro-atx-lite-s1-formato-slim-con-usb-3-0-fuente-300w-incluida))
This build is designed to be power-efficient and extremely budget-friendly while delivering solid performance for general server-related tasks.
**Estimated power usage:**
- Light Use: ~40W–80W
- Moderate Workloads: ~100W–150W
- Full Load: ~170W–200W
Most of these components, other than storage, were picked up for $10 AUD at a yard sale.  I specifically chose this setup because the CPU supports [Intel Quick Sync](https://en.wikipedia.org/wiki/Intel_Quick_Sync_Video), which is a dedicated hardware encoder/decoder that speeds up transcoding significantly.
The SSD comes from my modded PS3, while the hard drive was an absolute steal from eBay at $125 due to a pricing error.

#### Considerable Hardware Upgrades and Extra Notes

In order of priority:
- Upgrade stock cooler for better performance on multi-threaded workloads (plus server would operate with less noise): would have to be a low-profile cooler that is less than 55 mm in height, due to the tight clearance of the Micro-ATX case.
- 2.5 GbE port for higher bandwidth: only possible PCIe upgrade since motherboard only houses 2 additional PCIe x1 slots (M.2 NVMe adapters are only PCIe x4 and x16, and with x16 M.2 storage is not bootable so the effort isn't worth it)
- Additional 10 TB HDD storage to enable parity: would need to remove the Blu-Ray reader compartment to house a second 3.5" HDD
- CPU max-upgrade for current motherboard setup: i7 4770k

### Proxmox

#### What is proxmox? (from forum.proxmox)

Proxmox is a complete open-source server virtualization management solution. Proxmox offers a web interface accessible after installation on your server which makes management easy, usually only needing a few clicks.

### Quick setup guide by Khoi

#### Requirements
- A secondary device for downloading necessary files
- An empty USB drive (at least 4 GB)
- A target device for Proxmox installation with an unpartitioned boot drive
#### Preparation on a Separate Computer
1. Download the latest stable ISO from [Proxmox](https://www.proxmox.com/en/downloads)
2. Download [Rufus](https://rufus.ie/en/)
3. Launch Rufus and create a bootable USB drive, selecting the Proxmox ISO as the boot selection.
#### Installation on the Target Computer
1. Access BIOS – Restart the target device and enter the BIOS/UEFI menu. Set the boot priority to USB and boot from the installation media.
2. Install Proxmox – Follow the on-screen setup instructions. Be sure to record all configuration details for future reference.
3. (Optional) Disable Subscription Warning – After installation, navigate to Datacenter > {Device Name} > Updates > Repositories and add a non-subscription repository to remove the startup subscription prompt.

#### More resources

- [Proxmox Forum Tutorial](https://forum.proxmox.com/threads/proxmox-beginner-tutorial-how-to-set-up-your-first-virtual-machine-on-a-secondary-hard-disk.59559/)
- [XDA Developeres Proxmox Guide](https://www.xda-developers.com/proxmox-guide/)

### Hard drive storage

Now that Proxmox is set up, we can configure the storage for our NAS. In my setup, I’m using a single 10TB hard drive in a Single-Disk array, but any storage configuration will work based on your needs.

If the data is critical, it is highly recommended to use RAID for redundancy and parity, as this helps protect against drive failures.

#### Setting Up a Single-Disk Array on Proxmox VE

##### 1. Identify Your Disks

Before proceeding, determine the device names of your drives:

```bash
lsblk
```

You'll see a list of drives. If your drives are `/dev/sdb`, `/dev/sdc`, `/dev/sdd`, and `/dev/sde`, these will be used in the array.

**2. Wipe Existing Partitions (Optional but Recommended)**

If the drives were used previously, clean them:

```bash
for disk in /dev/sdb /dev/sdc /dev/sdd /dev/sde; do
    wipefs -a "$disk"
    dd if=/dev/zero of="$disk" bs=1M count=100
done
```

**3. Create the ZFS Pool**

In my setup, I ran the following command to create a ZFS pool named `tank` with a single disk:
```bash
zpool create -f tank /dev/sdb
```

Alternatively, if you have a RAID setup, run the following command to create a RAID-Z1 pool named `tank`:

```bash
zpool create -f tank raidz1 /dev/sdb /dev/sdc /dev/sdd /dev/sde
```

**4. Verify the Pool**

Check the status of your new ZFS pool:

```bash
zpool status
```

**5. Enable Compression (Recommended)**

ZFS has built-in compression that improves performance and saves space:

```bash
zfs set compression=lz4 tank
```

**6. Enable Deduplication (Optional)**

Deduplication saves space but requires significant RAM:

```bash
zfs set dedup=on tank
```

**7. Enable Automatic Scrubbing**

To maintain data integrity, schedule a scrub:

```bash
echo "0 3 * * 0 root zpool scrub tank" >> /etc/crontab
```

This runs a scrub every Sunday at 3 AM.

**8. Mount the ZFS Pool**

By default, ZFS mounts pools under `/tank`. You can verify this with:

```bash
zfs list
```

#### Future Upgrades

If you initially set up a single-disk ZFS pool and later want redundancy with multiple disks, you have two main options:

**1. Convert to a Mirrored Pool (RAID 1 Equivalent)**
If you want redundancy, you can convert your single-disk pool into a ZFS mirror:
```bash
zpool attach tank /dev/sdb /dev/sdc
```
This converts your single-disk pool (/dev/sdb) into a mirrored setup with /dev/sdc. However, this does not provide parity (like RAID-Z1) but instead mirrors the data.

**2. Create a New RAID-Z1 Pool (Requires 3 Disks and Data Migration)**

Unfortunately, ZFS does not allow in-place conversion from a single-disk pool to RAID-Z1. The best approach is:

1. Back up your data from tank.
2. Destroy the existing pool:
```bash
zpool destroy tank
```
3. Create a new RAID-Z1 pool with the new drive(s)
```bash
zpool create -f tank raidz1 /dev/sdb /dev/sdc
```

### Docker

#### What is Docker? (from AWS)

Docker is a software platform that allows you to build, test, and deploy applications quickly. Docker packages software into standardized units called containers that have everything the software needs to run including libraries, system tools, code, and runtime. 

#### Why Docker?

While Proxmox can manage both virtual machines (VMs) and containers, Docker is often a more flexible way to deploy applications, especially for lightweight microservices like NGINX Proxy Manager. Furthermore, I'm trying to get practical experience with tools that are commonly used in the industry, and Docker will allow me to quickly test applications in isolated environments.

#### My Docker Setup

To streamline the setup process, I used [Tteck's Proxmox VE Helper-Scripts](https://tteck.github.io/Proxmox/#docker---kubernetes) to install the LXC (Linux Container) to host Docker. The LXC configuration are as follows:
- CPU: 2 cores
- RAM: 2 GB
- Root Disk: 10 GB
- Operating System: Debian 12
- Installed Software: Portainer and Docker Compose

Once the container is set up, Docker and Portainer should be accessible via an IP address on your network. This IP may be automatically assigned or manually configured, depending on your setup. To find the assigned IP address, access Docker’s shell and run the following command:
```bash
ip r
```
The IP address will be displayed in the format 192.168.x.x. By default, Portainer can be accessed through its standard port, 9000. You will need to configure your account settings within 5 minutes of the container being active, otherwise Portainer would time out and you'd have to restart the container:
```bash
docker restart portainer
```

### Jellyfin

#### What is Jellyfin and Why use it?

From jellyfin.org: Jellyfin enables you to collect, manage, and stream your media.

I needed a reliable platform to stream my media collection with accurate metadata, ensuring a seamless experience for my family (mostly me). Having previously used both Jellyfin and Plex on my Windows-based server, I found that Jellyfin more ideal with the service being open-source and customisable.

#### My Jellyfin Setup

I used [Tteck's Proxmox VE Helper-Scripts](https://tteck.github.io/Proxmox/) to install the LXC (Linux Container) to host Docker. The LXC configuration are as follows:
- CPU: 2 cores
- RAM: 2 GB
- Root Disk: 8 GB
- Operating System: Debian 12

Once the container is set up, Jellyfin should be accessible via an IP address on your network. This IP may be automatically assigned or manually configured, depending on your setup. To find the assigned IP address, access Docker’s shell and run the following command:
```bash
ip r
```
The IP address will be displayed in the format 192.168.x.x. By default, Jellyfin can be accessed through its standard port, 8096. Here you will configure your Admin account to access the dashboard.

#### Enabling Transcoding

1. Navigate to Dashboard > Playback > Transcoding
2. Set the hardware acceleration to Intel QuickSync and select the checkboxes next to the video compression standards for which you wish to enable hardware decoding
3. Select the checkbox for 'Enable hardware encoding'
4. (Optional) Navigate to Trickplay and enable hardware decoding to significantly speed up the trickplay generation process. Trickplays are the thumbnails you see when you hover your mouse over the timeline.

#### Useful Plugins

While Jellyfin is fairly minimalistic by default, you can significantly enhance its functionality by installing plugins. Jellyfin already offers a range of plugins available through its catalog, but in this list, I will also highlight some external plugins that can further enrich your experience. These additions can bring your setup closer to the features found in professional streaming services (which became necessary as my parents frequently exclaimed that it didn’t resemble Netflix enough).

**Plugins**

- **Chapter Segment Provider**: Enables chapter markers for your media, allowing for easier navigation, especially for longer movies or TV shows.
  
- **DLNA**: Adds support for streaming to DLNA-compatible devices, making it easy to cast media to other devices on your network, such as smart TVs or game consoles.
  
- [**Editor's Choice**](https://github.com/lachlandcp/jellyfin-editors-choice-plugin/tree/main/EditorsChoicePlugin): Provides a curated list of content, which can help to highlight popular or recommended shows and movies, giving users a more streamlined browsing experience.
  
- [**InPlayerEpisode Preview**](https://github.com/Namo2/InPlayerEpisodePreview): Adds episode previews directly into the Jellyfin interface, similar to how Netflix shows trailers or previews of upcoming episodes.
  
- [**Intro Skipper**](https://github.com/intro-skipper/intro-skipper): Replaces the old Intro Skipper, automatically skipping intros for TV shows using audio fingerprint detection.
  
- **Open Subtitles**: Integrates OpenSubtitles.org with Jellyfin, making it easier to automatically fetch and display subtitles for your media. Make sure to log in to your OpenSubtitles account to be able to use this plugin.
  
- **Session Cleaner**: Automatically cleans up idle sessions to free up resources, ensuring your server runs efficiently and preventing unnecessary resource consumption from inactive users.
  
- **TheTVDB**: Provides accurate metadata and episode information for TV shows, ensuring that your library remains well-organized with proper details for each series.
  
- **Trakt**: Syncs your Jellyfin library with your **Trakt** account, allowing you to track watched content across platforms, manage watchlists, and discover new shows.

### NGINX Proxy Manager

### Openmediavault

For most of the services we're going to setup, we want to have a dedicated or shared "data" folder that they use. For example, where all the nextcloud data goes, where your gitea remote repositories live, etc. There are a few ways to enable that, but here we're going to first make a NAS VM and use "shared folders" and nfs connections to our other services. This way, our storage is more or less managed in one central location.


## Initial networking setup

### Domain registration

### Cloudflare

### Cloudflare tunnel

## Services

### Nextcloud

