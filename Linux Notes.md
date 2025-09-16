# Linux LVM
LVM stands for **Logical Volume Manager**.  
It’s a **flexible disk management system** in Linux that allows you to manage storage more dynamically compared to traditional partitioning.

Instead of dealing directly with physical disks and partitions, LVM adds a **layer of abstraction**, making it easier to resize, add, or remove storage **without downtime** (in most cases).

## 🔹 Key Components of LVM

1. **Physical Volume (PV)**
    - Actual hard disks or partitions (`/dev/sda1`, `/dev/sdb`, etc.) initialized for use by LVM.
    - Command: `pvcreate /dev/sdb`
2. **Volume Group (VG)**
    - Pool of storage created by combining multiple PVs.
    - Think of it as a big storage tank.
    - Command: `vgcreate myvg /dev/sdb /dev/sdc`
3. **Logical Volume (LV)*
    - A "virtual partition" carved out from the VG.
    - This is what gets mounted and used by the filesystem (`/opt`, `/home`, etc.).
    - Command: `lvcreate -L 10G -n mylv myvg`, `lvcreate -l 100%FREE -n mylv myvg
## 🔹 Advantages of LVM
✅ **Flexible resizing** – You can extend or shrink logical volumes without repartitioning.  
✅ **Online storage management** – Extend file systems while they are mounted (with `xfs_growfs` or `resize2fs`).  
✅ **Combining multiple disks** – Create one large storage pool from multiple small disks.  
✅ **Snapshots** – Take point-in-time copies of volumes for backup/testing.  
✅ **RAID-like features** (striping, mirroring) are supported with LVM2.
## Common LVM Commands

- `pvcreate /dev/sdb` → Create physical volume
- `vgcreate vg1 /dev/sdb /dev/sdc` → Create [^7]volume group
- `lvcreate -L 20G -n lv1 vg1` → Create logical volume
- `lvextend -L +10G /dev/vg1/lv1` → Extend LV
- `xfs_growfs /mountpoint` or `resize2fs /mountpoint` → Grow filesystem
- `lvremove /dev/vg1/lv1` → Remove LV
- `vgextend` / `vgreduce` → Add or remove PVs from a VG

## When to Use LVM?

- In environments where **storage requirements change frequently**.
- On servers, especially for **databases, applications, and virtualization**, where you need flexible storage.

## How to scan disks?
In Linux, when you add a new disk (physical or virtual), the system might not detect it automatically. You need to **rescan the SCSI bus** to make the OS see the new disk.
Here are the common ways:
 🔹 1. Rescan All SCSI Hosts
`echo "- - -" > /sys/class/scsi_host/hostX/scan`
- Replace `hostX` with the actual host adapter (like `host0`, `host1`, etc.).
- `- - -` means:
    - `-` → all channels
    - `-` → all SCSI IDs
    - `-` → all LUNs
To rescan all:
`for host in /sys/class/scsi_host/host*; do     
    echo "- - -" > $host/scan
done`
🔹 2. Using `rescan-scsi-bus.sh`
In **Linux**, the **`sg3_utils`** package is a collection of command-line utilities for working with devices that use the **SCSI (Small Computer System Interface) command set**.
If the `sg3_utils` package is installed, you can use:
`rescan-scsi-bus.sh`
 🔹 3. Verify New Disks
After rescanning, check available disks:
`lsblk, fdisk -l`
## What happens during SCSI scan?
When you run:
`echo "- - -" > /sys/class/scsi_host/hostX/scan`
You are telling the **kernel SCSI subsystem**:  
👉 “Hey, go rescan this host adapter for new devices across all channels, targets, and LUNs.”
- First `-` = check all channels 
- Second `-` = check all targets (device IDs)
- Third `-` = check all LUNs
The kernel then queries the storage system → “Are there any new disks/LUNs?”  
If yes, it creates new device files like `/dev/sdb`, `/dev/sdc`, etc.
🔹 Why do we need to rescan?
- When storage admins **add a new LUN** from SAN, the OS doesn’t know about it immediately.
- Rescanning the **SCSI bus** forces Linux to detect the new device without rebooting. 

✅ **Short Answer:**  
_"SCSI bus is the communication channel between the host and storage devices. When we run `echo "- - -" > /sys/class/scsi_host/hostX/scan`, we are asking the kernel to rescan that bus for new devices (new disks or LUNs). This makes the OS detect newly added storage without requiring a reboot."_

### Fstab entries

| Filesystem / Network share / UUID / Label      | Mount Point | Type | Options (how fs is mounted) | dump(0/1)<br>No Backup or Backup | Pass (0/1/2)<br>checked by fsck at boot time |
| ---------------------------------------------- | ----------- | ---- | --------------------------- | -------------------------------- | -------------------------------------------- |
| UUID=1a2b3c4d-xxxx-xxxx-xxxx-123456789abc <br> | /           | ext4 | defaults                    | 1                                | 1 (check by fsck on priority)                |
| UUID=5e6f7g8h-xxxx-xxxx-xxxx-abcdef123456      | /home       | xfs  | defaults                    | 1                                | 2 (checked after reboot)                     |
| /dev/sdb1                                      | data        | ext4 | rw,noexec                   | 0                                | 2                                            |
| /dev/sdc1                                      | none        | swap | sw                          | 0                                | 0 (skip)                                     |
|                                                |             |      |                             |                                  |                                              |



# Multipathing in Linux

## What is Multipathing?

- Multipathing provides **multiple physical paths** between the **server (host)** and **storage (SAN LUNs)**.
- These paths can go through **different HBAs (Host Bus Adapters)**, switches, and storage controllers.
- The goal is:
    - ✅ **High availability (HA):** if one path fails, I/O continues on the other path(s).
    - ✅ **Load balancing:** distribute traffic across multiple paths.

Example:
- Server → HBA1 → Switch A → Storage Controller A → LUN
- Server → HBA2 → Switch B → Storage Controller B → Same LUN

Both are different physical routes to the same disk (LUN).

---

## 🔹 Multipath Setup in Linux

### 1. **Install multipath tools**
`yum install device-mapper-multipath -y    # RHEL/CentOS 
`apt install multipath-tools -y            # Ubuntu/Debian`
### 2. **Enable multipath service**
`mpathconf --enable --with_multipathd
 `systemctl enable multipathd
`systemctl start multipathd`
### 3. **Check configuration file**

Default: `/etc/multipath.conf`  
Usually you don’t need to configure much unless you want custom aliases for LUNs.

---

## 🔹 Adding a New LUN (with multipath)

1. **Storage admin maps a LUN** to the server.
2. **Rescan SCSI bus** to detect new disk:
    `echo "- - -" > /sys/class/scsi_host/hostX/scan`
    (or use `rescan-scsi-bus.sh`)
3. **Check new disk**:
    `lsblk fdisk -l`
    You may see the same LUN multiple times (`/dev/sdb`, `/dev/sdc`, `/dev/sdd` …).
4. **Check multipath**:
    `multipath -ll`
    Output will show one **multipath device** (like `/dev/mapper/mpathb`) with multiple underlying paths (`/dev/sdb`, `/dev/sdc`, etc.).
    

---

## 🔹 Validating Multipath

- Check daemon status:    
    `systemctl status multipathd`
- Check multipath devices:
    `multipath -ll`
    Example output:
    ``mpathb (360060e801234abcd567890000000001) dm-2 EMC,Symmetrix size=50G features='1 queue_if_no_path' hwhandler='1 emc' wp=rw |-+- policy='round-robin 0' prio=50 status=active | `- 0:0:0:1 sdb 8:16 active ready running `-+- policy='round-robin 0' prio=10 status=enabled   `- 1:0:0:1 sdc 8:32 active ready running``
    
- Here:    
    - `mpathb` = multipath device (use `/dev/mapper/mpathb`)
    - `sdb`, `sdc` = underlying physical paths

---

## 🔹 Multipath vs PowerPath

| Feature         | Multipath (Linux native)                                      | PowerPath (EMC proprietary)                      |
| --------------- | ------------------------------------------------------------- | ------------------------------------------------ |
| **Vendor**      | Open-source (device-mapper-multipath)                         | EMC (now Dell EMC)                               |
| **Support**     | Works with many SAN vendors (EMC, NetApp, Hitachi, IBM, etc.) | Optimized for EMC storage                        |
| **Cost**        | Free (part of Linux)                                          | Requires license                                 |
| **Performance** | Generic load balancing + failover                             | Better performance tuning, EMC-specific features |
| **Management**  | Commands: `multipath`, `multipathd`                           | Commands: `powermt`, `powermig`                  |

👉 In short:

- **Multipath** = universal solution, free, built into Linux.
- **PowerPath** = commercial EMC solution with vendor-optimized features.

---

### 🔹 Quick Interview Pointers

✅ _What is multipathing?_  
“It provides multiple physical paths between the server and storage for high availability and load balancing.”

✅ _How to enable multipath?_  
“Install `device-mapper-multipath`, enable `multipathd`, configure `/etc/multipath.conf`, and verify with `multipath -ll`.”

✅ _How do you validate a new LUN with multipath?_  
“Rescan SCSI bus, check with `lsblk`, then confirm a single multipath device is created using `multipath -ll`.”

✅ _Difference between Multipath and PowerPath?_  
“Multipath is Linux native and vendor-agnostic; PowerPath is EMC’s licensed solution optimized for EMC arrays with more advanced features.”
👌 Here’s a **practical step-by-step flow** (with commands) 
## 🔹 Step-by-Step: Adding a New LUN with Multipath
### **1. Storage Team Action**
- Storage admin maps a new LUN to your Linux server (via SAN zoning and masking).
### **2. Rescan SCSI Bus on the Linux Server**
`for host in /sys/class/scsi_host/host*; do     echo "- - -" > $host/scan done`
👉 This makes the OS detect new disks (without reboot).
### **3. Verify New Disk**
`lsblk fdisk -l | grep Disk`
👉 You should see a new device like `/dev/sdb` or `/dev/sdc`.  
(Sometimes you’ll see the **same LUN multiple times** via different paths).
### **4. Check Multipath Daemon**
`systemctl status multipathd`
If not running, enable it:
`mpathconf --enable --with_multipathd` 
`systemctl enable --now multipathd`
### **5. Discover Multipath Devices**
`multipath -ll`
👉 You’ll see output like:
``mpathb (360060e801234abcd567890000000001) dm-2 EMC,Symmetrix size=50G features='1 queue_if_no_path' hwhandler='1 emc' wp=rw |-+- policy='round-robin 0' prio=50 status=active | `- 0:0:0:1 sdb 8:16 active ready running `-+- policy='round-robin 0' prio=10 status=enabled   `- 1:0:0:1 sdc 8:32 active ready running``
- `mpathb` = single multipath device created for LUN 
- `/dev/sdb` and `/dev/sdc` = underlying physical paths
### **6. Verify Multipath Device File**
Check under `/dev/mapper/`:
`ls -l /dev/mapper/`
👉 You’ll see something like `/dev/mapper/mpathb`.  
That’s the device you should use (instead of raw `/dev/sdb`, `/dev/sdc`).
### **7. Partition & Create Filesystem**
Example: create a partition, then format with XFS:
`parted /dev/mapper/mpathb mklabel gpt parted /dev/mapper/mpathb mkpart primary 0% 100% mkfs.xfs /dev/mapper/mpathbp1`
### **8. Mount the LUN**
`mkdir /data`
`mount /dev/mapper/mpathbp1 /data`
👉 Add to `/etc/fstab` for persistence:
`/dev/mapper/mpathbp1   /data   xfs   defaults   0 0`
## 🔹 Quick Recap for Interview Answer

> “When a new LUN is added, I first rescan the SCSI bus to detect it, then check the new device with `lsblk`. I make sure `multipathd` is enabled, and confirm with `multipath -ll` that a multipath device is created under `/dev/mapper/`. Then I partition it, create a filesystem, and mount it for use. This way, the server uses the multipath device instead of raw disks, ensuring high availability and load balancing.”

---
🙌 A **diagram/whiteboard-style explanation** of **Multipathing** makes a strong impression in interviews. Here’s how you can draw and explain it step by step:
## 🔹 Multipathing Diagram (Concept)
`        ┌───────────────┐
        │   Linux Host  │
        │               │
        │   HBA1   HBA2 │
        └─────┬─────┬───┘
              │     │
         Path A│     │Path B
              │     │
      ┌───────┴─────┴────────┐
      │     SAN Switches      │
      └───────┬─────┬────────┘
              │     │
       ┌──────┘     └──────┐
       │                   │
┌──────┴──────┐     ┌──────┴──────┐
│ Storage Ctl A│     │ Storage Ctl B│
└──────┬──────┘     └──────┬──────┘
       │                   │
       └──────┬───────┬────┘
              │       │
            ┌─┴───────┴─┐
            │   LUN 01   │
            └────────────┘
`

## 🔹 How to Explain While Drawing

1. **Linux Host**
     - Say: “This is my Linux server with two HBAs (Host Bus Adapters).”
    - Each HBA has a separate connection to the SAN.
2. **SAN Fabric**    
    - Say: “Each HBA connects to a SAN switch (sometimes two for redundancy).”
    - This ensures there’s no single point of failure.
3. **Storage Controllers**
    - Say: “On the storage side, there are two controllers (Controller A & B).”
    - Both present the same LUN (logical unit / disk) to the host.
4. **Multiple Paths**
    - Draw arrows from HBA1 → Switch → Controller A → LUN.
    - Then from HBA2 → Switch → Controller B → LUN.
    - Say: “So the host sees multiple paths to the same disk.”
5. **Multipathing Software**
    - Write: `multipathd` inside the Linux host box.
    - Say: “Multipath merges all these physical paths into one logical device (`/dev/mapper/mpathX`).”

## 🔹 Short Interview Pitch While Pointing at Diagram

> “Here’s my server with two HBAs, each connected to separate SAN switches and storage controllers. The storage presents the same LUN through multiple routes. Without multipath, the OS would see this LUN multiple times as `/dev/sdb`, `/dev/sdc`, etc. With multipath enabled, all these routes are combined into one logical device (`/dev/mapper/mpathb`). This gives high availability (if one path fails, I/O continues on the other) and load balancing across paths.”

---
## 🔹 Multipathing Interview Q&A

### **1. What is multipathing?**

👉 Multipathing provides multiple physical paths between a server and storage (SAN LUN). It ensures **high availability** (failover if one path goes down) and **load balancing** (traffic across multiple paths).
### **2. Why do we need multipathing?**
- High Availability (path redundancy)
- Load Balancing (performance improvement)
- Better utilization of SAN infrastructure
### **3. How does the server see a new LUN without multipath?**

👉 It sees the same LUN multiple times (e.g., `/dev/sdb`, `/dev/sdc`, `/dev/sdd`), one per path. Multipath merges them into a single device (`/dev/mapper/mpathX`).
### **4. What is `multipathd`?**

👉 It’s the multipath daemon that:
- Continuously monitors paths.
- Handles path failover.
- Maintains the mapping of LUNs to multipath devices.
### **5. How do you enable multipathing in Linux?**

`yum install device-mapper-multipath -y`
`mpathconf --enable --with_multipathd`
`systemctl enable --now multipathd`

Then verify with:
`multipath -ll`
### **6. How do you validate multipath is working?**
- Check multipath devices:
    `multipath -ll`
- Check daemon: 
    `systemctl status multipathd`
- Verify `/dev/mapper/` entries:
    `ls -l /dev/mapper/`
### **7. What happens if one path fails?**
👉 Multipath will detect the failure and automatically reroute I/O through the remaining active paths.
### **8. How do you add a new LUN to a server with multipath enabled?**

1. Storage admin maps the LUN.
2. Rescan SCSI bus:
    `echo "- - -" > /sys/class/scsi_host/hostX/scan`
3. Verify new device with `lsblk`.
4. Check multipath binding with `multipath -ll`.
5. Partition, format, and mount using `/dev/mapper/mpathX`.

### **9. What is the difference between Multipath and PowerPath?**

- **Multipath**: Linux native, free, vendor-agnostic.
- **PowerPath**: EMC proprietary, licensed, with optimized performance/features for EMC storage.
### **10. How do you identify which physical paths belong to a multipath device?**
👉 Use:
`multipath -ll`
It shows multipath device + underlying physical devices (e.g., `sdb`, `sdc`).
### **11. Where is the multipath configuration stored?**

👉 `/etc/multipath.conf`

### **12. What policies are used for path selection?**

- **round-robin** → I/O distributed evenly.
- **failover** → Use one path until failure, then switch.
- **multibus** → All paths used simultaneously.
### **13. How do you remove/unmap a LUN safely in multipath?**

1. Unmount filesystem.
2. Remove device from multipath:
    `multipath -f mpathX`    
3. Rescan SCSI to remove stale devices.
### **14. How to check if multipath is using all paths for I/O?**

👉 Use:
`multipath -ll iostat -x -d 1`
You’ll see I/O distributed across paths if load balancing is active.

### 🔹 Interview Tip

If they ask you:  
**“What’s the difference between multipath and a RAID solution?”**  
👉 Answer:

- Multipath = redundancy between **server and storage** (path-level).
- RAID = redundancy **inside storage** (disk-level).  
    Both are complementary, not replacements.
## 🔹 Scenario: One Path Failure in Multipathing

### **1. Initial Check – Multipath Status**

Before failure:
`multipath -ll`

Example output:
``mpathb (360060e801234abcd567890000000001) dm-2 EMC,Symmetrix size=50G features='1 queue_if_no_path' hwhandler='1 emc' wp=rw |-+- policy='round-robin 0' prio=50 status=active | `- 0:0:0:1 sdb 8:16 active ready running `-+- policy='round-robin 0' prio=10 status=enabled   `- 1:0:0:1 sdc 8:32 active ready running``

👉 Here, `sdb` and `sdc` are two **active paths**.

### **2. Simulate Path Failure**
- Physically: unplug one HBA cable, or
- Ask storage team to disable one path (masking/zoning).
### **3. Check Logs**

Immediately check system logs:
`dmesg | grep -i multipath journalctl -xe | grep multipath`
👉 You’ll see messages about path failure:
`device-mapper: multipath: Failing path 8:32.`

### **4. Check Multipath Status Again**

`multipath -ll`
Output after failure:
``mpathb (360060e801234abcd567890000000001) dm-2 EMC,Symmetrix size=50G features='1 queue_if_no_path' hwhandler='1 emc' wp=rw |-+- policy='round-robin 0' prio=50 status=active | `- 0:0:0:1 sdb 8:16 active ready running `-+- policy='round-robin 0' prio=10 status=failed   `- 1:0:0:1 sdc 8:32 failed faulty offline``

👉 Notice how one path is now marked **failed**, but I/O continues on the surviving path.
### **5. Verify Application / Filesystem I/O**
`df -h /data 
`touch /data/testfile`
👉 Filesystem remains available (thanks to multipath).
### **6. Restore the Path**

- Reconnect HBA cable / storage team re-enables the path.
- Check again:
`multipath -ll`
👉 The path should change from **failed** → **active ready running**.

## 🔹 Interview-Style Answer

> “If one HBA cable or SAN switch path fails, multipath detects it automatically. The failed path will show as _faulty_ or _failed_ in `multipath -ll`, but the LUN remains available via surviving paths. I can confirm this by checking logs with `dmesg` or `journalctl`, and filesystem I/O continues without disruption. Once the path is restored, multipath automatically reactivates it.”

---
#  What is NTP?
**NTP = Network Time Protocol**
- It’s a protocol used to **synchronize the time of computers** over a network.    
- Ensures all systems (servers, databases, applications, network devices) have the **same accurate time**.
### 🔹 Why is NTP important?
- **Logs correlation** → If servers have mismatched time, troubleshooting becomes difficult.
- **Security** → Kerberos, certificates, and authentication depend on accurate time.
- **Databases / Clusters** → Time mismatches can cause data inconsistency.
- **Financial transactions** → Require precise timestamps.
### 🔹 How NTP Works
1. Your Linux server (NTP client) connects to an **NTP server**.
2. The NTP server gets accurate time from an **authoritative source** (like an atomic clock or GPS).
3. The server adjusts its clock gradually (not suddenly) to stay in sync.
### 🔹 NTP Configuration in Linux
### 1. Install NTP
`yum install ntp -y       # RHEL/CentOS 
`apt install ntp -y       # Ubuntu/Debian`
### 2. Configure NTP servers in:
- `/etc/ntp.conf` (traditional) 
- Or in newer systems: `/etc/chrony.conf` (Chrony is preferred replacement).
Example (Chrony):
`server 0.pool.ntp.org iburst server 1.pool.ntp.org iburst`
### 3. Start & enable service
`systemctl enable --now chronyd   # modern 
`systems systemctl enable --now ntpd      # older systems`
### 🔹 Validating NTP

- Check status:
`timedatectl`
- With Chrony:
`chronyc sources -v`
- With NTP:
`ntpq -p`

### Checking Stratum of NTP server
## If your server uses **ntpd**
`ntpq -p`
✅ Sample output:
```
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
*time1.example.c .GPS.            1 u   45   64  377    1.234   -0.123   0.456
+time2.example.c 192.168.1.1      2 u   42   64  377    2.345   -0.234   0.567
```
- `st` column = **stratum**
    - Here, `time1.example.c` is stratum **1** (directly from GPS).
    - `time2.example.c` is stratum **2** (syncing from another server).
You can also run:
`ntpq -c rv | grep stratum`
Output:
`stratum=2`
## 🔹 If your server uses **chrony**
`chronyc tracking`
✅ Sample output:
```
Reference ID    : C0A80101 (192.168.1.1)
Stratum         : 3
Ref time (UTC)  : Sat Sep 13 07:20:45 2025
System time     : 0.000000005 seconds fast of NTP time
```
- Here the server is at **stratum 3**.
- Stratum shows **how far the server is from the reference clock**.
- **Stratum 16** = unsynchronized.
- Always try to sync to a **lower stratum** (1 or 2) for better accuracy.
- In real setups, servers are usually **stratum 2 or 3**, syncing from reliable stratum 1 sources.
### 🔹 Interview Short Answer
> “NTP stands for Network Time Protocol. It synchronizes the system clock of a server with a reliable time source. This ensures all servers in an environment have consistent and accurate time, which is critical for logs, authentication, and transactions. In modern Linux, `chrony` is often used instead of the old `ntpd` service.”
### Q n A
**1. One NTP server not syncing while others are**  
This usually happens due to firewall blocks on UDP/123, wrong configuration, or if its stratum is too high. I would first test network connectivity, review `/etc/ntp.conf`, check logs, and restart the service.

**2. A→B→C→D chain, D not in sync**  
If D is not syncing, it usually means C is unreachable or advertising an invalid/high stratum. I’d check connectivity, config, and also suggest avoiding chaining by syncing all servers directly to reliable sources.

**3. Stratum increased, not syncing**  
A rising stratum means the server cannot reach its upstream source. I’d verify the upstream is online and reachable, then check the server’s configuration and logs.

**4. Identify if NTP not working**  
The quickest way is `ntpq -p` or `chronyc sources` — if no peer has a `*` (selected source) or reachability is 0, NTP isn’t working. I’d also compare system time with an external clock.

**5. ntpq -p reachability = 0**  
It means the server cannot reach its upstream NTP host. The fix is to check firewall rules, service status, DNS resolution, and connectivity.

**6. Server time drifting**  
I’d stop the NTP service, manually sync once using `ntpdate` or `chronyc makestep`, then restart NTP/Chrony for continuous sync. This resets the time and ensures it stays accurate.

**7. Good practice: chaining?**  
Chaining isn’t recommended because one failure cascades downstream. Best practice is syncing all servers directly with external/public stratum 1/2 servers.

**8. Primary server fails in chain**  
If the primary fails, all downstream servers will lose sync or jump to higher stratum. This leads to time drift and inconsistency in the environment.

**9. Client-server vs peer**  
In client-server, clients just pull time from a server. In peer setup, servers exchange time with each other for redundancy and better accuracy.

**10. Best way with 4 servers**  
I’d peer all four servers together and also connect them to multiple external NTP sources. That way, if one fails, the others still maintain accurate time.

**11. Data center design with 4 servers**  
The best design is to peer the four servers, have each talk to reliable external stratum 1/2 servers, and configure all internal clients to use all four for redundancy.

**12. Best practice: chain vs peer**  
Peering with a common set of external sources is best — it prevents dependency and single-point failures. Chaining should be avoided in production setups.

**13. Ensure cluster consistent time**  
I’d configure all nodes in the cluster with the same set of NTP servers and ensure each has multiple sources. Monitoring `ntpq` or `chronyc tracking` ensures consistency across the cluster.
# Application Performance

### 🔹 Step 1: Performance Areas to Monitor
When ensuring application performance, you must check the **4 golden resources**:
1. **CPU**
2. **Memory (RAM)**
3. **Disk I/O** 
4. **Network**
### 🔹 Step 2: Key Linux Commands (with Explanation)
### **1. CPU Usage**
- Command:
    `top`
    👉 Shows real-time CPU/memory usage by processes.
    - `%us` = CPU in user space
    - `%sy` = CPU in kernel
    - `%id` = idle
    - `%wa` = waiting for I/O
- Command:
    `mpstat -P ALL 1`
    (from `sysstat` package)  
    👉 Per-core CPU utilization every second.

### **2. Memory Usage**
- Command:
    `free -h`
    👉 Shows used, free, and available memory (including buffers/cache).
- Command:
    `vmstat 1`
    👉 Real-time CPU, memory, and swap stats.
    - `si/so` = swap in/out (bad if high).
    - `r` = run queue (waiting processes).
### **3. Disk I/O**
- Command:
    `iostat -x 1`
    👉 Extended disk I/O stats per device.
    - `%util` = how busy disk is (close to 100% = bottleneck).
    - `await` = avg. I/O wait time (high = slow disk).
- Command:
    `df -h`
    👉 Disk space usage (important for application performance).
- Command:
    `du -sh /path/*`
    👉 Find which directories consume most disk space.
### **4. Network Performance**

- Command:
    `sar -n DEV 1`
    👉 Network traffic stats per interface (rx/tx).
- Command:
    `ss -tulpn`
    👉 Shows listening ports and socket connections.
- Command:
    `iperf3 -c <server-ip>`
    👉 Measure network throughput between two systems.
### **5. Process-Specific Monitoring**
- Command:
    `ps -eo pid,comm,%cpu,%mem --sort=-%cpu | head`
    👉 Top CPU-consuming processes.
- Command:
    `strace -p <pid>`
    👉 Trace system calls of a running process (useful for debugging).
### 🔹 Step 3: Additional Tools for Deep Performance Analysis

- **sar (System Activity Report)** :  From `sysstat`, collects historical CPU, memory, disk, network performance.
- **htop** :    Interactive process viewer (better than `top`).
- **dstat** :    Combines CPU, disk, network stats in one view.
- **atop**  :    Advanced monitor with logging — great for historical analysis.
- **perf**  :    Kernel-level performance profiler (for developers).
- **Nagios / Zabbix / Prometheus + Grafana**  :    Monitoring + alerting solutions used in enterprises.
- **Application Profiling**  :    Some apps (DBs, web servers) have their own performance tools/logs — e.g., `mysqltuner` for MySQL.
### 🔹 Step 4: Interview-Style Answer
> “To ensure application performance is met, I start by checking the 4 key resources — CPU, memory, disk, and network. I use commands like `top`, `mpstat`, and `vmstat` for CPU and memory, `iostat` and `df` for disk I/O and space, and `sar -n` or `iperf` for network. For processes, I use `ps` and `strace`. For long-term analysis, I rely on tools like `sar`, `htop`, `atop`, or enterprise monitoring solutions like Zabbix and Prometheus. This way, I can quickly identify bottlenecks and ensure the application’s SLAs are being met.”

### 🔹 Troubleshooting Flow for Application Performance

#### **Step 1: Check Overall System Load**
- Command
    `uptime`
    
    👉 Shows **load average** (1 min, 5 min, 15 min).
    - If higher than CPU cores → system is overloaded.
#### **Step 2: CPU Investigation**
- Command:
    `top -o %CPU`
    👉 Identify top CPU-consuming processes.
- If CPU usage is high:
    - Check if it’s **user processes** (apps, scripts) → optimize code.
    - Or **kernel/system** usage → maybe interrupts, I/O bottleneck.
- Extra command:
    `mpstat -P ALL 1`
    👉 See per-core utilization. Sometimes only 1 core is maxed out while others idle.
#### **Step 3: Memory & Swap**
- Command:
    `free -h`
    👉 Check if swap is being used.
- Command:
    `vmstat 1`
    👉 Look at `si/so` (swap in/out). If non-zero → memory pressure.
- If memory is full:
    - Increase RAM
    - Tune application memory usage
    - Check for memory leaks with:
        `ps aux --sort=-%mem | head`
#### **Step 4: Disk I/O Bottlenecks**
- Command:
    `iostat -x 1`
    👉 Focus on:
    - `await` → avg. response time (ms)
    - `%util` → disk busy percentage
- Command:
    `df -h`
    👉 Check free disk space (full disks slow down apps).
- If disk is slow:
    - Migrate to faster storage (SSD, SAN, NVMe)
    - Use **LVM striping** or RAID for performance
    - Optimize application queries / cache
### **Step 5: Network Issues**
- Command:
    `sar -n DEV 1`
    👉 See packet drops/errors.
- Command:
    `ss -tulpn`
    👉 Check for high number of connections.
- Command:
    `iperf3 -c <server>`
    👉 Test bandwidth.
- If network is bottleneck:
    - Increase NIC bonding / teaming
    - Optimize firewall rules
    - Check latency with `ping` / `mtr`
### **Step 6: Application-Specific Checks**
- Database? → Check slow queries (`mysqltuner`, `pg_stat_activity`).
- Web server? → Check logs (`tail -f /var/log/httpd/*`).
- JVM apps? → Use `jstat`, `jmap`, `jconsole`.
    
### **Step 7: Long-Term Monitoring**

- Use **sar, atop, collectl** for historical data.
- Use **Nagios, Zabbix, Prometheus + Grafana** for alerts and dashboards.
### 🔹 Interview-Ready Short Answer
> “My troubleshooting approach is systematic:  
> I start with `uptime` to check system load. Then I look at CPU (`top`, `mpstat`), memory (`free`, `vmstat`), disk I/O (`iostat`, `df`), and network (`sar -n`, `ss`).  
> Based on the bottleneck, I dig deeper — for example, high memory → check for swap and leaks, high disk I/O → check response time and disk utilization, high network latency → test with `iperf`.  
> For long-term performance, I use monitoring tools like `sar`, `atop`, and enterprise monitoring solutions like Zabbix or Prometheus. This ensures application SLAs are consistently met.”



# Hard Link vs Soft (Symbolic) Link

### **Hard Link**
- A **mirror reference** to the same inode (actual data on disk).
- Original file and hard link are **indistinguishable** — both point to the same data.
- If you delete the original file → data is still accessible through the hard link.
- Cannot link across filesystems.
- Cannot be created for directories (to avoid loops).
**Command to create:**
`ln file1 file2`
### **Soft Link (Symbolic Link)**
- A **shortcut / pointer** to the filename, not directly to the data.
- Stores the path of the target file.
- If the original file is deleted → soft link becomes **broken (dangling link)**.
- Can cross filesystems.
- Can be made for directories as well.
**Command to create:**
`ln -s file1 link1`
## 🔹 **How to Identify?**

### 1. Using `ls -l`
`ls -l`
- **Soft link** → shows `l` at start and `->` pointing to the target:
    `lrwxrwxrwx  1 user  user   5 Aug 25  link1 -> file1`
- **Hard link** → looks like a normal file, but:
    - Check **link count** (2nd column > 1).
    - Example:
        `-rw-r--r--  2 user  user  0 Aug 25  file1 -rw-r--r--  2 user  user  0 Aug 25  file2`
    Both `file1` and `file2` are hard links to the same inode.

### 2. Using `ls -i` (inode number)
`ls -i`
- **Hard links** → share the same inode number.
- **Soft link** → has a different inode (since it’s a new file that points to path).
    
✅ **Quick Interview One-Liner Answer:**  
“A hard link points directly to the same inode, so deleting the original file doesn’t break it. A soft link is just a pointer to the file path, and it breaks if the target is deleted. We can check with `ls -li`: same inode = hard link, `l` and `->` = soft link.”

# SSH Hardening

SSH (Secure Shell) is the most common way to remotely administer Linux/Unix servers, so attackers often target it. **SSH hardening** means applying best practices and configurations to reduce the attack surface and secure remote access.
## 🔑 Key Areas of SSH Hardening
### 1. **Access Control**
- **Disable root login**  
    In `/etc/ssh/sshd_config` → `PermitRootLogin no`  
    → Force users to log in with their accounts and then escalate with `sudo`.
- **Allow only specific users or groups**  
    `AllowUsers admin1 admin2`  
    or  
    `AllowGroups sshusers`
### 2. **Authentication Security**
- **Use key-based authentication** (instead of passwords)
    - Generate SSH key pair: `ssh-keygen`
    - Copy public key to server: `ssh-copy-id user@server`
    - Disable password authentication: `PasswordAuthentication no`
- **Enforce strong key types**  
    Use modern algorithms like `ed25519` or `rsa ≥ 4096`.
### 3. **Network & Port Settings**
- **Change the default port (22)**  
    → Not foolproof, but reduces random scans: `Port 2222`
- **Limit access with firewall**  
    Only allow SSH from trusted IP ranges.
- **Enable rate limiting / intrusion detection**  
    Tools like **fail2ban** or **iptables recent module** can block repeated failed attempts.
### 4. **Protocol & Cipher Security**
- **Disable SSH protocol 1**  
    Only allow protocol 2: `Protocol 2`
- **Restrict ciphers and MACs**  
    Example in `sshd_config`:
```
Ciphers aes256-ctr,aes192-ctr,aes128-ctr
MACs hmac-sha2-512,hmac-sha2-256
KexAlgorithms curve25519-sha256
```
### 5. **Timeouts & Session Control**
- Set idle session timeout:
```
ClientAliveInterval 300
ClientAliveCountMax 2
```
    → Disconnect idle users after 10 minutes.
- Restrict X11 forwarding and agent forwarding unless required:  
    `X11Forwarding no`  
    `AllowAgentForwarding no`
### 6. **Logging & Monitoring**
- Enable detailed logging in `/var/log/secure` or `/var/log/auth.log`.
- Regularly check for suspicious attempts (`grep sshd` in logs).
- Consider using centralized log monitoring.
### 7. **Additional Layers**
- **Use Multi-Factor Authentication (MFA)** with PAM + Google Authenticator or Duo.
- **Port knocking** or **VPN before SSH** for sensitive environments.
## 🎯 Example Answer for Interview

> _“SSH hardening is about minimizing risk of unauthorized access. I usually disable root login, enforce key-based authentication, and restrict users/groups. I also tune `sshd_config` by disabling password login, protocol 1, and weak ciphers. On the network side, I restrict access with firewalls and intrusion detection tools like fail2ban. Additionally, I configure idle session timeouts, monitor logs for brute-force attempts, and where possible, add MFA for stronger security.”_

---
Let’s build a **practical SSH hardening checklist** you can **apply on servers** and also **use as a crisp interview-ready answer**.

## 🔐 SSH Hardening Checklist

### 1. Backup Config
`cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak`
### 2. Disable Root Login
Edit `/etc/ssh/sshd_config`:
`PermitRootLogin no`
### 3. Restrict Users/Groups
```
AllowUsers admin1 admin2
# OR
AllowGroups sshusers
```
### 4. Enforce Key-Based Authentication
```
ssh-keygen -t ed25519
ssh-copy-id user@server
```
- Disable password login:
```
PasswordAuthentication no
ChallengeResponseAuthentication no
```
### 5. Change Default Port
`Port 2222`
_(Remember to update firewall rules & inform team before changing!)_
### 6. Disable Weak Protocols & Ciphers
```
Protocol 2
Ciphers aes256-ctr,aes192-ctr,aes128-ctr
MACs hmac-sha2-512,hmac-sha2-256
KexAlgorithms curve25519-sha256
```
### 7. Set Idle Session Timeout
`ClientAliveInterval 300 ClientAliveCountMax 2`
_(Disconnects idle sessions after ~10 mins)_
### 8. Disable Forwarding (if not needed)
`X11Forwarding no AllowAgentForwarding no`
### 9. Firewall Protection
Allow SSH only from trusted IPs:
```
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" port protocol="tcp" port="2222" accept'
firewall-cmd --reload
```
### 10. Intrusion Prevention

Install **fail2ban** to block brute-force attempts:
```
yum install fail2ban -y    # RHEL/CentOS
apt install fail2ban -y    # Ubuntu/Debian
systemctl enable --now fail2ban
```
### 11. Enable Logging
Check logs at:
`tail -f /var/log/secure    # RHEL tail -f /var/log/auth.log  # Ubuntu/Debian`
### 12. Advanced Security (Optional)
- Enable **MFA** (Google Authenticator / Duo).
- Use **VPN before SSH** for critical systems.
- Set up **port knocking** for hidden access.
### 13. Restart SSH Service
`systemctl restart sshd`

✅ **Quick Interview Line**:  
_"I usually harden SSH by disabling root and password logins, enforcing key-based authentication, restricting users, changing the default port, and disabling weak ciphers. I also set idle timeouts, restrict forwarding, and protect access with firewall rules and fail2ban. Finally, I monitor logs and, where needed, add MFA or VPN access for an extra layer of security."_

# Boot Process and Issues

## Boot Process

BIOS/ UEFI Initialization:
 - When you power onn your machine, BIOS/ UEFI Firmware runs  
 - It performs POST(Power On Self Test) to check hardware like RAM, CPU, Keyboard.
 - Further it looks for a bootable device like hard disk, USB.
-MBR/GPT and Bootloader Stage 1
- On Bios system - MBR (Master Boot Record) is read from first sector of boot disk(512 bytes)
- On UEFI system (Unified Extended Firmware Interface - modern replacement of BIOS offering faster boot times and larger disk over 2TB) - It looks for bootloader in EFI system partition.
- The bootloader like Grub is loaded into memory
-Bootloader Stage 2
- Grub is most common Bootloader, it shows the menu to choose from, if multiple OS/Kernel versions are present, selected kernel loads and also loads initramfs (Initial RAM Filesystem - temporary drivers and tools to access disk)
-Kernel Initialization Stage 3
- The Linux Kernel is uncompressed and starts running, loads and initialize hardware like CPU, RAM, I/O devices.
- Mounts the root FS from Disk using drivers from initramfs and handovers to init process
-Init/Systemd Process Stage 4
- The first process started by kernel is PID 1.
- Systemd reads the configuration file and starts the system processes like network, SSH, cron, firewall etc. according to Target.
-Target: At last it reaches the **desired target**: `multi-user` or `graphical`.

Below are Common Targets / Runlevels

| Target / Runlevel   | What it means (easy words)                                                        |
| ------------------- | --------------------------------------------------------------------------------- |
| `multi-user.target` | Text mode → server mode, no GUI. You see a **command-line login**.                |
| `graphical.target`  | GUI mode → desktop, with windows and icons. You see a **graphical login screen**. |

## Boot Issues

Before issues, recall the boot stages:
1. **BIOS/UEFI** → Hardware check, loads bootloader.
2. **Bootloader (GRUB2)** → Loads kernel & initramfs.
3. **Kernel** → Mounts root filesystem, starts init/systemd.
4. **Systemd/init** → Brings up services → login.
If something breaks in between → boot issue.

![[Pasted image 20250902235730.png]]

### 1️⃣ **GRUB Issues (Bootloader problem)**

👉 GRUB = the car’s **ignition key system**. If it’s broken, engine won’t start.
#### When it happens:
- You restart server → screen shows **“grub rescue>”** instead of Linux login.
- Means GRUB can’t find boot partition (maybe disk order changed, or config got corrupted).
#### Where you are?
You’re at the **bootloader screen**, not Linux shell. You didn’t run any command — system brought you here because it didn’t know how to boot.
#### How to Fix:
1. **Find where Linux root is**
`ls`
This shows all disks/partitions like `(hd0,msdos1) (hd0,msdos2)` etc.
2. **Check each partition for Linux**
`ls (hd0,msdos1)/`
If you see `/boot` or `/vmlinuz` → that’s the right partition.
3. **Tell grub where boot files are**
`set root=(hd0,msdos1) 
`set prefix=(hd0,msdos1)/boot/grub` 
`[^1]insmod normal` 
`normal`
`boot`

Now system should try to boot Linux.
👉 Then once system is up, reinstall grub permanently:
`grub2-install /dev/sda 
`grub2-mkconfig -o /boot/grub2/grub.cfg`

🧠 **How to explain in interview:**  
“GRUB rescue happens when bootloader can’t find kernel. I find the correct partition using `ls`, set it as root, load grub modules, boot the system, and then reinstall grub permanently.”

---

### 2️⃣ **Kernel Panic (Missing or corrupted kernel/initramfs)**

👉 Think: the **engine** is missing.
#### Symptoms:
- System boots, then stops with “Kernel panic – not syncing”.
#### Fix: with kernel reinstall
1. At GRUB menu, select **Older Kernel** (if available).
2. If works → reinstall kernel package:
`yum reinstall kernel`
or
`dnf reinstall kernel`
#### Fix with `dracut`

 **What is `dracut`?**
- `dracut` is the tool that **creates the initramfs (initial RAM filesystem)**.
- Initramfs = the small mini-OS loaded into RAM before the actual Linux root filesystem mounts.
- If it’s missing or corrupted → you’ll get **Kernel panic: unable to mount root fs**.
  
1. Boot into **rescue mode** or **older kernel** (if available).
2. Rebuild initramfs:
`dracut -f /boot/initramfs-$(uname -r).img $(uname -r)`
- `-f` → force overwrite existing one.
- `$(uname -r)` → uses current running kernel version.
👉 Example:
`dracut -f /boot/initramfs-3.10.0-1160.el7.x86_64.img 3.10.0-1160.el7.x86_64`
3. Reboot:
`reboot`

---
##### Step 1: How to know if **kernel** is missing vs **initramfs** missing?

When server fails to boot, the error message is your clue:
1. **Kernel missing/corrupt**
    - Error like:
        `error: file '/vmlinuz-<version>' not found`
    - Or GRUB menu shows no kernel entries at all.
    - Means the actual Linux kernel binary (`vmlinuz`) is gone.
    - Fix = reinstall kernel package (`yum reinstall kernel`).
2. **Initramfs missing/corrupt**
    - Error like:
        `Kernel panic - not syncing: VFS: Unable to mount root fs`
    - Or message:
        `initramfs image not found`
    - Check `/boot/initramfs-<kernel-version>.img` → if file missing/broken.
    - Fix = regenerate using `dracut`.

👉 Easy trick to remember:
No kernel file? → reinstall kernel.
Kernel loads but cannot mount root fs? → rebuild initramfs.

 ### **Corrupt/missing initramfs or kernel**

🔹 **Symptoms:**
- Boot fails after grub menu
- Error: "init not found"
🔹 **Fix:**  
Boot from rescue → chroot into system:
`mount /dev/sdaX /mnt   # root partition`
`mount --bind /dev /mnt/dev`
`mount --bind /proc /mnt/proc`
`mount --bind /sys /mnt/sys`
`chroot /mnt`

Rebuild kernel/initramfs: 
`dracut -f yum reinstall kernel`    # RHEL/CentOS
##### 🌟 Step 2: How to boot into **older kernel**?

If system has multiple kernels installed, GRUB menu will list them.  
When booting:

1. Press **`Esc`** (or **Shift** in some distros) during boot → GRUB menu opens.
2. Select **“Advanced options for RHEL/CentOS”**.
3. Pick an **older kernel version** (not the latest one).
👉 This works if you have previously installed updates and older kernel still exists.

---

##### 🌟 Step 3: How to boot into **rescue mode**?

Rescue mode = minimal environment for repair.
##### Option A: From GRUB
1. At GRUB menu → select kernel.
2. Press **`e`** to edit boot entry.
3. Find line starting with `linux16` or `linux`.
4. At end, add:
    `systemd.unit=rescue.target`
5. Press **Ctrl+x** → boot into rescue mode (single-user shell).
##### Option B: Using boot ISO/DVD
- Boot from RHEL/CentOS DVD/ISO.
- Choose **Troubleshooting → Rescue a Red Hat system**.
- It will mount your system under `/mnt/sysimage`.
- Run:
    `chroot /mnt/sysimage`
    (Now you’re “inside” your broken system and can fix it).

---

##### 🌟 Step 4: Connecting the dots

- If kernel missing → boot into rescue/older kernel → reinstall kernel    
- If initramfs missing → boot into rescue/older kernel → run `dracut -f`.
- If both okay but FS broken → you’ll be dropped into **emergency mode** automatically → fix `fstab` or run `fsck`.
---

✅ So in interviews you can say:

> “If kernel panic happens due to missing initramfs, I boot from another kernel or rescue mode and run `dracut -f` to rebuild initramfs. If kernel package itself is broken, I reinstall kernel.”

---

### 3️⃣ **Wrong fstab entries (bad UUID or mount)**

👉 Think: you told car to start with **wrong key**.
### Symptoms:
- Boot stops → you get **emergency mode** prompt.
- Emergency mode = special safe mode so you can fix system. (You’ll see `(emergency)` at shell.)
### Fix:
1. Open fstab:
`vi /etc/fstab`
2. Comment wrong UUID or mount (add `#`).
3. Save and exit.
4. Reboot:
`reboot`
👉 Yes — system comes up after reboot because now it ignores the wrong mount entry.

🧠 **Interview way:**  
“If system drops to emergency mode due to wrong fstab, I go in, comment bad entries, reboot, then correct the UUID later with `blkid` and update fstab properly.”

---

### 4️⃣ **SELinux in Enforcing Mode (policy blocks boot)**
👉 Think: security guard is too strict and doesn’t let engine start.
#### Fix:
1. At GRUB, press `e` to edit boot entry.
2. Find line starting with `linux16` or `linux`.
3. Add:
`selinux=0`
or
`enforcing=0`
4. Boot system (`Ctrl+x`).
Now system boots because SELinux is **Permissive** (it just warns, doesn’t block).
👉 After boot, fix SELinux properly in `/etc/selinux/config`.
🧠 **Explain:**  
“Permissive means SELinux logs denials but doesn’t enforce them. I disable it temporarily from grub to boot, then fix config later.”
### 5️⃣ **Filesystem Errors**
👉 Think: Car wheels (disk) are broken.
### Symptoms:
- System stops, says: `fsck required`.
- Sometimes directly in **emergency mode**.
### Fix:
Run fsck manually:
`fsck -y /dev/sda1`
Then reboot:
`reboot`
### 6️⃣ **Password Lost (root password reset)**

👉 Think: you lost car keys.
### Fix:
1. At GRUB menu, press `e`.
2. On `linux16` line, add:
`rd.break`
3. Boot (`Ctrl+x`).
4. You’ll be in emergency shell. Root is mounted read-only.  
    Make it read/write:
`mount -o remount,rw /sysroot`
👉 Why? Because we need to change files, and rw = read-write mode.
5. Chroot (change root directory):
`chroot /sysroot`
6. Reset password:
`passwd root`
7. Relabel SELinux (important, or login may fail):
`touch /.autorelabel`
8. Exit and reboot.
### 7️⃣ **Network/Service startup failure**

🔹 **Symptoms:** Boots but no network/ssh
🔹 **Fix:**
- Boot into rescue/emergency → enable service:
`systemctl enable NetworkManager`
`systemctl start NetworkManager`
- Check logs:
`journalctl -xb`
### ✅ **How to Remember**:
- **GRUB** = ignition key missing → set root + reinstall grub (`grub2-install`).
- **Kernel panic** = engine missing → boot old kernel + reinstall kernel .
- **fstab error** = wrong key → emergency mode → comment entry → reboot.
- **SELinux enforcing** = security guard blocking → permissive mode.
- **Filesystem error** = wheels broken → run fsck.
- **Root password lost** = keys lost → rd.break → reset.
- **Services not starting?** → `systemctl`, `journalctl`.
# Top command in Linux

When you run: `top`
It shows a **live dashboard** of system performance.  
Think of it like a **doctor’s monitor** for your server’s health.

---
## 🔹 Top Section = System Summary (Doctor’s dashboard header)

1. **`top - 14:12:31 up 3 days, 2:05, 3 users, load average: 0.01, 0.05, 0.10`**    
    - `14:12:31` → Current system time.
    - `up 3 days, 2:05` → Uptime (how long server is running).
    - `3 users` → Logged-in users.
    - `load average: 0.01, 0.05, 0.10` → CPU load in last **1, 5, 15 minutes**.  
        👉 Rule of thumb:
        - If load ≈ number of CPUs → system is busy.
        - If load > CPUs → system is overloaded.
---
2. **Tasks** (like “patients in queue)    
    `Tasks: 123 total, 1 running, 122 sleeping, 0 stopped, 0 zombie`
    - Total processes = 123
    - `running` → actively using CPU
    - `sleeping` → waiting (normal state for most daemons)
    - `stopped` → paused
    - [^3]zombie` → dead but not cleaned up (bad sign!)
---

3. **CPU usage** (doctor checking heart activity ❤️)    
    `%Cpu(s):  1.0 us, 0.5 sy, 0.0 ni, 98.0 id, 0.2 wa, 0.0 hi, 0.0 si, 0.3 st`
    - `us` (user space) → CPU used by apps.        
    - `sy` (system/kernel) → CPU used by OS.
    - `ni` ([^2]nice) → CPU for priority tasks.
    - `id` (idle) → Free CPU.        
    - [^4]wa (iowait) → CPU waiting on disk I/O.
    - [^5]hi (hardware [^6]interrupts).
    - `si` (software interrupts).
    - `st` (steal time → hypervisor stealing CPU in VMs).

👉 If `wa` is high → disk bottleneck.  
👉 If `sy` is high → kernel/driver issue.  
👉 If `id` is low and `us` is high → CPU heavy load.

---

4. **Memory usage** (doctor checking body fluids 💧)    
    `KiB Mem :  16318172 total,  456732 free, 12000000 used, 3851440 buff/cache KiB Swap:  8191996 total,  8180000 free,   11996 used.`
    - **Mem total** → Installed RAM.
    - **free** → RAM not used.
    - **used** → RAM used by processes.
    - **buff/cache** → RAM used for cache (Linux will use free RAM for speed; cache can be freed if needed).
    - **Swap** → Virtual memory on disk.
👉 If Swap is being used heavily → server needs more RAM.
---
## 🔹 Bottom Section = Process Table (like patient list 📝)

Columns you’ll see:

| Column      | Meaning             | Interview Tip                                 |
| ----------- | ------------------- | --------------------------------------------- |
| **PID**     | Process ID          | Unique number for each process                |
| **USER**    | Owner of process    | Helps know who started it                     |
| **PR**      | Priority            | Lower number = higher priority                |
| **NI**      | Nice value          | Value to adjust priority                      |
| **VIRT**    | Virtual memory used | Includes RAM + swap                           |
| **RES**     | Resident memory     | Actual physical RAM used                      |
| **SHR**     | Shared memory       | Shared libraries used                         |
| **S**       | State (R/S/D/T/Z)   | Running, Sleeping, Disk wait, Stopped, Zombie |
| **%CPU**    | CPU usage           | If one process hogs → bottleneck              |
| **%MEM**    | Memory usage        | Helps find memory-hungry process              |
| **TIME+**   | Total CPU time used | Long-running hogs show up here                |
| **COMMAND** | Process name        | Which program it is                           |
- **PR** = **Priority** of the process (what the kernel scheduler uses).
- **NI** = **Nice value** of the process (user-level way to suggest priority).
  ##### How they relate?
- Every process has a **base priority**.
- When you set a **nice value (NI)**, Linux converts it into a **priority (PR)**.
- **Lower PR = higher priority** (opposite of what people expect).
 PR = 20 + NI
- Default NI = 0 → PR = 20 (normal).
- NI = -5 → PR = 15 (higher priority). (CPU favors it)
- NI = +5 → PR = 25 (lower priority). (CPU gives less attention to this)
---
## 🔹 Useful Keys inside `top`

While `top` is running, you can press:

- `P` → Sort by CPU usage.
- `M` → Sort by memory usage.
- `k` → Kill a process (will ask for PID).
- `u` → Show processes for a specific user.
- `1` → Show usage of each CPU core.
- `q` → Quit.
---

✅ **Interview-style summary**:

> “The `top` command gives a live view of system performance. At the top, it shows load average, tasks, CPU, and memory usage. At the bottom, it lists all processes with details like PID, CPU%, MEM%, and state. We can sort processes interactively, monitor bottlenecks, and identify high CPU, memory, or I/O consuming processes.”

## 📝 Key Interview Point

- **High load + low CPU** → usually **I/O bottleneck** (disk, NFS, network storage).
- **High load + high CPU** → true CPU saturation.

# Tuning to Fix load issues.

"Tuning depends on where the bottleneck is: for CPU, I adjust priority (nice/renice, cgroups, thread limits); for I/O, I check with `iostat/iotop` and optimize storage or app access patterns; for DB queries, I use `EXPLAIN`, add indexes, and tune DB parameters. I’ve applied these in production — for example, I reduced DB query load by adding indexes, which cut query time from 2 minutes to 3 seconds."
# 🧠 What is DNS?

**DNS (Domain Name System)** is like the **phonebook of the internet**. It maps **domain names** (like `example.com`) to **IP addresses** (like `192.0.2.1`).
## 📦 Components of DNS Setup (Linux)

### 1. **Resolver (Client-side DNS)**
Used by local machines to resolve names to IP addresses.
- **Files involved**:
        - `/etc/resolv.conf`: Lists DNS servers (nameservers)
            `nameserver 8.8.8.8 nameserver 1.1.1.1`
         - `/etc/hosts`: Manual name resolution (bypasses DNS)
        `127.0.0.1   localhost 192.168.1.10 myserver.local`
- **Tools**:
    - `dig`, `nslookup`, `host` – for DNS queries
    - `ping` – checks if name resolves
    - `systemd-resolve --status` – check systemd-resolved
    - `resolvectl` (newer distros)
---
### 2. **DNS Server (Name Server)**

Resolves names for others. Can be:
- **Caching nameserver** (like `bind9` or `dnsmasq`)    
- **Authoritative nameserver** (holds actual domain records)
- **Recursive resolver** (resolves full chain from root to domain)
---
## ⚙️ Setting Up DNS Server (BIND - Common in Linux)

Install BIND:
`sudo apt install bind9    # Debian/Ubuntu sudo yum install bind     # RHEL/CentOS`
Main config files (usually in `/etc/bind/` or `/var/named/`):

| File                        | Purpose                          |
| --------------------------- | -------------------------------- |
| `named.conf`                | Main config file                 |
| `named.conf.options`        | Global options (e.g., recursion) |
| `named.conf.local`          | Define zones                     |
| `db.example.com`            | Forward zone (A, MX, CNAME)      |
| `db.1.168.192.in-addr.arpa` | Reverse zone (PTR records)       |

---
### 🧾 Zone File Example (`db.example.com`)

```
$TTL 86400
@   IN  SOA ns1.example.com. admin.example.com. (
        2025090301 ; Serial
        3600       ; Refresh
        1800       ; Retry
        604800     ; Expire
        86400 )    ; Minimum TTL

    IN  NS  ns1.example.com.
    IN  A   192.168.1.10
www IN  A   192.168.1.11
mail IN  A   192.168.1.12
    IN  MX  10 mail.example.com.

```
---

### 🔄 Reverse Zone Example (for PTR Records)

Maps IP to name.

For IP `192.168.1.11`, zone file:
```
$TTL 86400
@   IN  SOA ns1.example.com. admin.example.com. (
        2025090301 ; Serial
        3600       ; Refresh
        1800       ; Retry
        604800     ; Expire
        86400 )    ; Minimum TTL

    IN  NS  ns1.example.com.
11  IN  PTR www.example.com.

```

---
## 🔥 Common DNS Records

| Type    | Purpose                |
| ------- | ---------------------- |
| `A`     | IPv4 address           |
| `AAAA`  | IPv6 address           |
| `CNAME` | Alias                  |
| `MX`    | Mail server            |
| `NS`    | Name server            |
| `PTR`   | Reverse DNS            |
| `SOA`   | Start of Authority     |
| `TXT`   | Text (e.g., SPF, DKIM) |

---

## ✅ How to Check DNS is Working

### On Client:
```
dig example.com
nslookup example.com
host example.com

```
### On Server:
```
named-checkconf             # check config
named-checkzone example.com /etc/bind/db.example.com
systemctl restart bind9     # restart service
journalctl -xe              # check logs

```


---

## 🔐 DNS Security Topics (Optional but Nice for Interview)

- **DNSSEC**: Signs DNS data to prevent spoofing.
- **Split-horizon DNS**: Different DNS responses for internal vs external users.
- **Firewall Ports**:
    - **Port 53 (UDP/TCP)** must be open for DNS.
---

## 📋 DNS Cheat-Sheet Summary (For Fast Revision)

```
🧠 DNS: Maps domain → IP (like phonebook)

📍Client Side:
- /etc/resolv.conf → DNS servers
- /etc/hosts → Manual mapping

🛠️ Tools: dig, nslookup, host, ping

📦 DNS Server (BIND):
- named.conf → main config
- named.conf.local → zones
- db.example.com → zone file

📝 Records:
- A, AAAA → IP addresses
- MX → mail
- CNAME → alias
- PTR → reverse lookup
- NS → nameserver
- SOA → authority

🧪 Check:
- dig, nslookup
- named-checkconf
- named-checkzone
- systemctl restart bind9

🛡️ Secure:
- DNSSEC, firewall (UDP/TCP 53)


```

---

## 🎓 Interview Tips

- **Know how DNS resolution works**: root → TLD → authoritative
- **Know difference**: Resolver vs Authoritative server
- **Practice dig output**: Understand sections (QUESTION, ANSWER)
- **Be ready to troubleshoot**: typos, TTLs, firewall, wrong records
  
  
```
📍 DNS Resolution Path:
  Client → Resolver → Root → TLD → Authoritative → IP

📌 Resolver vs Authoritative:
  Resolver = Queries DNS
  Authoritative = Stores Answers

🔍 dig output:
  - QUESTION: What was asked
  - ANSWER: IP or data returned
  - AUTHORITY: Which server owns domain
  - TTL: Time to cache

🛠️ Troubleshooting DNS:
  - Typos in names
  - TTL/delay in propagation
  - Firewall on port 53
  - Wrong or missing records
  - DNS server not restarted

```

# DHCP

📌 DHCP = Dynamic Host Configuration Protocol
- Assigns IP, DNS, gateway automatically

⚙️ Components:
- Client: dhclient
- Server: isc-dhcp-server
- Lease: IP with timeout (renewable)

📤 DORA Process:
1. Discover
2. Offer
3. Request
4. Acknowledge

🛠️ Server Config (dhcpd.conf):
- subnet 192.168.1.0 netmask 255.255.255.0 {
    range 192.168.1.100 192.168.1.200;
    option routers 192.168.1.1;
    option domain-name-servers 8.8.8.8;
  }

📄 Leases: /var/lib/dhcp/dhcpd.leases

🧪 Commands:
- Start server: systemctl restart isc-dhcp-server
- Request IP: sudo dhclient eth0
- Check IP: ip a
- Logs: journalctl -u isc-dhcp-server

🔐 Ports:
- Server: UDP 67
- Client: UDP 68

| Question                     | Key Answer                                                         |
| ---------------------------- | ------------------------------------------------------------------ |
| **Two DHCP servers respond** | Client picks the first offer it gets (possible IP conflict risk)   |
| **Prevent IP conflicts**     | Use non-overlapping ranges, reservations, static IPs outside pool  |
| **Static IP via DHCP**       | Use `host { hardware ethernet ...; fixed-address ...; }` in config |

# **Multithreading vs Multiprocessing**

### **1. Definition**

- **Multithreading** → A single process is divided into multiple threads.    
- **Multiprocessing** → Multiple processes run simultaneously, each with its own memory space.
### **2. Memory**
- **Multithreading**: Threads **share the same memory** of the parent process → lightweight.
- **Multiprocessing**: Each process has its **own memory** → heavy.
### **3. Communication**
- **Multithreading**: Easy, since all threads share memory (but risk of race conditions).
- **Multiprocessing**: Harder, requires **IPC (Inter-Process Communication)** like pipes, sockets, shared memory.
### **4. Example**
- **Multithreading**: Web browser → one thread loads images, another handles user input, another plays video.
- **Multiprocessing**: Opening multiple browser windows (each window is a process).
### **5. Performance**
- **Multithreading**: Faster for tasks that need shared data.
- **Multiprocessing**: More stable—if one process crashes, others keep running.
### **6. Short Interview Answer**

_"Multithreading allows a single process to run multiple threads in the same memory space, making it lightweight and fast but prone to race conditions. Multiprocessing runs multiple independent processes, each with its own memory, which is heavier but more stable. A web browser tab is multiprocessing, while rendering within a tab (loading images, scripts, video) is multithreading."_

## 🔹 **Checking Processes vs Threads in Linux**
### **1. Normal Process View**
`ps -ef`
- This shows **processes**.
- Each line is a separate process (with its own PID).
### **2. Show Threads of a Process**
`ps -Lf -p <PID>`
- `-L` → shows threads.
- `-p` → specify the process ID.
- Example:
    ps -ef | grep firefox 
    ps -Lf -p 1234`
    - You’ll see one Firefox process may have **dozens of threads** (rendering, network, GPU, etc.).
### **3. With `top`**
- Run `top` normally → shows processes.
- Press **H** inside `top` → switches to **thread view**.
- You’ll see LWP (Lightweight Process = thread) IDs.
### **4. With `htop`**
- Run `htop`.
- Press **Shift + H** → toggle **thread display**.
- You can visually see how one process (like Chrome, Firefox, or Java) has many threads.
### **5. Multiprocessing Example**
Run:
`firefox & gedit &`
- These are **separate processes** → each has its own PID and memory space.
### **6. Multithreading Example**
Run:
`java MyThreadProgram`
Then:
`ps -Lf -p <Java_PID>`
- You’ll see multiple threads inside one Java process.
---

✅ **Interview-ready short answer:**  
_"In Linux, I use `ps -ef` for processes and `ps -Lf -p <PID>` to see threads of a process. In `top`, pressing `H` shows threads, and in `htop`, Shift+H toggles thread view. For example, Firefox runs multiple processes, while a Java program can run multiple threads within one process."_

#  **What is logrotate?**

- A Linux utility that **automatically rotates, compresses, and removes old log files**.
- Prevents logs from filling up the disk.
- Runs daily via **cron** by default (`/etc/cron.daily/logrotate`).

## 🔹 **Configuration Files**

1. **Main config file** → `/etc/logrotate.conf`
2. **Per-service configs** → `/etc/logrotate.d/` (e.g., `/etc/logrotate.d/httpd` for Apache logs).
---
## 🔹 **Basic Configuration Example**

Let’s say we want to rotate `/var/log/myapp.log`.
Create a file:
`sudo vi /etc/logrotate.d/myapp`
Add:
/var/log/myapp.log {
    daily              # rotate logs daily
    rotate 7           # keep last 7 logs, then delete old
    compress           # compress old logs with gzip
    delaycompress      # compress from second rotation onwards
    missingok          # ignore if log is missing
    notifempty         # don’t rotate empty logs
    create 0640 root root  # create new log with permissions
    postrotate
        systemctl reload myapp.service > /dev/null 2>&1 || true
    endscript
}

---
"We don’t need to create a new cron entry for every app. Logrotate already has a daily cron job (`/etc/cron.daily/logrotate`) that runs `/etc/logrotate.conf`. Since this file includes `/etc/logrotate.d/`, any configs we add there — like `/etc/logrotate.d/myapp` — are picked up automatically. Cron only needs to be touched if we want a custom schedule like hourly."

---
## 🔹 **Key Options**
- `daily`, `weekly`, `monthly` → frequency.    
- `rotate N` → how many backups to keep.
- `compress` → compress with `.gz`.
- `notifempty` → skip empty logs.
- `missingok` → no error if log file doesn’t exist.
- `postrotate/endscript` → run a command after rotation (commonly `systemctl reload` for services).
---
## 🔹 **Test Configuration**
Run in debug mode:
`sudo logrotate -d /etc/logrotate.conf`
Force rotation:
`sudo logrotate -f /etc/logrotate.conf`

---

✅ **Interview-style short answer:**  
_"Logrotate manages log files by rotating, compressing, and removing old logs. Configurations go in `/etc/logrotate.conf` or `/etc/logrotate.d/`. For example, I can set `/var/log/myapp.log` to rotate daily, keep 7 copies, compress old ones, and reload the service using a `postrotate` script. I test it with `logrotate -d` and can force rotation with `logrotate -f`."_

# 🌐 Networking in Linux

## 1. **Linux Networking Components**

Think of networking in Linux like layers:
1. **Interfaces (NICs):**
    - Your physical/virtual network cards: `eth0`, `ens33`, `eno1`.
    - Can have **IP addresses, routes, and VLANs** assigned.
2. **IP Address & Netmask:**
    - Identifies the server on a network.        
    - Netmask defines the **subnet** (e.g., `/24` = 255.255.255.0).
3. **Gateway (Default Route):**
    - The router used to reach networks outside your subnet.
4. **DNS:**
    - Translates names → IPs.
5. **Configuration Files:**
    - **RHEL/CentOS/Rocky/AlmaLinux:**  
        `/etc/sysconfig/network-scripts/ifcfg-<interface>`
    - **RHEL 8+/Rocky/Ubuntu with Netplan/NetworkManager:**  
        `/etc/NetworkManager/system-connections/` (nmcli-managed).
    - **Ubuntu (netplan):**  
        `/etc/netplan/*.yaml`.
    - **DNS:**  
        `/etc/resolv.conf`.
## 2. **Key Commands for Networking**

Here’s the toolkit every L3 should master:
### Interface & IP Management
- `ip a` → Show IPs & interfaces.
- `ip addr add 192.168.1.10/24 dev eth0` → Add IP.
- `ip addr del 192.168.1.10/24 dev eth0` → Remove IP.
- `ip link set eth0 up/down` → Enable/disable interface.
- `ifconfig` (older, from net-tools).
### Routing

- `ip r` → Show routes    
- `ip route add 10.10.0.0/16 via 192.168.1.1 dev eth0` → Add route.
- `ip route del ...` → Delete route.
- `ip route show default` → Show default gateway.
### DNS
- `cat /etc/resolv.conf` → Check DNS servers.
- `dig google.com` / `nslookup google.com` → Test DNS resolution.
### Testing Connectivity
- `ping 8.8.8.8` → Check ICMP connectivity.
- `ping google.com` → Check DNS + ICMP.
- `curl -I http://example.com` → Check HTTP response.
- `traceroute 8.8.8.8` → Path packets take.
- `nc -zv <IP> <port>` → Test TCP connectivity.
### Advanced Tools
- `ethtool eth0` → NIC speed/duplex info.
- `mii-tool eth0` → Older NIC status tool.
- `ss -tulnp` → Show listening ports.
- `tcpdump -i eth0 port 22` → Capture packets (SSH in this case).

## 3. **Configuration Files & Examples**
### Example 1: **RHEL/CentOS (Network Scripts)**
File: `/etc/sysconfig/network-scripts/ifcfg-eth0`
```
DEVICE=eth0
BOOTPROTO=static
ONBOOT=yes
IPADDR=192.168.1.100
NETMASK=255.255.255.0
GATEWAY=192.168.1.1
DNS1=8.8.8.8
DNS2=1.1.1.1
```
Restart network:
```
nmcli con reload
nmcli con up eth0
```
### Example 2: **Ubuntu Netplan**

File: `/etc/netplan/01-netcfg.yaml`
```
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: no
      addresses:
        - 192.168.1.100/24
      gateway4: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]

```

Apply:
`netplan apply`
### Example 3: **Bonding (High Availability NICs)**

Bonding = combining 2 NICs for redundancy/performance.  
File: `/etc/sysconfig/network-scripts/ifcfg-bond0`
```
DEVICE=bond0
BOOTPROTO=static
ONBOOT=yes
IPADDR=192.168.10.100
NETMASK=255.255.255.0
GATEWAY=192.168.10.1
BONDING_OPTS="mode=1 miimon=100"

```
Then slave configs (ifcfg-eth0 & ifcfg-eth1):
```
DEVICE=eth0
MASTER=bond0
SLAVE=yes
ONBOOT=yes
```
### Example 4: **VLANs**

Add VLAN 100 on eth0:
```
ip link add link eth0 name eth0.100 type vlan id 100
ip addr add 192.168.100.10/24 dev eth0.100
ip link set eth0.100 up
```

---

## 4. **Troubleshooting Flow (Interview Gold ⭐)**
If **server cannot reach network**, steps:
1. `ip a` → Does interface have IP?
2. `ip r` → Is default route present?
3. `ping <gateway>` → Is gateway reachable?
4. `ping 8.8.8.8` → Is external IP reachable?
5. `ping google.com` → Is DNS working?
6. `ss -tulnp` → Is service listening?
7. `tcpdump` → Capture packets if advanced debugging needed.
## 5. **Frequently Asked Interview Questions (with Answers)**

### Q1. Difference between `ifconfig` and `ip` command?
**A:**  
`ifconfig` is old (net-tools package), now deprecated.
- `ip` (from iproute2) is modern, supports advanced networking (VLANs, namespaces, etc).
### Q2. How to add a temporary IP vs permanent IP?
**A:**
- **Temporary:**    
    `ip addr add 192.168.1.50/24 dev eth0`
    Lost after reboot.
- **Permanent:**  
    Update config file (`ifcfg-eth0` or netplan YAML).
### Q3. What is bonding?
**A:** Combining 2+ NICs for redundancy/performance. Modes include:
- Mode 0 (round-robin) – load balancing.
- Mode 1 (active-backup) – high availability.
- Mode 4 (802.3ad) – LACP with switch support.
### Q4. What’s the difference between a bridge, bond, and VLAN?
**A:**
- **Bridge:** Works like a virtual switch (used in KVM).
- **Bond:** Combines multiple NICs into one.
- **VLAN:** Logical separation of traffic on the same physical interface.
### Q5. Where do you configure default gateway & DNS?
**A:**
- Gateway: `/etc/sysconfig/network-scripts/ifcfg-<interface>` (`GATEWAY=`).
- DNS: `/etc/resolv.conf` or via interface config (`DNS1=, DNS2=`).
### Q6. How do you debug if a port is blocked?
**A:**
1. `ss -tulnp | grep <port>` → Is service listening?
2. `telnet <IP> <port>` or `nc -zv <IP> <port>` → Check remote connectivity.
3. Check firewall (`firewalld-cmd --list-all`).
4. Check external firewall/switch rules.
### Q7. What is MTU? How to change it?
**A:**
- MTU = Maximum Transmission Unit (default 1500).
- Jumbo frames (9000) used in storage networks.
- Change:
    `ip link set dev eth0 mtu 9000`
### Q8. Difference between static, DHCP, and DHCP reservation?
**A:**
- Static: Manually assign IP in server config.
- DHCP: Server automatically assigns IP.
- Reservation: DHCP always assigns the **same IP** to a MAC.

# OS Migration Process

Migration means **moving OS + apps + configs + data** from one system to another with **minimum downtime**.

Pre-Migration Planning
Before touching the servers, plan carefully:
- **Identify Scope:**
    - Which server(s) you’re migrating (hostname, IP, apps, DB, services).
    - From OS version → To OS version.
    - Hardware (physical → virtual / cloud).
- **Check Compatibility:**
    - Applications supported on new OS?
    - Kernel modules, drivers, dependencies.
- **Inventory Collection (on old system):**
```
  uname -r              # Kernel version
cat /etc/*release     # Current OS version
df -hT                # Filesystem & usage
mount                 # Mount points
free -h               # Memory
lscpu                 # CPU info
rpm -qa > packages.txt   # (RHEL-based) installed packages
dpkg -l > packages.txt   # (Debian-based)
systemctl list-unit-files --state=enabled > services.txt
crontab -l > cron.txt
ip addr show > network.txt
```
  
  **Backups:**
- Take **full backup** of OS, configs, and data.
- Example:
```
- rsync -avz /etc /root/backup/etc_$(date +%F)
tar czf /root/home_backup.tar.gz /home
```

## Migration Approaches
You usually choose **one** depending on your case:
1. **In-Place Upgrade**
    - Upgrade existing system to newer OS.
    - Example: RHEL 7 → RHEL 8 using `leapp`.
    - Risky but faster.
2. **Fresh Installation + Data Migration (Recommended)**
    - Install new OS cleanly.
    - Reinstall packages and restore configs/data.
    - Safer, fewer legacy issues.

### Migration Steps (Fresh Install + Data Move)
#### Step 1 – Prepare Target System
- Install new OS (RHEL/Ubuntu/Rocky etc.)
- Partition disks properly. Example (for RHEL):
    `lsblk fdisk /dev/sda mkfs.xfs /dev/sda1 mount /dev/sda1 /mnt`
- Create mount points (`/opt`, `/data`, etc.).
#### Step 2 – Configure Networking
```
nmcli con add con-name ens33 ifname ens33 type ethernet ip4 192.168.1.100/24 gw4 192.168.1.1
nmcli con mod ens33 ipv4.dns "8.8.8.8 8.8.4.4"
nmcli con up ens33
```
#### Step 3 – Update & Install Packages

`dnf update -y; dnf install -y <package_name>`
(or `apt-get` for Debian-based).
#### Step 4 – Copy Users, Groups, Crontabs
From old system:
```
getent passwd > passwd.old
getent group > group.old
getent shadow > shadow.old
crontab -l > cron.old
```

Transfer to new system (use caution – may merge instead of overwrite).
#### Step 5 – Data Migration

Use `rsync` (keeps permissions):
```
rsync -avz --progress -e ssh /data/ user@newserver:/data/
rsync -avz --progress -e ssh /etc/ user@newserver:/etc_backup/
```
#### Step 6 – Restore Services
- Copy service configs (e.g., Apache, Nginx, MySQL).
- Re-enable services:
    `systemctl enable httpd; systemctl start httpd`
#### Step 7 – Verify Applications
- Test web apps, databases, scripts.
- Check logs:
    `journalctl -xe tail -f /var/log/messages`

#### Validation & Post-Migration
- Ensure hostname, networking, firewall, and SELinux are configured.
- Verify storage mounts in `/etc/fstab`.
- Confirm backup jobs, monitoring agents, cron jobs running.
- Test user login, SSH access, and application performance.
#### Rollback Plan
Always have a **fallback**:
- Keep old server powered off but not wiped.
- If migration fails, switch DNS/IP back to old server.

✅ **In simple words:**
- Migration = move apps, configs, and data to a new OS.
- Best method = install fresh OS, copy over data & configs, reinstall apps.
- Always backup, test, and keep rollback plan.


### In-Place Migration Steps

#### Pre-Migration Tasks
- **Check current version**
    `cat /etc/*release uname -r`
- **Update all packages to latest**
```
yum update -y         # RHEL 7 / CentOS 7
dnf update -y         # RHEL 8 / Rocky / Alma
apt update && apt upgrade -y   # Ubuntu/Debian
```
- **Backup**
    - Take full system backup (configs, DB, apps).
```
rsync -avz /etc /root/etc_backup_$(date +%F)
tar czf /root/home_backup.tar.gz /home
```
- **Check disk space**
    `df -h /`    
    Ensure at least **5–10 GB free** on `/var` and `/boot`.
#### RHEL / CentOS In-Place Upgrade (Example: RHEL 7 → 8)
1. **Install Leapp upgrade tool**
    `yum install leapp
    `leapp-repository -y`
2. **Pre-upgrade check**
    `leapp preupgrade`
    - This generates a report: `/var/log/leapp/leapp-report.txt`.
    - Fix warnings (unsupported drivers, removed packages).
3. **Run upgrade**
    `leapp upgrade`
    - This downloads RHEL 8 packages.
4. **Reboot into upgrade initramfs**
    `reboot`
    System boots into special upgrade mode, applies changes, then reboots again into RHEL 8.
5. **Verify**
    `cat /etc/redhat-release uname -r`
#### Post-Migration Tasks
- Verify OS version, kernel, and apps:
```
  cat /etc/*release
systemctl list-units --failed
```
- Check networking, storage mounts (`/etc/fstab`), and services.
- Validate application performance.
- Remove obsolete packages (packages that are no longer maintained or supported in current updated system):
```
yum autoremove -y     # RHEL/CentOS
apt autoremove -y     # Ubuntu/Debian
```

#### Key Differences: Fresh vs In-Place
| Method        | Pros                              | Cons                                 |
| ------------- | --------------------------------- | ------------------------------------ |
| Fresh Install | Clean, stable, less legacy issues | Takes longer, must migrate data      |
| In-Place      | Faster, apps/configs remain       | Risk of broken deps, rollback harder |

✅ **In simple words:**  
In-Place upgrade = upgrade existing OS to new version using `leapp` (RHEL) or `do-release-upgrade` (Ubuntu). Always update, back up, run pre-checks, upgrade, reboot, then verify.



# Cloud Migration - WIP

# What is Tuning in Linux?
- **Tuning** means adjusting Linux system settings to improve **performance, stability, or resource usage** for specific workloads.
- It’s like “fine-tuning” a car engine — Linux works fine out of the box, but with tuning you make it run **better for your environment**.
## Areas of Linux Tuning
### CPU Tuning
#### What is CPU Tuning?
CPU tuning means adjusting how your Linux system uses its CPU cores to **get better performance** for different kinds of workloads.
- If you run **databases / applications that need fast response**, you want the CPU to react quickly.
- If you run **batch jobs or background work**, you want efficient CPU usage.
- Tuning helps balance **speed vs. power saving**.
#### 🔑 Key CPU Tuning Areas
##### 1. **CPU Governor (Power Mode)**
The governor decides how CPU frequency changes:
- `performance` → CPU always runs at max speed ⚡ (best for servers, databases).
- `powersave` → CPU runs slower to save energy 🔋.
- `ondemand` / `schedutil` → CPU speed goes up/down depending on load (default in many distros).
👉 **Command** to check and set governor:
```
# Check current governor
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

# Set to performance
echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```
##### 2. **CPU Affinity (Pinning a Process to CPU)**
Sometimes it’s better to run a process on **specific cores** for stability or performance.
👉 **Command:**
```
# Run a program on CPU 0 and 1 only
taskset -c 0,1 myprogram
```
👉 **Check running process CPU affinity:**
`taskset -cp <PID>`
##### 3. **Priority of Processes (nice & renice)**
The CPU scheduler decides which process gets more CPU time.
- **nice value**: lower = higher priority.
- Default nice = 0. Range: `-20` (highest) to `19` (lowest).
👉 **Commands:**
```
# Start a process with low priority
nice -n 10 myprogram

# Change priority of a running process
renice -n -5 -p <PID>
```
##### 4. **IRQ Balance (Handling Hardware Interrupts)**
For servers, especially with multiple CPUs, distributing **hardware interrupts (IRQ)** across CPUs avoids bottlenecks.
👉 **Check IRQ distribution:**
`cat /proc/interrupts`

👉 **Enable/disable irqbalance service:**
```
systemctl status irqbalance
systemctl stop irqbalance
systemctl start irqbalance
```
##### 5. **tuned (Automatic Tuning Profiles)**
Linux provides a service called **tuned** to apply best practices automatically.
👉 **Commands:**
```
# Install tuned
sudo yum install tuned   # (RHEL/CentOS)
sudo apt install tuned   # (Ubuntu/Debian)

# Start service
sudo systemctl enable --now tuned

# See available profiles
tuned-adm list

# Apply performance profile
sudo tuned-adm profile throughput-performance
```
✅ **In short:**
- **CPU Governor** → controls speed/power.
- **taskset** → bind process to CPU.
- **nice/renice** → control priority.
- **irqbalance** → distribute interrupts.
- **tuned** → pre-defined performance profiles.
### Memory Tuning
### I/O and Disk Tuning
### Network Tuning
### Service and Daemon Tuning

### Tools for Tuning

### How do we Tune







# THP (Transparent huge pages)
What are Huge Pages?
- Normally, memory in Linux is managed in **small blocks** called **pages** (usually **4 KB** in size).
- For big applications (like databases), using so many small pages can waste CPU time managing them.
- **Huge Pages** are **much larger memory pages** (usually **2 MB or 1 GB**).
- Using bigger pages = fewer page lookups = **better performance**.
What are Transparent Huge Pages (THP)?
- **Transparent Huge Pages (THP)** is a **Linux kernel feature** that **automatically manages huge pages for you**.
- “Transparent” means: Applications don’t need to be rewritten to use huge pages.
- The kernel decides when and how to use them.

### How to Check THP Status
Run:
`cat /sys/kernel/mm/transparent_hugepage/enabled`
You might see something like:
`[always] madvise never`
- `[always]` → THP always on
- `madvise` → on only if apps request it
- `never` → THP disabled

# Physical server (Dell Poweredge) WIP

# Footnotes


---
[^1]:## 🔹 What is `insmod`?
	- `insmod` = **insert module**.  
	- In **Linux**: it loads a kernel module into the running kernel.
	- In **GRUB (bootloader)**: it loads a GRUB module into memory so GRUB can use its features.
	---
	## 🔹 `insmod` in GRUB
	When your system is at the GRUB command line, GRUB itself is very minimal.
	- By default, it doesn’t know how to handle all commands or filesystems.
	- That’s why you **load modules** with `insmod`.
	- `insmod normal` → loads the **normal module**, which allows GRUB to switch into its **normal mode** (with menus, configs, etc.), instead of staying in rescue mode.
	- Other examples:
	    - `insmod linux` → lets GRUB load Linux kernels.

---
[^2]: ### 🖥️ What is Nice Value?
	
	- The **Nice value (NI)** tells the Linux kernel how "nice" a process is to other processes when sharing CPU time.
	- It’s basically a **priority hint** for the CPU scheduler.
	### 📊 Range of Nice Values:
	- Values go from **-20 to +19**:
	    - **-20** → Highest priority (least nice to others, grabs CPU aggressively).
	    - **0** → Default value for most processes.
	    - **+19** → Lowest priority (very nice to others, only uses CPU when free).
	### ⚙️ How it Works
	
	- The lower the nice value, the **more CPU priority** the process gets.
	- The higher the nice value, the **less CPU priority** it gets.
	- Example:    
	    - A process with **NI = -10** will get more CPU than a process with **NI = +10** when both are competing.
	### 🔧 Commands
	
	- **Check nice value** in `top` → look at the **NI** column.
	- **Start a process with a nice value**:
	    `nice -n 10 mycommand`
	    (runs `mycommand` with NI=10 → lower priority)
	- **Change nice value of a running process (requires root for negative values):**
	    `renice -n -5 -p 1234`
	    (changes process with PID 1234 to NI = -5 → higher priority)
	
	✅ **Quick recall:**
	- **NI = Nice value** in `top`.
	- **-20 = greedy**, **+19 = polite**.
	Adjusted using `nice` (at start) or `renice` (afterwards).

---
[^3]: ## 🧟 Real-Life Zombie Scenario
	
	### Scenario
	You’re managing an **application server** (say Apache or Tomcat).
	- Apache spawns child processes (workers).
	- One child process finishes execution (handled a request and exited).
	- Normally, the **Apache parent process** should call `wait()` to collect the child’s exit status.
	- If the parent doesn’t (maybe due to a bug, misconfiguration, or resource overload) → that child remains as a **Zombie**.
	### What you’d see
	`ps -ef | grep defunct`
	Output:
	`apache  2345  1200  0 ? Z 00:00:00 [httpd] <defunct>`
	- `<defunct>` = Zombie.
	- Parent PID = `1200` (main Apache process).
	### Why it happened
	- Parent (Apache master) didn’t reap the child.
	- Could be a bug or resource overload.
	### How you fixed it
	- First, check if it’s **temporary** (they usually clear quickly).
	- If they **pile up** and don’t go away →
	    - Restart parent process:
	        `systemctl restart httpd`
	    - Or send signal to force it to clean up:
	        `kill -HUP 1200`
	---
	## 👶 Real-Life Orphan Scenario
	### Scenario
	You’re working on a server and a user launches a **long-running job** in the background:
	`./backup.sh &`
	Then the user logs out of SSH.
	- Normally, when a parent shell (bash) exits, it should clean up.
	- But the job (`backup.sh`) is still running.
	- Since the parent shell (bash) is gone, `systemd` adopts the job → it becomes an **Orphan**.
	### What you’d see
	`ps -ef | grep backup.sh`
	Output:
	`root  3456  1  0 12:30 ? 00:00:20 backup.sh`
	- PPID = `1` → orphaned, now managed by systemd.
	- Process still alive and doing work.
	### Why it happened
	- User logged out, parent shell gone.
	- Child didn’t die, so it got reparented.
	### How you fixed it
	- Usually **no action needed** → systemd manages orphans fine.
	- If it’s unwanted:
	    `kill 3456`
	- If you want the job to **survive logout intentionally**, you’d run it using:
	    `nohup ./backup.sh & disown`
	---
	
	 📝 Summary (Workplace Perspective)
		
| Situation     | What Caused It                                                                       | What You Saw                            | Fix                                             |
| ------------- | ------------------------------------------------------------------------------------ | --------------------------------------- | ----------------------------------------------- |
| **Zombie** 🧟 | Parent process (like Apache, MySQL, etc.) didn’t `wait()` for child                  | `<defunct>` in `ps`, `Z` state          | Restart parent, or kill parent so systemd reaps |
| **Orphan** 👶 | Parent shell/app exited while child still running (like user logs out during backup) | Process shows **PPID=1**, still running | No issue, systemd manages. Kill if unwanted     |

[^4]: ## What is `wa` (iowait)?
	- **iowait** = the percentage of CPU time spent **waiting for I/O operations (disk, network, etc.) to complete**.
	- Specifically, it’s about **disk I/O** most of the time.
	- While CPU is in `wa`, it’s basically idle, doing nothing, just waiting for data to come from or go to disk.
	---
	## 📊 Where you see it
	Run `top` or `mpstat`:
	`%Cpu(s):  5.0 us,  1.0 sy,  0.0 ni, 70.0 id, 24.0 wa,  0.0 hi,  0.0 si,  0.0 st`
	Here:
	- **`wa = 24.0`** → 24% of CPU time is wasted waiting on disk I/O.
	---
	## ⚡ Real-Life Scenario
	
	Imagine your database server (MySQL/Oracle) runs a heavy query:
	- CPU asks disk: _“Hey, give me this data block.”_
	- Disk is slow (maybe under load, or spinning HDD instead of SSD).
	- CPU can’t proceed → it just sits idle in **iowait** until disk responds.
	So if `wa` is high, it usually means:
	- **Disk bottleneck** (too many reads/writes, or slow disks).
	- Or **bad query / application doing too much disk I/O**.
	---
	## 🔧 How to Diagnose High iowait
	
	1. Check CPU stats:
	    `top mpstat 1`
	    Look at `wa` percentage.
	2. Check disk activity:
	    `iostat -x 1`
	    - `await` = avg wait time per I/O request.
	    - `svctm` = service time of disk.
	    - `util` = % time disk was busy.  
	        High values mean disk is overloaded.
	3. Check which processes are causing I/O:
	    `iotop`
	    (shows per-process disk usage).
	---
	## ✅ Fixes (Real-World)
	
	- **If app issue** → tune queries, reduce I/O heavy operations.
	- **If hardware bottleneck** →
	    - Move to SSD/NVMe.
	    - Add more disks (RAID-10, striping).
	    - Use multipathing for SAN storage.
	- **If temporary** → monitor with tools like `sar`, `iostat`.
	---
	## 📝 Quick Recall
	
	- **`wa` = CPU waiting on disk I/O.**
	- High `wa` = disk is too slow or overloaded.
	- Tools to check → `top`, `mpstat`, `iostat`, `iotop`.
	- Fix = optimize app OR improve disk subsystem.

---

[^5]: ### **`hi` = hardware interrupts**
	
	- Hardware devices (network card, disk controller, keyboard, etc.) interrupt CPU to say _“I have data for you!”_.
	- When CPU is busy handling these, it shows up under `hi`.
	- Normal = very low (<1%).
	- **High `hi`** = maybe faulty/overloaded device driver or hardware flooding CPU with interrupts.
	
	📌 **Scenario:**
	- A server’s NIC (network card) receives a flood of packets.
	- CPU keeps servicing hardware interrupts instead of running user processes.
	- You’ll see `hi` spike.
	- **Fix:** enable interrupt coalescing, update drivers, or upgrade NIC hardware.
	  
  ---
[^6]: ## IRQ 
	
	`hi` (hardware interrupts) is all about **IRQs**.  
	Let’s break it down simply first, then I’ll give you a real-world picture.
	
	---
	## 🔎 What is IRQ?
	
	- **IRQ = Interrupt Request.**
	- It’s like a **signal sent by a hardware device to the CPU** that says:  
	    👉 “Hey CPU, stop what you’re doing, I need attention now!”
	---
	## 🖥️ Real Examples of IRQs
	
	- **Keyboard**: When you press a key, it sends an IRQ to the CPU so it knows you typed something.
	- **Mouse**: Moving/clicking sends IRQs.
	- **Disk**: When data is ready to read/write, disk controller sends an IRQ.
	- **Network Card**: When a packet arrives, NIC sends an IRQ.
	---
	## ⚙️ How IRQ Works
	1. CPU is busy running a program.
	2. Device (say, NIC) raises an **IRQ line**.
	3. CPU pauses current task briefly and runs the **Interrupt Handler** (small kernel code for that device).
	4. After handling, CPU goes back to what it was doing.
	---
	## 📟 IRQ Numbers
	Each device gets an **IRQ number** → like a roll number in school.
	- Old PCs: limited IRQs (0–15).
	- Modern systems: thousands of virtual IRQs (with **APIC**).
	- You can see them in Linux:
	    `cat /proc/interrupts`
	    Example output:
	```
```
	               CPU0       CPU1
24:    1000000          0  IR-PCI-MSI  eth0
25:     500000     500000  IR-PCI-MSI  nvme0

```
	```    Here `eth0` (network card) uses IRQ 24, `nvme0` (disk) uses IRQ 25.
---

[^7]: 

[^7]: 

[^7]: 

[^7]: 

[^7]: 

[^7]: 

[^7]: 

[^7]: 

[^7]: 

[^7]: 
