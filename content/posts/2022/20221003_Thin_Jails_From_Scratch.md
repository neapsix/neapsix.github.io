---
title: "Artisanal Thin Jails from Scratch on FreeBSD"
date: 2022-10-03T00:00:00-06:00
---

More about FreeBSD virtualization, this time with jails, the native system for running applications in isolated containers.
This article walks through how to manually set up jails that share an immutable base for common files.

<!--more-->

Note that the built-in jail management tools described here are a little basic.
Programs like `iocage` and `bastille` make this process much easier.
I decided to write about making thin jails from scratch to better understand how they work before I try to automate them.

## Why Use Thin Jails?

To run an application in a jail, create a jail and install a copy of the FreeBSD system, sans kernel, into it.
You can then use the jail as though it were its own machine with its own configuration, packages, and data.
However, each jail contains duplicate copy of a standard FreeBSD system and needs duplicate chores to maintain it.

You can avoid duplicating the whole system by creating thin jails with only the unique files, like packages, application-specific configuration, and user data.
All your thin jails point to a single, shared base jail with system files and everything else that's the same from jail to jail.
The basic procedure to create a thin jail is to make an empty directory, mount the base jail's filesystem in it as read-only, and mount the thin jail's unique files over it as a writable layer.

This approach makes it easier to update or replace the system image for a container, because it keeps the application or user's data separate from the vanilla system files that can be recreated any time.
Thin jails use the `nullfs` tool built into FreeBSD to layer filesystems on top of each other.
For comparison, Docker on Linux does something similar by using bind mounts to layer a file or directory on top of a disposable system image.

Usual disclaimer that I'm not trying to say that jails or Docker containers are better.
They're different tools, and both have benefits and drawbacks.

## Creating Thin Jails from Scratch

To create artisanal thin jails by hand, you need to:

1. Create a base jail.
1. Create a template for your thin jails.
1. Move files you want to be jail-specific from the base jail to the thin jail template.
1. In the base jail, create a skeleton directory structure to mount a thin jail over.
1. Clone the thin jail template and mount the base jail and thin jail together into a mountpoint directory.
1. Configure the host to start the combined jail automatically.

### Create a Base Jail

Make a new ZFS dataset and install the jail there:

```console
# zfs create -o mountpoint=/usr/local/jails zroot/jails
# zfs create -p zroot/jails/base/13.1-RELEASE-base
# bsdinstall jail /usr/local/jails/base/13.1-RELEASE-base
```

Make a snapshot of the base jail when you're done.

```console
# zfs snap zroot/jails/base/13.1-RELEASE-base@install
```

You now have a jail with a complete FreeBSD userland installed into it.

```console
$ cd /usr/local/jails/base/13.1-RELEASE-base
$ ls -F
COPYRIGHT       entropy         libexec/        proc/           sys@
bin/            etc/            media/          rescue/         tmp/
boot/           home@           mnt/            root/           usr/
dev/            lib/            net/            sbin/           var/
```

To verify that it works, start the base jail with the four required arguments: directory, hostname, IP address, and command.

```console
# jail -c path=/usr/local/jails/base/13.1-RELEASE-base \
    mount.devfs \
    host.hostname=basejail \
    ip4.addr=192.168.1.70 \
    command=/bin/sh
```

And see what's mounted in the jail:

```console
# mount
zroot/jails/base/13.1-RELEASE-base on / (zfs, local, noatime, nfsv4acls)
```

So far, so good.

### Create a Thin Jail Template

Next, create a template for the thin jail, the writeable part of the jail that contains files specific to one jail.

```console
# zfs create -p zroot/jails/base/13.1-RELEASE-template
```

Determine which files and directories should be jail-specific.
Remove them from the base jail and move them to the template.
I chose all directories for configuration files, installed ports, user homes, and dynamic data like log files.

```console
# cd /usr/local/jails/base/13.1-RELEASE-base

# set TEMPLATE="/usr/local/jails/base/13.1-RELEASE-template"

# mkdir $TEMPLATE/usr

# mv    etc        $TEMPLATE/etc
# mv    usr/home   $TEMPLATE/usr/home
# mv    usr/local  $TEMPLATE/usr/local
# mv    root       $TEMPLATE/root
# mv    tmp        $TEMPLATE/tmp
# mv    var        $TEMPLATE/var
# chflags 0 var/empty; rm -r var

# unset TEMPLATE
```

You need the extra `chflags` command because the file `/var/empty` doesn't get deleted when you move `/var`.

Make changes as needed to the files in the template.
For example, you might want to set the default route for your thin jails to access the internet:

```console
# cd /usr/local/jails/base/13.1-RELEASE-template

# echo 'default_router="192.168.1.1"' >> ./etc/rc.conf
```

Make a snapshot of the template when you're done.
This directory is the gold master for your future thin jails.

```console
# zfs snap zroot/jails/base/13.1-RELEASE-template@gold
```

### Create a Skeleton Directory Structure to Mount Thin Jails Over

To mount a thin jail over your base jail, you need a tree of empty directories to mount the thin jail directories into.
For example, if you mount the `basejail` filesystem and try to mount `thinjail/etc` over it, the `mount` command returns an error if there isn't an empty directory called `basejail/etc`.

You could create empty directories in your base jail where the thin jail directories used to be, but it's better to build the directory structure within a subdirectory and make symlinks into that subdirectory for `etc`, `usr`, `var`, and so on.
This setup separates the thin jail files from the base jail files in the resulting union filesystem.

Back in the base jail, create a "skeleton" directory structure to mount the template into, then link the directories in the root directory to the directories in the skeleton directory.

```console
# cd /usr/local/jails/base/13.1-RELEASE-base
# mkdir skeleton
# ln -s ./skeleton/etc          etc
# ln -s ../skeleton/usr/home    usr/home
# ln -s ../skeleton/usr/local   usr/local
# ln -s ./skeleton/root         root
# ln -s ./skeleton/tmp          tmp
# ln -s ./skeleton/var          var
```

> #### Note
>
> The first argument to `ln` is a directory relative to the second argument, **not** relative to the current working directory.
> For example, to link /usr/local to /skeleton/usr/local, run either `ln -s ../skeleton/usr/home /usr/home` or `ln -s /skeleton/usr/home /usr/home`.

Make a snapshot when you're done.
The base jail with files removed and the skeleton directory structure added is the gold master for your base jail.

```console
# zfs snap zroot/jails/base/13.1-RELEASE-base@gold
```

Now, the two jail filesystems are as follows:

```console
# ls -F /usr/local/jails/base/13.1-RELEASE-template
etc/    root/   tmp/    usr/    var/

# ls -F /usr/local/jails/base/13.1-RELEASE-base
COPYRIGHT       home@           proc/           tmp@
bin/            lib/            rescue/         usr/
boot/           libexec/        root@           var@
dev/            media/          sbin/
entropy         mnt/            skeleton/
etc@            net/            sys@
```

### Clone the Template and Mount Base and Thin Jails Together

You're ready to create a thin jail using this base jail and template.
First, create the thin jail by adding a zfs dataset and replicating the thin jail template to that dataset.
Note that I'm not using `zfs clone` for this step as some other guides online do.
I'm not sure I see the benefit to having each thin jail dataset linked as a clone to the template dataset.

```console
# zfs create zroot/jails/thinjail0
# zfs send zroot/jails/base/13.1-RELEASE-template@gold | zfs receive zroot/jails/thinjail0/data
```

Next, create the mountpoint location where the two parts of the jail will be mounted:

```console
# zfs create zroot/jails/.mountpoints; mkdir /usr/local/jails/.mountpoints/thinjail0
```

Each thin jail also needs an `fstab` file with instructions to layer the thin jail on top of the read-only base jail.
Create the file `/usr/local/jails/thinjail0/fstab` as follows:

#### `fstab`

```fstab
/usr/local/jails/base/13.1-RELEASE-base  /usr/local/jails/.mountpoints/thinjail0  nullfs  ro  0   0
/usr/local/jails/thinjail0/data  /usr/local/jails/.mountpoints/thinjail0/skeleton  nullfs  rw  0   0
```

Let's take a look at the final layout of these directories.
This is my personal preference.
You can lay yours out in whatever way makes most sense to you.

```
usr/
  local/
    jails/
      base/
        13.1-RELEASE-base       <- Immutable base for shared system files
          - bin/
          - boot/
          - lib/
          [...]
          - skeleton/           <- Empty directory structure to mount into
            - etc/
            - home/
            - usr/
            [...]
        13.1-RELEASE-template   <- Original files to modify
          - etc/
          - home/
          - usr/
          [...]
      thinjail0/                <- Writable jail-specific data
        - data/
          - etc/
          - home/
          - usr/
          [...]
        - fstab
      .mountpoints/
        thinjail0/              <- Union filesystem is mounted here using nullfs
```

### Start Your Thin Jail

To mount these filesystems according to the instructions in your `fstab` file, add the `mount.fstab` argument when you start the jail.
Speaking of...

This time, instead of using the `jail` command to start the jail, create a `jail.conf` file.
You can then start it with `service` and automatically on boot.
Create a file similar to the following one at `/etc/jail.conf`.

Note that you might see a `jail.conf.d` directory there for you already, but don't be tempted to use it.
It's still too limited to be useful to me (see below for details).

#### `/etc/jail.conf`

```
host.hostname = "$name";
path = "/usr/local/jails/.mountpoints/$name";

mount.fstab = "/usr/local/jails/$name/fstab";

exec.start = "/bin/sh /etc/rc";
exec.stop = "/bin/sh /etc/rc.shutdown";
exec.clean;

mount.devfs;

interface = "em0"; # Replace with your network interface.
allow.raw_sockets; # Optional, makes ping work inside the jail.

thinjail0 {
    ip4.addr = 192.168.1.70; # Replace with an IP on the host's subnet
}
```

Tie a bow on everything by adding the following to `rc.conf` so that all jails in your `jail.conf` file are started automatically on boot.

#### `/etc/rc.conf`

```
jail_enable="YES"
```

To start the jail, run `service jail start`. Yay!

### Review the Final Jail

Verify that the jail is running and get its ID.

```console
# jls
   JID  IP Address      Hostname                      Path
     2  192.168.1.70    thinjail0                     /usr/local/jails/.thinpool/thinjail0
```

Start a shell in the jail to check that the filesystems are present and mounted correctly.
You shouldn't be able to edit files in the read-only part of the jail.
You should in the writable part.

```console
# jexec 2

# ls -F
.cshrc          dev/            libexec/        rescue/         tmp@
.profile        entropy         media/          root@           usr/
COPYRIGHT       etc@            mnt/            sbin/           var@
bin/            home@           net/            skeleton/
boot/           lib/            proc/           sys@

# touch /usr/bin/my_new_file; echo $?
touch: /usr/bin/my_new_file: Read-only file system
1

# touch /etc/my_new_file; echo $?
0

# mount
/usr/local/jails/base/13.1-RELEASE-base on / (nullfs, local, noatime, read-only, nfsv4acls)
```

But wait, why is it showing only one mounted filesystem?
This is a security feature.
By default, jails are run with the `enforce_statfs` parameter set to 2, which means that only the root mount point appears when a user runs `mount` inside the jail.
As we saw, the filesystems are correctly mounted.

I recommend leaving this parameter as is, but just to test, I tried setting `enforce_statfs` to a lower value in `jail.conf`.
Now, `mount` returns everything we specified in the `fstab` and `jail.conf` files, as expected:

```console
/usr/local/jails/base/13.1-RELEASE-base on /usr/local/jails/.mountpoints/thinjail0 (nullfs, local, noatime, read-only, nfsv4acls)
/usr/local/jails/thinjail0/data on /usr/local/jails/.mountpoints/thinjail0/skeleton (nullfs, local, noatime, nfsv4acls)
devfs on /usr/local/jails/.mountpoints/thinjail0/dev (devfs)
```

## Caveats and Follow-Ups

On the whole, the from-scratch approach is probably too much work.
I plan to write another article on using automated jail management tools.

I also ran into a few quirks and limitations while testing these steps.
Here are some notes that might save you some time.

### Why Not to Use `jail.conf.d`

I tried splitting my `jail.conf` file so that `/etc/jail.conf` defined default options, and each jail got its own file in `/etc/jail.conf.d`.
Unfortunately, there are two limitations that make `/etc/jail.conf.d` a lot less useful:

- The jail init script reads files in `/etc/jail.conf.d` only if you also list the name of the jail in `rc.conf`, like `jail_list="thinjail0`.
  If you define the jails directly in `/etc/jail.conf`, you can start everything with just `jail_enable="YES"`. More details [here](https://cgit.freebsd.org/src/commit/?id=7955efd574b9).
- Files in `/etc/jail.conf.d` don't check for global settings in the main `jail.conf` file. If you use multiple config files, you have to repeat the global settings in each file.
  Which defeats the purpose of global settings.

### You Can't Use a Symlink for `/etc/jail.conf`

I also tried keeping my `jail.conf` file alongside all my other jail files and making `/etc/jail.conf` a symlink to that file, but that led to this error when starting a jail.

```console
Starting jails:jail: /etc/jail.conf: Too many levels of symbolic links
```

This limitation isn't a big deal, it would be nice if the jail system could follow a symlink to the config file.

### Thin Jail Mountpoints

In case you're wondering, the separate `.mountpoints/thinjail0` directory is necessary.
I tried mounting the base jail into the thin jail's directory and mounting the thin jail again over top of it.
That almost worked, but the thin jail data was read-only.
It's clear why, in retrospect--I couldn't write to it because the base jail was covering those files up.

## References

I can't take credit for most of the steps in this article.
Others already did the hard work of figuring them out.
That said, I hope this article advances the conversation a bit by pulling together different sources, validating the steps in 2022, and using friendlier naming conventions.

Thanks and credit to the authors of the following articles:

- <https://jacob.ludriks.com/2017/06/07/FreeBSD-Thin-Jails/> (Jacob Ludriks)
- <https://clinta.github.io/freebsd-jails-the-hard-way/> - the article above seems to draw from this one
- <https://docs.freebsd.org/en/books/handbook/jails/#jails-application> (FreeBSD Docs Project)
- <https://srobb.net/nullfsjail.html> - for a similar approach using unionfs
- <https://forums.freebsd.org/threads/thin-jail-woes.54530/> - for warning me off of unionfs
