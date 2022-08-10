---
title: "Setting Up Bhyve with Ansible"
date: 2022-07-24T00:00:00-06:00
---

This weekend, I moved the system setup for my virtualization server into Ansible, and I got virtual machines up and running with `bhyve`.
I hit a small bump when combining the two that I wanted to share. I'd also like to rave a bit about my experience trying `bhyve`.
It's amazing!

<!--more-->

For background, I'm migrating this server from sketchy and half-assed to maintainable and well planned out.
I'm also migrating it from Ubuntu to FreeBSD, but note that I'm not trying to bash Ubuntu.
The badness came from how I set it up, not from Ubuntu.

I had already installed FreeBSD and done some manual setup as a proof of concept.
My goal was to automate what I did by recreating it with Ansible and then add the required setup to start running virtual machines with `bhyve`.

### New Machine Setup with Ansible

It was pretty easy to recreate my basic setup for a new machine using Ansible.
I created two playbooks.

The first playbook takes a fresh install with just my user, an ansible user, and `sshd`, and it bootstraps some prerequisites.
It logs in, becomes root with `su`, and uses the raw command module to `pkg install python`. 
Without the Python interpreter, running raw commands is all Ansible can do.
It then installs `sudo`, updates `sudoers`, and locks the root password.

The second playbook copies my authorized SSH keys and sets up a secure `sshd` config.
I got some ideas from Michael W. Lucas's [managing sshd with Ansible](https://mwl.io/archives/1819) blog post.
However, note that the ansible syntax he uses is now out of date.
Instead of `with_items`, you need `loop`.

### `bhyve` Setup with Ansible

After implementing my basic setup in Ansible, I started working on the setup to run virtual machines with `bhyve`.
For this part, I followed the [From 0 to Bhyve on FreeBSD 13.1](https://klarasystems.com/articles/from-0-to-bhyve-on-freebsd-13-1/) guide that Jim Salter wrote for Klara Systems.

I implemented the steps in the guide as a new Ansible playbook.

```yaml
- name: Set up bhyve
  hosts: freebsd
  remote_user: ansible
  become: yes

  tasks:
    # Tasks go here!
```

For each step in the guide, I added a task. First, install the required packages:

```yaml
  tasks:
    - name: install bhyve packages
      package:
        name:
          - vm-bhyve
          - bhyve-firmware
```

Create zfs datasets to hold VMs and templates:

```yaml
    - name: create bhyve dataset
      community.general.zfs:
        name: zroot/bhyve
        state: present
        extra_zfs_properties:
          recordsize: 64K

    - name: create bhyve templates dataset
      community.general.zfs:
        name: zroot/bhyve/.templates
        state: present
```

And enable services:

```yaml
    - name: enable bhyve service in rc.conf
      community.general.sysrc:
        name: vm_enable
        state: present
        value: "YES"

    - name: set bhyve vm path in rc.conf
      community.general.sysrc:
        name: vm_dir
        state: present
        value: "zfs:zroot/bhyve"

    - name: enable virtualization support in loader.conf
      community.general.sysrc:
        name: vmm_load
        state: present
        value: "YES"
        path: /boot/loader.conf
```

The next step was where I ran into trouble.
To enable network access for the VMs, you can use the `vm-bhyve` utility to set up a virtual switch and attach a network interface.
These commands are run just once.

```console
# vm init 
# vm switch create public 
# vm switch add public <your network interface>
```

That's easy enough to do in Ansible using the `command` module, but this approach creates a problem:

```yaml
    - name: initialize vm-bhyve
      ansible.builtin.command: /usr/local/sbin/vm init

    - name: create virtual switch
      ansible.builtin.command: /usr/local/sbin/vm switch create public

    - name: attach interface to virtual switch
      ansible.builtin.command: 
        cmd: "/usr/local/sbin/vm switch add public {{ ansible_facts['default_ipv4']['interface'] }}"
```

The commands work just fine the first time.
However, each time I change something and re-run the playbook, Ansible does these run-once commands again.

```console
TASK [initialize vm-bhyve] ****************************************************
changed: [vmhost]

TASK [create virtual switch] **************************************************
fatal: [vmhost]: FAILED! => {"changed": true, "cmd": ["/usr/local/sbin/vm", "switch", "create", "public"], "delta": "0:00:00.013700", "end": "2022-07-24 15:57:38.339834", "msg": "non-zero return code", "rc": 1, "start": "2022-07-24 15:57:38.326134", "stderr": "/usr/local/sbin/vm: ERROR: switch public already exists", "stderr_lines": ["/usr/local/sbin/vm: ERROR: switch public already exists"], "stdout": "", "stdout_lines": []}

PLAY RECAP ********************************************************************
vmhost          : ok=7    changed=1    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0
```

This time, the second command returns an error because the switch is already there.
Ansible then halts halfway through with a failure, even though the error is not a problem.

All of the other tasks are *declarative*.
You declare the end state you want, and Ansible either verifies that the machine is already there or does whatever is needed to get it there.
However, these commands are *imperative*.
You tell Ansible what steps to take, and it does them exactly.
Either approach make sense, but mixing them is really awkward.

I could just log in and run the commands manually, but I didn't want to give up.
Instead, I came up with two potential solutions.

#### The Hard Way

The first thing I did was to look for an Ansible module to interact with the `vm-bhyve` utility in a declarative way.
I didn't find one, so, naturally, I decided to write one.
I've never written an Ansible module before, and I'm not much of a Python developer, but after about six hours I had something that pretty much worked.

My `ansible-vmbhyve` module supports creating and destroying virtual switches and adding and removing interfaces.
If you want to try it or contribute to it, you can find it on GitHub here: [neapsix/ansible-vmbhyve](https://github.com/neapsix/ansible-vmbhyve).

With this module, I could set up the virtual switch in declarative language without getting errors on subsequent runs:

```yaml
    - name: create virtual switch and attach interface
      vm-bhyve: /usr/local/sbin/vm init
        switch: public
        state: present
        interfaces: "{{ ansible_facts['default_ipv4']['interface'] }}"
```

Yay, it works!
However, because this module has to be installed manually, it's no easier than running the commands manually or commenting them out after the first run.
Fortunately, there's another way to fix the issue, at least until my half-baked module is ready for primetime.

#### The Easy Way

Ansible supports some [options for error handling](https://docs.ansible.com/ansible/latest/user_guide/playbooks_error_handling.html) when a task fails.
You can choose to ignore errors from a certain task and keep going even if it fails.
To do so, add `ignore_errors: yes` to that task:

```yaml
    - name: create virtual switch
      ansible.builtin.command: /usr/local/sbin/vm switch create public
      ignore_errors: yes

    - name: attach interface to virtual switch
      ansible.builtin.command: 
        cmd: "/usr/local/sbin/vm switch add public {{ ansible_facts['default_ipv4']['interface'] }}"
      ignore_errors: yes
```

I didn't love that option, though.
I do want the playbook to fail if there's an error *the first time*.
To make that happen, you can change what Ansible considers a failure.
I set the `failed_when` option to consider the task a failure if the command returns an error, just not this particular error.

```yaml
    - name: create virtual switch
      ansible.builtin.command: /usr/local/sbin/vm switch create public
      register: result
      failed_when: >
        (result.stderr != '') and
        ("ERROR: switch public already exists" not in result.stderr)

    - name: attach interface to virtual switch
      ansible.builtin.command: 
        cmd: "/usr/local/sbin/vm switch add public {{ ansible_facts['default_ipv4']['interface'] }}"
      register: result
      failed_when: >
        (result.stderr != '') and
        ("ERROR: failed to add member igb0 to the virtual switch public" not in result.stderr)
```

Ansible now passes these tasks with a result of `changed: [vmhost]`.

One caveat: the `vm switch add` command gives only a generic "failed to add" message instead of a specific "already exists" message.
I assume it returns the same error when it fails for any reason, so there's no benefit to defining `failed_when` over `ignore_errors` for this command.

### Conclusion and Plug

When I finished this setup, I created a Ubuntu 22.04 VM to test it out, and I was shocked at how easy it was.
Right out of the box, my VM had networking, internet access, and its own IP address.
It could even run Docker containers.
I expected to have to mess around with NAT and bridging, but `vm-bhyve` did it all for me.

If you use VMs and you haven't used `bhyve`, I highly recommend you try it.
It's gained a ton of features since I first looked at it a few years ago, and `vm-bhyve` makes setting up VMs quick and comfy.

I'm so excited and grateful to have these tools on FreeBSD, and I can't wait to do more with them!
