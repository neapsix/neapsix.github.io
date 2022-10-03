---
title: "Artisanal Thin Jails from Scratch on FreeBSD"
date: 2022-10-03T00:00:00-06:00
---

More about FreeBSD virtualization with jails, the native system for running applications in isolated containers. 
This article walks through how to manually set up jails that share an immutable base for common files.

<!--more-->

Note that the built-in jail management tools are a little basic. Programs like `iocage` and `bastille` make this process much easier. I wrote this article becuase I wanted to be comfortable creating and and and troubleshooting jails on my own before automating them.

To run an application in a jail, create a jail and install a copy of the FreeBSD system, sans kernel, into it.
You can use the jail as though it were its own machine with its own configuration, packages, and data.
However, with multiple jails, you have duplicate copies of a standard FreeBSD system and duplicate chores to maintain them all.

You can avoid duplicating the whole system by creating thin jails containing just the unique files, like packages and service-specific configurations.
All thin jails point to a single, shared base jail containing system files and other files common to all your jails.
The basic procedure is to make an empty directory, mount the base jail's filesystem in it as read-only, and mount the unique files over it as a writable layer.

For comparison, Docker on Linux lets you separate persistent data and configuration from the system image using bind mounts.
Jails can do something similar using the `nullfs` tool built into FreeBSD to layer filesystems on top of each other.
The built-in jail management tools are a little basic, but I wanted to know how to set up jails manually before trying something like `iocage` or `bastille`.

Usual disclaimer that I'm not trying to say that jails or Docker containers are better.
They're different tools, and both have benefits and drawbacks.

## Creating Thin Jails from Scratch

To create artisanal thin jails by hand, you need to:

1. Create a base jail.
1. Create a template for your thin jails.
1. Move files you want to be jail-specific from the base jail to the thin jail template.
1. In the base jail, create a skeleton directory structure to mount thin jails over.
1. Clone the thin jail template and mount the base jail and thin jail together into a mountpoint directory.
1. Configure the host to start your thin jail automatically.

### Create a Base Jail

Make a new ZFS dataset and install the jail there, as described [in the FreeBSD Handbook](https://docs.freebsd.org/en/books/handbook/jails/#jails-build).
Make a snapshot of the base jail when you're done.

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

Start the base jail with the four required arguments: directory, hostname, IP address, and command.

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

Next, create a template for the writeable part of the jail.

```console
# zfs create -p zroot/jails/base/13.1-RELEASE-template
```

Determine which files and directories should be jail-specific.
Remove them from the base jail and move them to the template.
I chose directories for configuration files, installed ports, home directories, and directories for dynamic data like log files.

```console
# cd /usr/local/jails/base/13.1-RELEASE-base

# TEMPLATE="/usr/local/jails/base/13.1-RELEASE-template/"

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

The file `/var/empty` doesn't get deleted when you move `/var`, so you need an extra command to remove it manually.

Make a snapshot of the template when you're done.
This directory is the gold master for your future thin jails.

```console
# zfs snap zroot/jails/base/13.1-RELEASE-template@gold
```

### Create a Skeleton Directory Structure to Mount Thin Jails Over

To mount a thin jail over your base jail, you need empty directories to mount the thin jail directories into.
For example, if you mount the `basejail` filesystem and try to mount `thinjail/etc` over it, the `mount` command returns an error if there isn't an empty directory called `basejail/etc`.

You could create empty directories in your base jail where the thin jail directories used to be.
However, it's better to build the directory structure within a subdirectory and create symlinks into that subdirectory for `etc`, `usr`, `var`, and so on.
With this setup, the thin jail files are cordoned off from the base jail files in the resulting union filesystem.

Back in the base jail, create a "skeleton" directory structure to mount the template into.
Make a snapshot when you're done.

```console
# cd /usr/local/jails/base/13.1-RELEASE-base
# mkdir skeleton
# ln -s skeleton/etc        etc
# ln -s skeleton/usr/home   usr/home
# ln -s skeleton/usr/local  usr/local
# ln -s skeleton/root       root
# ln -s skeleton/tmp        tmp
# ln -s skeleton/var        var
```

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

We're ready to create a thin jail using on this base jail and template.
First, clone the thin jail template to a new directory for the thin jail's persistent data.
Note that I'm not using `zfs clone` for this step as some other guides online do.
I'm not sure I see the benefit to having each thin jail dataset linked as a clone to the template dataset.

```console
# zfs send zroot/jails/base/13.1-RELEASE-template@gold | zfs receive zroot/jails/thinjail0/data
```

And create the mountpoint location where the two parts of the jail will be mounted:

```console
# zfs create zroot/jails/.mountpoints; mkdir /usr/local/jails/.mountpoints/thinjail0
```

Each thin jail needs an `fstab` file with instructions to layer the thin jail on top of the read-only base jail.
Create the file `/usr/local/jails/thinjail0/fstab` as follows:

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
      thinjail0/                <- Jail-specific data
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

thinjail0 {
    ip4.addr = 192.168.1.70;
}
```

Tie a bow on everything by adding the following to `rc.conf` so that all jails in your `jail.conf` file are started automatically on boot.

#### `/etc/rc.conf`

```
jail_enable="YES"
```

To start the jail, run `service start jail`. Yay!

### Review the Final Jail

Verify that the jail is running and get its ID.

```console
# jls
   JID  IP Address      Hostname                      Path
     6  192.168.1.70    thinjail0                     /usr/local/jails/.thinpool/thinjail0
```

Start a shell in the jail to check that the filesystems are present and mounted correctly.
You shouldn't be able to edit files in the immutable part of the jail.
You should in in the writable part.

```console
# jexec 6

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

I tried splitting my `jail.conf` file so that `/etc/jail.conf` defined default options, and each jail got its own file in `/etc/jail.conf.d`.
Unfortunately, there are two limitations that make `/etc/jail.conf.d` a lot less useful:

- The jail init script reads files in `/etc/jail.conf.d` only if you also list the name of the jail in `rc.conf`, like `jail_list="thinjail0`. More details [here](https://cgit.freebsd.org/src/commit/?id=7955efd574b9).
- Files in `/etc/jail.conf.d` don't check for global settings in the main `jail.conf` file.
  So, if you separate jail configs into multiple files, you have to repeat the global settings in every single file.
  Which defeats the purpose of global settings.

I also tried keeping my `jail.conf` file alongside all my other jail files and symlinking it to `/etc/jail.conf`, but doing that led to this error when starting the jail.

```console
Starting jails:jail: /etc/jail.conf: Too many levels of symbolic links
```

Not a big deal, but it would be nice if the jail system could work with a symlinked config file.

In case you're wondering, the separate `.mountpoints/thinjail0` directory is necessary.
I tried mounting the base jail into the thin jail's directory and mounting the thin jail again over top of it.
That almost worked, but the thin jail data was read-only.
It's obvious why, in retrospect--I couldn't write to it because the base jail was covering those files up.

I plan to write another article on making thin jails in a _much_ simpler way using automated jail management tools.

## References

I can't take credit for much of this work.
Others already did the hard work of figuring out how to create thin jails.
That said, I hope I was able to advance the conversation a bit by pulling together different sources, using friendlier naming conventions, and validating the steps in 2022.

Thanks and credit to the authors of the following articles:

- <https://jacob.ludriks.com/2017/06/07/FreeBSD-Thin-Jails/> (Jacob Ludriks)
- <https://clinta.github.io/freebsd-jails-the-hard-way/> - I gather that the excellent article above draws from this one
- <https://docs.freebsd.org/en/books/handbook/jails/#jails-application> (FreeBSD Docs Project) - for a cryptic but effective directory layout
- <https://srobb.net/nullfsjail.html> - for a seemingly simpler approach using unionfs
- <https://forums.freebsd.org/threads/thin-jail-woes.54530/> - for warning me off the unionfs approach
