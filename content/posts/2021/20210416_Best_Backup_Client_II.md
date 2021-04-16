---
title: "Deep Dive: Cloud Backup for Slow Connections, Part II"
date: 2021-04-16T00:00:00-06:00
---

This is the second post in my deep dive series on cloud backup for network attached storage (NAS). In this article, I'll walk through preparing the test machine and the tools I used to evaluate candidate backup clients

If you're not interested in `kvm` or shell scripts and just want the results, you might want to skip to the next part. However, if you're new to setting up virtual machines and ZFS or are interested in doing similar testing, keep reading to see what I did.

### Initial Setup

To model my NAS configuration fairly closely, I set up a virtual machine in KVM running FreeBSD 12.2 RELEASE:
```
# virt-install \
--name freebsd-12.2-gold \
--import \
--memory 1024 \
--vcpus 1 \
--disk FreeBSD-12.2-RELEASE-amd64.qcow2 \
--os-variant freebsd12.0 \
--network bridge:br0 \
--noautoconsole
```

 My VM host is running Ubuntu Server 20.04 and has a network bridge created with `netplan` at `br0`. This command sets up a machine from the prebuilt QEMU image for FreeBSD 12.2 using bridged networking. The `noautoconsole` option prevents the install from hanging without a graphical output. The "gold" machine is cloned to create a clean test rig for each candidate.

#### A Little Housekeeping

Since I'm setting up this VM for the first time, I did a couple things to make it easier to access.

Both SSH and the serial port console (needed to connect from the host using `virsh console`) are disabled by default, so I had to connect using the graphical VNC console.
```
$ ssh -X <container host> # Make sure X11 forwarding is enabled on the container host first.
$ virt-viewer freebsd-12.2-gold
```

The ssh -X command uses X11 forwarding to display the GUI console application on the machine accessing the server. For a one-off test, I could have just used this console for the whole test process, but I prefer using SSH and virsh to manage VMs.

On the virtual machine, I logged in as root and ran the following commands to enable SSH and the serial console:
```
# echo sshd_enable="YES" >> /etc/rc.conf
# echo 'console="comconsole"' >> /boot/loader.conf
```
For reference information about these commands, refer to the [serial console](https://docs.freebsd.org/en_US.ISO8859-1/books/handbook/serialconsole-setup.html) and [SSH](https://docs.freebsd.org/en_US.ISO8859-1/books/handbook/openssh.html) topics in the FreeBSD Handbook.

I also added a user, locked root login, and updated FreeBSD to the latest patch version.

#### Deploying the Test Machine

I then cloned the gold image to create a test machine:
```
# virt-clone -o freebsd-12.2-gold -n freebsd-12.2-backuptest --auto-clone
```

Next, I installed two spare physical hard drives in the VM host to use in my test zpool and passed them through to the test VM:
```
# virsh attach-disk freebsd-12.2-backuptest /dev/disk/by-id/ata-XXXX vdb --config
# virsh attach-disk freebsd-12.2-backuptest /dev/disk/by-id/ata-XXXX vdc --config
```
Virtual disks would work too, but I had some old drives on hand and wanted to model the real-world use as closely as possible. The two drives I used are WD consumer drives that are about a decade old. Regardless, I expect the backup to be limited by my upload speed, not the read speed of the drives.

In the VM, I configured a pool with one mirror vdev containing the two drives.
```
# zpool create tank mirror /dev/vtbd1 /dev/vtbd2
```
>##### Don't do this!
>This command is missing two parameters that you should always include when creating a real vdev:
> * First, always check your drive's physical block size and specify the correct `ashift` value. You usually would need `-o ashift=12` or `13` in this command when adding modern drives with a 4k or 5k block size, respectively. These ancient drives actually have a physical block size of 512, so the default ashift value of 9 is correct. Check yours with `smartctl -i <disk>`, keeping in mind that most drives report a logical blocksize of 512 for compatibility reasons. On Linux, you can skip installing smartmontools and use `lsblk -o name,phy-sec`.
> * Second, you should specify the drives with /dev/diskid/DISK-XXX (on FreeBSD) or /dev/disk/by-id/ata-XXX (on Linux). I used the assigned disk identifiers because the VM doesn't present disk IDs, and I won't ever need to swap or replace the drives in the VM.

I also enabled compression and disabled access times to match my real storage pool and set FreeBSD to mount zpools at startup:
```
# zfs set compression=lz4 tank
# zfs set atime=off tank
# echo 'zfs_enable="YES"' >> /etc/rc.conf
```

Next, I created some datasets in the pool, copied over a representative chunk of data from my NAS, and created a snapshot.
```
# zfs create -o casesensitivity=mixed tank/docs
# zfs create -o casesensitivity=mixed -o recordsize=1M tank/photos
# zfs create -o casesensitivity=mixed -o recordsize=1M tank/audio
# zfs create -o casesensitivity=mixed -o recordsize=1M tank/video
# chown -R ben:ben /tank/*

[...]

 # zfs snapshot -r tank@initial
```

### Test Methodology and Procedures
I tested the following backup programs:
* duplicity (de-duplicated full and incremental backups)
* restic (de-duplicated incremental without needing periodic full backups)
* rclone (simple sync utility with excellent cloud support)
* b2-sync (simple sync utility provided by Backblaze)

Although I didn't test duplicacy, I'm indebted to the widely shared [tests by Gilbert Chen](https://github.com/gilbertchen/benchmarking), the author of duplicacy. Chen covers some of the same variables that I'm interested in and covers duplicity and restic. I'll be interested to see whether I can replicate his results.

I tested three use cases: the initial backup, a series of incremental backups, and a complete restore. In each use case, I measured:
* Time
* Network usage
* Size on disk

I also recorded CPU and memory usage during the backup, but those measurements are primarily to check for issues due to testing in low-spec virtual machine.

To measure performance, I created that runs the backup utility along with the following tools, which are available in the FreeBSD 12.2 base system:
* `time`
* `netstat`
* `vmstat`

Note that you might need to tweak the script below to run the same test on Linux, because the BSD versions of these tools have different options and syntax. Linux users might want to use `sar` instead of `netstat` to measure network utilization.

I created this script to run the backup utility and the monitoring utilities concurrently:
```sh
#!/bin/sh
command="<Your backup command>"

# <Set environment variables for credentials and other data here>

# On Ctrl+C, kill all the processess in the group.
trap 'kill 0' SIGINT

# Do the main command.
time -l $command &&
# After it's done, kill all the processes in the group.
kill 0  &
# Run these commands concurrently with the main command.
vmstat 1 --libxo xml,pretty > ./vmstat.txt &
netstat -I vtnet0 -w 1 --libxo xml,pretty > ./netstat.txt
```

This script makes sure that we stop monitoring right when the backup finishes. To break down the options for each command:
* The `vmstat 1` and `netstat -w 1` commands write an output to a file once per second.
* The `--libxo xml,pretty` argument tells them to output as structured XML data instead of the usual on-screen display, so it's easier to process the data. As it turned out, the XML output needed manual tweaking, so I'd say it's a tossup whether the libxo option is useful in this case.
* The `time -l` option shows some CPU and memory stats in addition to times (somewhat duplicative of vmstat).
* Finally, `netstat -I vtnet0` just means only count the traffic through the VM network interface.

Here are the commands that I plugged into the script above for each backup client:

Duplicity:
```sh
# Backblaze B2 configuration variables
B2_ACCOUNT='<Account ID>'
B2_KEY='<Application key>'
B2_BUCKET='<Bucket name>'

# GPG key
export ENC_KEY='<Last 8 characters of GPG key>'
export PASSPHRASE='<GPG key passphrase>'

command = 'duplicity --encrypt-sign-key $ENC_KEY /tank b2://${B2_ACCOUNT}:${B2_KEY}@${B2_BUCKET}'

# Restore:
duplicity restore --force b2://${B2_ACCOUNT}:${B2_KEY}@${B2_BUCKET} /tank/
```

Restic:
```sh
export B2_ACCOUNT_ID='<Account ID>'
export B2_ACCOUNT_KEY='<Application key>'
export RESTIC_REPOSITORY='b2:<Bucket name>'
export RESTIC_PASSWORD_FILE='/path/to/password/file'

# To back up:
command = 'restic backup /tank'

# To restore:
command = 'restic restore latest --target /tank'
```

Rclone (after initial setup to authorize the account):
```sh
command = 'rclone sync --checksum --log-file ./rclone.log --progress --stats-one-line --transfers 4 --verbose /tank remote:<Bucket name>'

# To restore, switch /tank and remote:<Bucket name>
```

B2 sync (after initial setup to authorize the account):
```sh
# To back up:
command = 'b2 sync --keepDays 30 --replaceNewer --threads 30 /tank b2://<Bucket name>/'

# To restore:
command = 'b2 sync --replaceNewer --threads 30 b2://<Bucket name/ /tank/'
```

### Next Steps

Next up, the test results. Part III will dig into the data from each test and my conclusions.
