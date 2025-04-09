# homelab
> *(This project is a fork of [trentz's homelab](https://github.com/trentzz/homelab), with custom modifications tailored to my specific requirements and hardware setup.)*

Homelab Adventure! This is intended as a blog or guide, as well as a way to document this project!

If you would like to follow this as a guide, it's structured more or less in the order of what you should do first, but feel free to skip around to the sections you're most interested in.

WIP at the moment. Stay tuned!

- [homelab](#homelab)
  - [Server hardware, VMs and NAS](#server-hardware-vms-and-nas)
    - [Proxmox](#proxmox)
    - [Hardware](#hardware)
    - [Docker](#docker)
    - [Openmediavault](#openmediavault)
  - [Initial networking setup](#initial-networking-setup)
    - [Domain registration](#domain-registration)
    - [Cloudflare](#cloudflare)
    - [Cloudflare tunnel](#cloudflare-tunnel)
  - [Services](#services)
    - [Jellyfin](#jellyin)
    - [NGINX Proxy Manager](#nginx-proxy-manager)
    - [Nextcloud](#nextcloud)

---

## Server hardware, VMs and NAS

### Hardware

- **CPU**: [Intel Core i5-4590](https://www.intel.com/content/www/us/en/products/sku/80815/intel-core-i54590-processor-6m-cache-up-to-3-70-ghz/specifications.html)
- **Motherboard**: [Asus H81M-A Micro ATX LGA1150](https://www.asus.com/us/supportonly/h81m-a/helpdesk_knowledge/)
- **Memory**: 16 GB (2 x 8 GB) DDR3-1600 CL10
- **Storage**: Patriot P210 256 GB SSD (Boot and LVM Drive) | Seagate Ironwolf 10 TB NAS SATA HDD
- **Case**: OEM Micro-ATX case ([specs](https://phoenixtechnologies.es/en/products/caja-de-ordenador-phoenix-micro-atx-lite-s1-formato-slim-con-usb-3-0-fuente-300w-incluida))

> This build is designed to be power-efficient and extremely budget-friendly while delivering solid performance for general server-related tasks.

**Estimated power usage:**
- **Light Use**: ~40W–80W
- **Moderate Workloads**: ~100W–150W
- **Full Load**: ~170W–200W
  
Most of these components, other than storage, were picked up for $10 AUD at a yard sale.  I specifically chose this setup because the CPU supports [Intel Quick Sync](https://en.wikipedia.org/wiki/Intel_Quick_Sync_Video), which is a dedicated hardware encoder/decoder that speeds up transcoding significantly.
The SSD comes from my modded PS3, while the hard drive was an absolute steal from eBay at $125 due to a pricing error.

#### Considerable Hardware Upgrades and Extra Notes

In order of priority:
- **Upgrade stock cooler for better performance on multi-threaded workloads** (plus server would operate with less noise): would have to be a low-profile cooler that is less than 55 mm in height, due to the tight clearance of the Micro-ATX case.
- **2.5 GbE port for higher bandwidth**: only possible PCIe upgrade since motherboard only houses 2 additional PCIe x1 slots (M.2 NVMe adapters are only PCIe x4 and x16, and with x16 M.2 storage is not bootable so the effort isn't worth it)
- **Additional 10 TB HDD storage to enable parity**: would need to remove the Blu-Ray reader compartment to house a second 3.5" HDD
- **CPU max-upgrade for current motherboard setup**: i7 4770k

---

### Proxmox

#### What is proxmox? (from forum.proxmox)

Proxmox is a complete open-source server virtualization management solution. Proxmox offers a web interface accessible after installation on your server which makes management easy, usually only needing a few clicks.

#### Quick setup guide by Khoi

**Requirements**

- A secondary device for downloading necessary files
- An empty USB drive (at least 4 GB)
- A target device for Proxmox installation with an unpartitioned boot drive

**Preparation on a Separate Computer**

1. Download the latest stable ISO from [Proxmox](https://www.proxmox.com/en/downloads)
2. Download [Rufus](https://rufus.ie/en/)
3. Launch Rufus and create a bootable USB drive, selecting the Proxmox ISO as the boot selection.

**Installation on the Target Computer**

1. **Access BIOS**: Restart the target device and enter the BIOS/UEFI menu. Set the boot priority to USB and boot from the installation media.
2. **Install Proxmox**: Follow the on-screen setup instructions. Be sure to record all configuration details for future reference.
3. **Disable Subscription Warning**: After installation, navigate to Datacenter > {Device Name} > Updates > Repositories and add a non-subscription repository to remove the startup subscription prompt.

#### More resources

- [Proxmox Forum Tutorial](https://forum.proxmox.com/threads/proxmox-beginner-tutorial-how-to-set-up-your-first-virtual-machine-on-a-secondary-hard-disk.59559/)
- [XDA Developeres Proxmox Guide](https://www.xda-developers.com/proxmox-guide/)

---

### Hard drive storage

Now that Proxmox is set up, we can configure the storage for our NAS. In my setup, I’m using a single 10TB hard drive in a Single-Disk array, but any storage configuration will work based on your needs.

>If the data is critical, it is highly recommended to use RAID for redundancy and parity, as this helps protect against drive failures.

#### Setting Up a Single-Disk Array on Proxmox VE

**1. Identify Your Disks**

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

---

### Docker

#### What is Docker? (from AWS)

Docker is a software platform that allows you to build, test, and deploy applications quickly. Docker packages software into standardized units called containers that have everything the software needs to run including libraries, system tools, code, and runtime. 

#### Why Docker?

While Proxmox can manage both virtual machines (VMs) and containers, Docker is often a more flexible way to deploy applications, especially for lightweight microservices like NGINX Proxy Manager. Furthermore, I'm trying to get practical experience with tools that are commonly used in the industry, and Docker will allow me to quickly test applications in isolated environments.

#### My Docker Setup

To streamline the setup process, I used [Tteck's Proxmox VE Helper-Scripts](https://tteck.github.io/Proxmox/#docker---kubernetes) to install the LXC (Linux Container) to host Docker. The LXC configuration are as follows:

- **CPU**: 2 cores
- **RAM**: 2 GB
- **Root Disk**: 10 GB
- **Operating System**: Debian 12
- **Installed Software**: Portainer and Docker Compose

Once the container is set up, Docker and Portainer should be accessible via an IP address on your network. This IP may be automatically assigned or manually configured, depending on your setup. To find the assigned IP address, access Docker’s shell and run the following command:
```bash
ip r
```
The IP address will be displayed in the format 192.168.x.x. By default, Portainer can be accessed through its standard port, 9000. You will need to configure your account settings within 5 minutes of the container being active, otherwise Portainer would time out and you'd have to restart the container:
```bash
docker restart portainer
```

---

### OpenMediaVault

> I primarily followed the [tutorial by What's New Andrew](https://www.youtube.com/watch?v=Bce7VT3kJ4g), WITH the exception of the RAID configuration since I already have my ZFS pool setup. If you did NOT setup a zfs pool in Proxmox and prefer a visual, step-by-step guide, I recommend checking out his tutorial.

**Import the OMV (OpenMediaVault) ISO into Proxmox**

This can be done via the Proxmox Web Interface, but how I did it was through the command line. Navigate to
```bash
/var/lib/vz/template/iso/
```
Then use wget or curl to download the ISO directly from OMV. Example below downloads OMV version 7.4.17
```bash
wget https://sourceforge.net/projects/openmediavault/files/iso/7.4.17/openmediavault_7.4.17-amd64.iso
```
You can then verify that the ISO is in Proxmox by navigating back to the Content tab in the Proxmox web interface under your storage.

**Create OMV Virtual Machine**

To create a new virtual machine (VM), click the Create VM button located in the top right corner of the Proxmox web interface. Below are the relevant configuration details I used for my setup (only the OMV specific settings are included):

- **General**
    - Start at boot enabled

- **OS**
    - ISO image: openmediavault_7.4.17-amd64.iso

- **Disks**
    - Bus/Device: SATA
    - Size: 16 GB

- **Cores**: 2

- **Memory**: 4 GB

#### Harddrive passthrough

1. Ensure that the virtual machine (VM) is powered off before proceeding.
2. In the Proxmox shell, execute the blkid command to identify the partition UUID for the desired drives. In this case, the partition sbd1 created during the earlier zpool create command was used.
3. For each drive you wish to mount as a resource on the VM, execute the following command for each drive:

```bash
qm set <VM ID> -sata# /dev/disk/by-partuuid/<Device UUID>
```

Once completed, the drives will be successfully mounted.

#### OS Walkthrough and Setup

- The installation process is straightforward and intuitive. Ensure that you select the correct partition during installation—specifically, the 16 GB partition in the disk partitioning section.

- After completing the installation and rebooting, OpenMediaVault (OMV) should be accessible via the IP address displayed next to the ens18 interface.

The default login credentials are:

Username: admin

Password: openmediavault

- Navigate to the Update Management section in the OMV web interface. If updates are available, you may choose to install them by clicking Install. You may be prompted after to reboot OMV in order to apply these updates.

#### Enabling ZFS Support in OpenMediaVault

Since OpenMediaVault (OMV) does not include native support for ZFS by default, you will need to install the OMV-Extras plugin, which provides access to additional functionality, including ZFS support.

1. Log in to your OMV web interface.

2. Open the OMV console on Proxmox (or via SSH).

3. Run the following command to install OMV-Extras:

```bash
wget -O - https://github.com/OpenMediaVault-Plugin-Developers/packages/raw/master/install | bash
```

4. Once the script completes, return to the OMV web interface.

5. Perform a hard refresh to load the newly available plugins by pressing Ctrl + Shift + R in your browser.

6. Navigate to System > Plugins, locate and install the ZFS plugin.

#### Configuring ZFS and File Sharing

1. In the OMV web interface, navigate to ZFS > Pools. Click on the Tools icon (wrench symbol) and select Import Pool.

2. In the import dialog:
- Check All Missing Pools.
- Enable the Force option — this is necessary if the pool was previously used on another system, to avoid errors such as:
```exit code '1': cannot import 'tank': pool was previously in use from another system.```

3. Once the pool is successfully imported, select it from the list. Click Add to create a new ZFS filesystem. Name the filesystem as desired — call it anything you want. I named it 'shared' and used the default file system settings. After rebooting the VM, the following entries should show up on Storage > File Systems:

![image](https://github.com/user-attachments/assets/abe5de44-0c46-4500-85ce-b19720ed076d)

4. Navigate to Storage > Shared Folders. Click the Create (+) button to add a new shared folder and customize the folder settings according to your preferences.

Once the shared folder is created, you can now make it accessible over the network using one or more of the following protocols, depending on your environment and use case:

- NFS – Ideal for Linux-based systems and hypervisors like Proxmox.

- SMB (CIFS) – Commonly used for Windows-based file sharing.

**NFS**

1. Go to Services > NFS > Settings and enable the service. Then, under Services > NFS > Shares, click Create (+).

2. NFS options configuration:

  - Client: Specify an IP address or subnet allowed to access the share (e.g., 192.168.1.0/24). I used my Proxmox IP as the drive will be mounted from there.

  - Options: Adjust permissions such as read/write and root access as needed. I went with ```rw,sync,no_subtree_check,crossmnt,no_root_squash``` for full permissions.

3. Next, to mount the shared folder on your Proxmox server and ensure it is automatically mounted on startup, edit the /etc/fstab file to include an entry for the NFS share. For example:
```bash
192.168.50.XX:/media /mnt/media/ nfs defaults 0 0
```
5. To mount the share immediately, run ```mount -a``` to mount. Run ```systemctl daemon-reload``` if prompted.

Once the NFS share is mounted on your Proxmox server, you can modify the LXC configuration files to mount the shared folder within your Proxmox containers.

**SMB/CIFS (Windows File Sharing)**

1. In User Management > Users, click Create (+) and customize settings according to your preferences. Just make sure to add 'users' in the group section.

2. Go to Services > SMB/CIFS > Settings and enable the service. Then, under Services > NFS > Shares, click Create (+).

3. SMB configuration:

  - Shared folder: Choose the shared folder created earlier.

  - Public: Choose whether the share is accessible without authentication.

  - Permissions: Set the appropriate access control for users and groups.

4. Once SMB is enabled and configured, the OpenMediaVault server should automatically appear under the Network section in Windows File Explorer — as long as your Windows machine is on the same local network.

5. If it does not appear automatically, you can try to manually access the share by entering the OMV server’s IP address into the File Explorer address bar using the following format ```\\<OMV-IP-ADDRESS>\```

---

## Initial networking setup

### Domain registration

### Cloudflare

### Cloudflare tunnel

## Services

### Jellyfin

#### What is Jellyfin and Why use it?

From jellyfin.org: Jellyfin enables you to collect, manage, and stream your media.

I needed a reliable platform to stream my media collection with accurate metadata, ensuring a seamless experience for my family (mostly me). Having previously used both Jellyfin and Plex on my Windows-based server, I found that Jellyfin more ideal with the service being open-source and customisable.

#### My Jellyfin Setup

I used [Tteck's Proxmox VE Helper-Scripts](https://tteck.github.io/Proxmox/) to install the LXC (Linux Container) to host Jellyfin. While Docker was an option, I opted for LXC due to its advantages in resource efficiency. The LXC configuration are as follows:
- **CPU**: 2 cores
- **RAM**: 2 GB
- **Root Disk**: 8 GB
- **Operating System**: Debian 12

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

#### Adding Your Media

If your drive with the media is already mounted, simply navigate to Dashboard > Libraries and add your media libraries. Wait for the scan to complete, and you'll be able to view your media.

Since I’m running OpenMediaVault as an VM on Proxmox for my NAS solution, the configuration process can be quite complex. Therefore, it's essential to properly set up OpenMediaVault before proceeding with this section.

---

### NGINX Proxy Manager

---

### Nextcloud

