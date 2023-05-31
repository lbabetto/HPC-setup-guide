# HPC-setup-guide
Setup guide for an InfiniBand HPC cluster based on Dell PowerEdge nodes running Ubuntu Server 20.04 LTS. We will use Slurm as a job scheduler.

> **Warning**:
> if you have a ConnectX-3 InfiniBand card, you **must** use Ubuntu Server 20.04. Later versions of Ubuntu no longer support drivers for these cards. If you have a ConnextX-4 card, this should not be a problem and you can upgrade to newer versions of the OS. Check compatibility of your card before continuing!

As an example, the following guide has been written with the following hardware configuration:
* Dell PowerEdge R620 master node (2x 8-core Intel Xeon E5-2680 w/ 128 GB RAM, RAID support)
* 4x Dell PowerEdge C6220 slave nodes (2x 8-core Intel Xeon E5-2680 w/ 256 GB RAM)
* Mellanox InfiniBand switch

# 1. Before starting
1. Create at least 1 USB key with [Ubuntu Server 20.04 LTS](https://ubuntu.com/download/server)
>**Note**:
> [Rufus](https://rufus.ie/en/) may sometimes cause issues, it is recommended to set up the stick with [UNetbootin](https://unetbootin.github.io/)
2. If the servers have RAID cards, you need to configure the disks in the RAID BIOS settings first, otherwise the installer will not be able to see the disks. For the Dell PowerEdge R620 master node follow [this](https://www.dell.com/support/kbdoc/en-uk/000183636/poweredge-how-to-configure-raid-with-device-settings-lifecycle-controller?lwp=rt) guide
3. Boot from USB and install the OS with the default options (1 single partition taking up all the disk space, untoggle the LVM checkmark)
4. Enable OpenSSH in settings, leave everything else as default

# 2. Configuring IPoIB (IP over InfiniBand)
This is necessary for being able to connect between nodes via ssh using the InfiniBand interface. The following must be done on all nodes:
1. `sudo apt install rdma-core opensm infiniband-diags ibverbs-utils`
2. Check that cards are correctly recognised with the commands `ibstat` or `ibv_devinfo`
3. Allow non-root users to use IB: `sudo chmod go+rw /dev/infiniband/umad0`
4. Check under which name the IB card is set with `ifconfig -a`. Should read something like this:

```
eno1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.16.27.31  netmask 255.255.255.0  broadcast 172.16.27.255
        inet6 fe80::1a03:73ff:feff:b45a  prefixlen 64  scopeid 0x20<link>
        ether 18:03:73:ff:b4:5a  txqueuelen 1000  (Ethernet)
        RX packets 922145  bytes 454874133 (454.8 MB)
        RX errors 0  dropped 16504  overruns 0  frame 0
        TX packets 1402282  bytes 546407927 (546.4 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
        device memory 0x91a20000-91a3ffff

eno2: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        ether 18:03:73:ff:b4:5b  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
        device memory 0x91a00000-91a1ffff

ibp3s0: flags=4163<BROADCAST,MULTICAST>  mtu 4092                                         <---
        unspec 80-00-02-08-FE-80-00-00-00-00-00-00-00-00-00-00  txqueuelen 256  (UNSPEC)  <---
        RX packets 0  bytes 0 (0.0 B)                                                     <---
        RX errors 0  dropped 0  overruns 0  frame 0                                       <---
        TX packets 0  bytes 0 (0.0 B)                                                     <---
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0                        <---

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 11485  bytes 1888342 (1.8 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 11485  bytes 1888342 (1.8 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```
  
5. Assign the IP address to each IB card manually: `sudo ipconfig ibp3s0 10.0.0.11/24`, where `10.0.0.11` must be a unique IP address for each node (for convenience, use the same subnet and just change the last number...)
6. If you want this to be set at startup, create the file `/etc/systemd/network/ibp3s0.network` file with the following content (`Name` and `Address` must be tailored to your specific situation):
```
[Match]
Name=ibp3s0
Address=10.0.0.11
```
7. Check that the connection is correctly established with the `route` command. The IB network should show up
8. You should now be able to `ssh` between the various nodes using IPoIB

# 3. Assign static IP addresses 
If your machines are also connected to an Ethernet switch, it's a good idea to assign static IP addresses to each machine, so that if you want to connect directly to a specific node, you will always know its IP address.
1. Identify the currently used network interface with `ip addr`. Look for the Ethernet interface in `BROADCAST` state:
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000   <---
    link/ether 18:03:73:ff:b4:5a brd ff:ff:ff:ff:ff:ff
    inet 172.16.27.31/24 brd 172.16.27.255 scope global eno1
       valid_lft forever preferred_lft forever
    inet6 fe80::1a03:73ff:feff:b45a/64 scope link
       valid_lft forever preferred_lft forever
3: eno2: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN group default qlen 1000
    link/ether 18:03:73:ff:b4:5b brd ff:ff:ff:ff:ff:ff
4: ibp3s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 2044 qdisc fq_codel state UP group default qlen 256
    link/infiniband 80:00:02:08:fe:80:00:00:00:00:00:00:50:6b:4b:03:00:80:34:41 brd 00:ff:ff:ff:ff:12:40:1b:ff:ff:00:00:00:00:00:00:ff:ff:ff:ff
    inet 10.0.0.11/24 brd 10.0.0.255 scope global ibp3s0
       valid_lft forever preferred_lft forever
    inet6 fe80::526b:4b03:80:3441/64 scope link
       valid_lft forever preferred_lft forever
```
2. Identify the current gateway with `route -n`:
```
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.16.27.5 <-- 0.0.0.0         UG    0      0        0 eno1 <--
10.0.0.0        0.0.0.0         255.255.255.0   U     0      0        0 ibp3s0
172.16.27.0     0.0.0.0         255.255.255.0   U     0      0        0 eno1
```
3. Edit the `/etc/netplan/*.yaml` file and edit the info about the network interface, setting `dhcp4: no` and adding the required addresses, gateway, and nameservers (for DNS purposes, you can use 8.8.8.8 and 1.1.1.1):
```
# This is the network config written by 'subiquity'
network:
  ethernets:
    eno1:   <--- Currently used network interface
      dhcp4: no   <--- Must be *no*
      addresses:
        - 172.16.27.31/24   <--- The IP address we want
      gateway4: 172.16.27.5 <--- Default gateway
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
    eno2:
      dhcp4: true
  version: 2
```
4. Apply the changes: `sudo netapp apply`

# 4. Set up Wake On Lan
If your cluster is located in a location difficult to access (a remote datacenter), it's always a good idea to enable Wake On Lan (WOL). This enables you to restart nodes remotely (for example, after a power outage) without requiring physical access to the servers.
> **Note**
> : WOL magic packets can only be sent from a machine on the same physical network as the "sleeping" one. Does not work with a VPN
1. Create the `/etc/systemd/system/wol.service` file with the following content (make sure the network interface is the connected one!):
```
[Unit]
Description=Enable Wake On Lan

[Service]
Type=oneshot
ExecStart = /usr/sbin/ethtool --change eno1 wol g   <--- we are connected via Ethernet on eno1

[Install]
WantedBy=basic.target
```
> **Note**
> : make sure the path to `ethtool` is correct. Check with `which ethtool` if you are not sure!
2. If you want to enable WOL without having to reboot the server: `sudo ethtool –change eno1 wol g`
3. Enable the service: `sudo systemctl enable wol.service`

# 5. Set up shared folders with NFS
This step will allow us to "propagate" folders from the master to the slave nodes. This is especially useful for syncing home directories and program installation folders.
1. **All nodes**: edit the `/etc/hosts` file and add a line for each node in the cluster using the IPoIB address:
> **Note**
> : comment out the 127.0.1.1 entry, otherwise the node will try to use this when attempting a connection.
```
127.0.0.1 localhost
#127.0.1.1 snorlax-01     <---

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

# Add all the cluster nodes to the Host list using IPoIB addresses
10.0.0.10 snorlax-master <---
10.0.0.11 snorlax-01     <---
10.0.0.12 snorlax-02     <---
10.0.0.13 snorlax-03     <---
10.0.0.14 snorlax-04     <---
```
2. **Master node**: `sudo apt install nfs-server`
3. **Slave nodes**: `sudo apt install nfs-client`
4. **All nodes**: create the folders you want to sync (e.g., `/shared`)
5. **Master node**: edit the `/etc/exports` file adding the following line (add more lines if you are syncing more folders): `/shared *(rw,sync)`
6. **Master node**: restart the NFS service with `sudo service nfs-kernel-server restart`
7. **Slave nodes**: edit the `/etc/fstab` file and add the following line (adjust for your specific configuration): `snorlax-master:/shared    /shared    nfs`
8. **Slave nodes**: remount all partitions with `sudo mount -a`
> **Note**
> : this step must be done everytime the nodes are rebooted!
9. Check that folders are correctly synced (create a file on the master node and see if it propagates to the slaves), and set the required permissions with the `chmod` command (otherwise slave nodes may not be able to access the shared folder contents)

# 6. Munge setup (necessary for Slurm)
1. **All nodes**: `sudo apt-get install -y libmunge-dev libmunge2 munge`
2. **Master node**: `sudo dd if=/dev/urandom bs=1 count=1024 | sudo tee /etc/munge/munge.key`
3. **Master node**: `sudo chown munge:munge /etc/munge/munge.key`
4. **Master node**: `sudo chmod 400 /etc/munge/munge.key`
5. **Master node** (once for each slave node): `scp /etc/munge/munge.key <node-user>@<node-ip>:/etc/munge/munge.key`
6. **All nodes**: edit the `/etc/passwd` file and modify the `munge` entry to: `munge:x:501:501::/var/run/munge;/sbin/nologin`
> **Note**
> : if problems arise, make sure the `munge` user has access to all its folders (`/etc/munge`, `/var/log/munge`, `/var/lib/munge`, `/run/munge`)
7. **All nodes**: `sudo systemctl enable munge`
8. **All nodes**: `sudo systemctl start munge`

# 7. Slurm setup
1. **All nodes**: `sudo apt-get install -y slurm-wlm`
2. **All nodes**: make sure the following folders and files exist. If not, create them:
  - `/etc/slurm-llnl`
  - `/var/spool/slurm`
  - `/var/log/slurm_jobacct.log`
3. Apply `sudo chown -R slurm:slurm <FOLDER>` to each folder and file in the previous point
4. Generate a `slurm.conf` file using the [online configuration tool](https://cluster.hpcc.ucr.edu/~jhayes/slurm/19.05.0/configurator.html), according to your cluster configuration. A few things to keep in mind:
  - Set your master node's hostname in `SlurmctldHost`
  - Set `CPUs` as appropriate, and optionally `Sockets`, `CoresPerSocket` and `ThreadsPerCore` (use `lscpu` to find out what you have exactly)
  - Set `StateSaveLocation` to `/var/spool/slurm`
  - Set `MpiDefault` to `MPI-PMI2` if you are using OpenMPI 3.1.4 or later
  - Set `ProctrackType` to `cgroup`
  - Make sure `SelectType` is set to `cons_tres` and `SelectTypeParameters` to `CR_CPU_Memory`
  - Set `JobAcctGatherType` to `Linux` and `AccountingStorageType` to `FileTxt`
5. **All nodes**: copy the generated `slurm.conf` file to `/etc/slurm-llnl`.
> **Note**
> : the `slurm.conf` file must be exactly the same in all nodes!
6. **All nodes**: create the `/etc/slurm-llnl/cgroup.conf` file with the following content:
```
CgroupAutomount=yes
CgroupReleaseAgentDir="/etc/slurm/cgroup"
ConstrainCores=yes
ConstrainDevices=yes
ConstrainRAMSpace=yes
```
7. **Master node**: `sudo touch /var/slurmctld.pid; sudo chown slurm:slurm /var/slurmctld.pid`
8. **Slave nodes**: `sudo touch /var/slurmd.pid; sudo chown slurm:slurm /var/slurmd.pid`
9. **Master node**: add the line `User=slurm` to `/lib/systemd/system/slurmctld.service`
10. **Slave nodes**: add the line `User=root` to `/lib/systemd/system/slurmcd.service`
11. **All nodes**: add relevant users to the `slurm` group: `sacctmgr create user name=<USERNAME> account=<GROUP>`
12. **Master node**: `sudo systemctl enable slurmctld`
13. **Master node**: `sudo systemctl start slurmctld`
14. **Slave nodes**: `sudo systemctl enable slurmd`
15. **Slave nodes**: `sudo systemctl start slurmd`
> **Note**
> : if the service cannot start, there might be issues with the ownership of PID files. Try the following:
> 1. Create the `/etc/tmpfiles.d/slurm.conf` file with the following content: `d /run/slurm 0770 root slurm -`
> 2. Edit the `slurm.conf` file and update the new PID file locations (`/run/slurm/...`)
> 3. Edit the `/lib/systemd/system/slurm*.service` files with the same info
> 4. Reboot all systems
16. Confiure `prolog`, `taskprolog` and `epilog` scripts in `/etc/slurm-llnl` if you need something to be done at the start/end of each job (edit the `slurm.conf` file indicating the path to the scripts if you use them). For example, the following files create temporary directories on each node at the start of a job, export a `$TMPDIR` environment variable accessible within the slurm script, and deletes the temporary folder at the end of a job (even if it crashed):
<details>
  <summary>prolog</summary>
  
  ```
  #!/bin/bash
  scratch_dir=/scratch/${SLURM_JOB_USER}/${SLURM_JOB_ID}
  /bin/mkdir -p ${scratch_dir}
  /bin/chmod 700 ${scratch_dir}
  /bin/chown ${SLURM_JOB_USER} ${scratch_dir}
  ```
</details>
<details>
  <summary>task_prolog</summary>
  
  ```
  #!/bin/bash
  scratch_dir=/scratch/${SLURM_JOB_USER}/${SLURM_JOB_ID}
  echo "export TMPDIR=${scratch_dir}"
  ```
</details>
<details>
  <summary>epilog</summary>
  
  ```
  #!/bin/bash
  scratch_dir=/scratch/${SLURM_JOB_USER}/${SLURM_JOB_ID}
  /bin/rm -rf ${scratch_dir}
  ```
</details>

# 7. Configure environment modules
The Modules package is a tool that simplifies shell initialization and lets users easily modify their environment during a session using modulefiles. This enables users to set up PATH exports and environment variables automatically for use with specific programs.
1. Download the desired version of environment modules from the [official website](https://modules.sourceforge.net/)
2. Install TCL via `sudo apt install tcl-dev`
3. Enter the program folder and configure the setup according to where you want to put the modulefiles: `./configure --prefix=/shared/modules-5.3.0 --modulefilesdir=/shared/modules-5.3.0/modulefiles`
4. Run `make` and `make install`
5. Enable the initialization of modules at startup via a symbolic link: `sudo ln -s PREFIX/init/profile.sh /etc/profile.d/modules.sh`
6. Create a `modulefile` for each program you need to configure. Example for Orca 5.0.4 with OpenMPI-4.1.4:

<details>
  <summary>/shared/modules-5.3.0/modulefiles/orca/orca-5.0.4</summary>
  
  ```
  #%Module1.0#####################################################################
  ##
  ## module to load orca-5.0.4
  ##
  proc ModulesHelp { } {
          global version prefix

          puts stderr "\tLoad orca-5.0.4"
  }

  module-whatis   "Load orca-5.0.4"

  # for Tcl script use only
  set     version         5.0.4
  set     prefix          /shared/orca_5_0_4_linux_x86-64_openmpi411

  prereq openmpi/openmpi-4.1.4

  prepend-path    PATH            $prefix
  setenv          ORCA_HOME       $prefix
  ```
</details>

8. Modules can now be enabled via the following syntax: `module load orca/orca-5.0.4`

# 8. Install Cockpit for performance metrics
[Cockpit](https://cockpit-project.org/) allows sysadmins to monitor the performance of a HPC cluster in real time, and optionally to carry out maintenance from a web interface. Setup is very easy and should not require any particular configuration steps. Simply install the software on all nodes and enable the service:

`sudo apt install cockpit cockpit-pcp`

`sudo systemctl start cockpit`

The Cockpit web interface can be found at the IP address of the corresponding machine, on port 9090:

`https://172.16.27.30:9090`

Login credentials are the same as for the machine.

> **Note**
> : you will receive a safety warning when visiting the website, as we did not install any certificates for that "web page". You can safely ignore the warning and proceed with the login.
