---
title: "Deep Dive: Cloud Backup for Slow Connections, Part I"
date: 2021-04-05T00:00:00-06:00
---

In case further proof were needed, last month's story about the [companies that lost irreplaceable data in a literal towering inferno](https://www.polygon.com/22323078/rust-facepunch-fire-eu-datacenters) proves again that everyone with important files needs offsite backups. I don't mean to make fun of the hosting provider or the companies who got caught without backups in multiple locations (OK, maybe them, a little bit). Keeping your data safe from a real-world catastrophe---whether you're managing business infrastructure or New Folders (1) through (23) on your desktop---is hard!

This is the first in a series of posts documenting a project I did to start backing my personal network attached storage (NAS) server to cloud storage. This post outlines my requirements for an effective cloud backup strategy and some challenges I've run into. Future posts will evaluate some different backup applications based on my testing and describe the technical implementation in more depth.

One view I'd like to promote is that everyone has important data, and it's worth the effort to plan for disaster recovery. In my opinion, the considerations and risks are the same whether you're an individual or a large organization. I hope these posts can be a resource for anyone looking for a reliable backup solution.

### It's a Series of Tubes

My biggest challenge to backing up data over the internet is uploading a lot of data with only a little upload bandwidth. My cable internet connection from Charter is rated for 300 Mbit/s down and 30 Mbit/s up, and I usually see a little less in practice. The only other option in my neighborhood is an even slower DSL connection. On the bright side, I'm fortunate not to have a data cap!

Backing up a terabyte of data on my connection would take three days using 100% of my upload bandwidth the entire time. Of course, the solution to this problem is to run the backup once, then use incremental backups that don't require a regular full backup. However, because the connection is [not a big truck](https://youtu.be/f99PcP0aFNE), the incremental process also needs to be efficient to avoid interfering with other traffic on the network.

The other challenge to running cloud backups for me personally is puzzling out what backup client best meets my needs and strikes a good balance between being stable, trusted, and reasonably convenient to use and administer. There are a lot of options for backing up a headless server to a cloud storage provider. If you read a lot of online forums, they all of them seem to have bad tradeoffs.

### Requirements and Considerations

I identified the following requirements for a good cloud backup process:
* Backups use upload bandwidth efficiently. This is measured in two ways:
    * Backups don't disrupt users on the network. If the backup must run during daytime hours, a user should be able to have a video call at the same time without issues.
    * Each incremental backup is done before the next one is scheduled to start. Ideally, they finish overnight.
* Backups are encrypted prior to upload so that the cloud storage provider doesn't have access to the files (for a reminder of why, refer to [this tweet](https://twitter.com/Benjojo12/status/1373707799054712836?s=20)).
* The software used is well documented and easy to use. My NAS is definitely a [pet, not a cow](http://cloudscaling.com/blog/cloud-computing/the-history-of-pets-vs-cattle/), so shell scripts and helper utilities are fair game, but it should be possible for a friend or family member to retrace my steps in case of an emergency.

The following criteria are nice-to-have's:
* Backups are storage efficient and don't grow too much over time. Most cloud storage providers set rates based on both the amount of data stored and the amount transferred, so storing more data costs more.
* Backups are atomic (the data on the disk doesn't change while the backup is running). Ideally, the state of the cloud backup can be mapped to a specific ZFS snapshot.
* The encryption method is confirmed to be reasonably strong, or a standard package like GPG is used.
* The backup client is free and open source. I don't mind paying for commercial software, but if possible I'd prefer to use a free or non-commercial tool.

Of course, the backup plan must have well documented procedures and defined recovery point and time objectives (RPO and RTO), and those procedures should be regularly tested.

Note that I see cloud backups as a supplement to other methods. For example, a cloud backup might be the third copy in a 3-2-1 strategy (three copies, two local, and one offsite). I run local backups by copying my data to a pair of USB hard drives using ZFS replication. My previous solution to offsite backups was to regularly swap the USB drives and keep one of them at my office.

### Next Steps

With my use case and end state goal (the why and the what) defined, the next task was to figure out how best to implement them. Part II will list the options I considered and the steps I took to test them.
