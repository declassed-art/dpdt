# Declassed Plausible Deniabity Toolkit

This toolkit provides a little code to ease deploying and bootstrapping a hidden system
without obvious encryption, as it happens with LUKS, or Thomb, or anything else that leaves traces.
The idea is to use unallocated space in a file system or unpartitioned space.
A similar tool could be recently emerged Shufflecake, but that's a third party kernel
module that increases pros of rubber-hose cryptanalysis.

This toolkit does not use anything non-standard. For example, Linux Mint provides everything
out of the box. For Debian you may need to install cryptsetup, though.
The drawback of the approach being described is the need to memorize a sequence of bootstrapping commands
and long enough passphrase. However, you can use a tiny device, safely hidden,
that would bootstrap your system or even the entire infrastructure. Up to you.

Although this toolkit is quite mature, being used in production for almost six years,
I haven't heard it was a subject of rubber-hose cryptanalysis.
So it's strength is still under the question.

This toolkit is for Debian-based distros only.
If you're using a distro with a decent init system, you might need to modify
this toolkit to get rid of `systemd` invocations.

## The basic idea

In Linux you can create a device anywhere with `losetup` command. Even on a mounted partition.
`dm` does not allow that, complaining the device is already in use.

The stable approach is using single loop device per encrypted volume. I mean, you could use `dm`
to join multiple unused areas on your file system into a single volume but that does not work
as expected. I have tried that and I saw a lot of errors in `dmesg` output related to asynchronous I/O.
Maybe things have changed since then, I have no desire to dig into this problem again, I just warn.

Also, the whole `losetup` approach is not reliable on old Allwinner H3 based boards
and I suspect the same applies to old RPI boards. They are probably too slow,
I saw the same async. I/O errors in `dmesg` output.

So, given that you have some file system, you can create an encrypted volume in unallocated space.
Thus, you can plausibly deny existence of any encrypted data.

You'll have to avoid any writes to that filesystem, otherwise your secret volume can be damaged.
Consider use unpartitioned space as well.

You'll be right if you note that the defense is so-so.
You'll have to explain any deviation from Quick Install, and you'll
have to explain why you turned TRIM off (otherwise you'd lose your secret partition)
and why all unused sectors on your device have unusually high entropy.
So, as a side note, this approach is better suited for spinning disks than for SSDs.

Anyway, be creative, and good luck. At least, you can refer to this repository and say
"I gave it a try, it's total shit and am no longer using it."

You also might say "This toolkit is a specialized thing! This would expose me!"
Actually, no. You should place this only on your encrypted volume.
Don't download it to your un-encrypted file system.
Remember I mentioned a sequence of bootstrapping commands above?

## Prerequisites

Get rid of everything that may leave traces:

* Turn off swap.
* Use tmpfs for logs. If you're using Armbian, turn armbian-ramlog off because it writes logs back to memory card, use plain tmpfs for /var/log.
* Turn history off for root's shell.

And fill your storage device with random data before using this approach:

```
dd if=/dev/urandom of=/dev/sda bs=4K
```

Replace `/dev/sda` with your actual device in the above command.

You will need a tiny bootstrap volume where you will keep this toolkit and the configuration for all the rest volumes.

Given that MBR takes only one sector, GPT takes 34 sectors and typically the first partition starts from sector 2048,
you have "plenty" space in that area for your hidden bootstrap volume.

If you're using an ARM board with uboot, you can create your hidden volume close to the beginning of the first partition
because uboot does not use all reserved sectors.
Basically, you can create your volume in any unpartitioned gap, bear in mind, however, that GPT also uses last 34 sectors
of your storage device.

Where you place your bootstrap volume also may highly depend on a number you can easily memorize: the start sector.

Once you have chosen the location (say, 20480) and a long enough passphrase for `cryptsetup`,
let's create and mount the bootstrap volume. 256K is more than enough for simple case, and it's easy to memorize too:

```
losetup --offset 20480 --sizelimit 256000 -f /dev/sda
cryptsetup open /dev/loop0 bootstrap --type plain
mkfs -t ext2 /dev/mapper/bootstrap
mkdir /mnt/stuff
mount /dev/mapper/bootstrap /mnt/stuff
```

If you need a lot of configuration to bootstrap your entire infrastructure from single point, you may need
to double the size of your bootstrap volume.

Now you can download this toolkit and create the configuration.

Along with passphrase, you will need to memorize the following commands to run when you reboot your system:

```
losetup --offset 20480 --sizelimit 256000 -f /dev/sda
cryptsetup open /dev/loop0 bootstrap --type plain
mount /dev/mapper/bootstrap /mnt/stuff
cd /
/mnt/stuff/my-computer/bootstrap
```

Where `/mnt/stuff/my-computer/bootstrap` is a script that does all the rest.

Basically, you may not need such a bootstrap volume. With this toolkit you can bootstrap remotely
via `ssh`. You can use some tiny device which you can safely hide.
That could bootstrap your entire infrastructure.
Another device could be a part of alarm system, sending broadcast packets to shut everything down
-- see `TeardownOnSignal` task in dpdt_tasks.py, for example.
Up to you. Be creative.

The final question, how to find unallocated space on the file system. I use `secha.py` script.
It's ugly but it works for me. I run it to collect sector hashes immediately after writing
random data to the device and then once again after installing the base system to find intact sectors.

## Configuration

The configuration is stored in subdirectories, one for each host. In the basic case this could be
`my-computer/config.json`.

Here's an example. Everything should be clear:

```json
{
    "devices": {
        "S23SNEAG516433Y": "ssd",
        "WD-WX61BA51RF6A": "hdd"
    },
    "volumes": {
        "my-data": {
            "device": "ssd",
            "start":  "128664014 * 512",
            "end":   "(((488397168 - 34) * 512) // 4096) * 4096",
            "sector_size":  4096,
            "key": "KAQbEe9XZ4kPbYEWzZ3XZlbydGnkV0yLCoPSZVIP0cgyxTYC",
            "mount_point": "/mnt/my-data"
        },
        "my-archive": {
            "device": "hdd",
            "start":  "838186828 * 512",
            "end":   "((1465149168 * 512) // 4096) * 4096",
            "sector_size":  4096,
            "key": "DthVIyikwY5tTUvmQwFpzfm6Fze2TvaV9iLbWp2W5eps64TF",
            "mount_point": "/mnt/my-archive"
        },
        "microsd": {
            "filename": "/dev/mmcblk0",
            "start":  1024000,
            "end":   "(3840000 * 512 // 4096) * 4096",
            "sector_size":  4096,
            "key": "amOnyfLGiRcq0PLZz5WprwZihEECZlRzJ4SmRUayZo24t0HM",
            "mount_point": "/mnt/microsd",
            "mount_options": ["commit=600"]
        }
    },
    "containers": [
        "devapps",
        "safedns"
    ]
}
```

The central part of configuration is volumes. All parameters are passed to losetup and cryptsetup.
Strings for `start` and `end` are evaluated to integers. `device` is resolved to a file name using `devices` mapping.

Keys can be generated by `dpdt_genkey` script.

All the rest in the configuration, such as `containers`, is for specific tasks.

You need to carefully think through your partitioning scheme before preparing your own configuration file.
Where you'll place your hidden volume? On a file system or on unpartitioned space?
If on a file system, there should be enough space on it. For example, ext4 scatters its data structures
across the partition, thus leaving relatively small free blocks, whereas FAT uses free space sequentially.
You might need to write some data to your filesystem before creating hidden partitions on it
so you could demonstrate to rubber-hose cryptanalysts that your partition is not dummy.

## Toolkit

The basic thing this toolkit provides is `Invoke` class that allows to run commands
either locally or remotely. It's similar to Invoke from Fabric project, but I did not look
at that, honestly. As I remember they used paramiko. My implementation is 100% mine,
just to avoid any dependencies and I simply use subprocess module and `ssh` command.

Another thing is `Task` class that declares setup and teardown methods.
A sequence of tasks can bring the system to the original state if something goes wrong, thanks to teardown
methods. Although for production deployment I'd strongly recommend reboot.

Basically, I did not bother with the design of the toolkit, it may look a bit messy just for historical
reasons.

## Bootstrap script

This toolkit does not provide any bootstrap script. This script can be specific for a particular system
and is placed in the same directory along with configuration file.
A couple of use cases is considered below.

In a nutshell, the script replaces `/etc` and other essential directories with updated versions
located on your hidden volume and then restarts some services and sets other things up as necessary.

### Bootstrap script for desktop

Your desktop system may look innocent toy, where you, a sole `user`, only watch cats on youtube and play tux racer.
This bootstrap script turns it into a combat system with another three users: work, browsing, and sans-vpn.
The former twos use VPN, and the latter one does not.
You can log in under different users simultaneously and switch between them using Ctrl-Alt-F7...Ctrl-Alt-F10 keys.
There are some issues with the sound, but basically this works fine.

```python
#!/usr/bin/env python3

import os
base_dir = os.path.dirname(os.path.abspath(__file__))

import sys
sys.path.insert(0, os.path.dirname(base_dir))

from dpdt_base import read_config, setup, Invoke
from dpdt_tasks import *

# can bootstrap a remote system via ssh
if len(sys.argv) > 1:
    remote = sys.argv[1]
else:
    remote = None

# read configuration and instantiate Invoke class
config = read_config(base_dir)
invoke = Invoke(remote=remote)
invoke.set_devices(config)  # process device map: resolve devices to their actual file names

setup(
    config, invoke,

    TmpfsMounts('/mnt'),  # re-mount /mnt to tmpfs
    MountRoot,            # get access to the root device via /mnt/root
    MountVolumes,         # mount all configured volumes
    TmpfsMounts(          # re-mount more directories to tmpfs:
        '/home',          #   we have more users, actually
        '/var/lib/lxc'    #   and more lxc containers
    ),
    BindMounts(           # replace /root directory
        ('/mnt/my-data/root', '/root')
    )
)

if remote:
    # allowed SSH key has changed after replacing /root so we need to re-instantiate Invoke
    invoke = Invoke(remote=remote, ssh_key='~/.ssh/secret_id_ecdsa')

# okay, come on
setup(
    config, invoke,

    # populate /home with users' home directories
    BindMounts(
        ('/mnt/my-data/work',     '/home/work'),
        ('/mnt/my-data/browsing', '/home/browsing'),
        ('/mnt/my-data/sans-vpn', '/home/sans-vpn')
    ),
    # re-mount essential directories using overlayfs
    OverlayMounts(
        ('/mnt/root/etc',       '/mnt/my-data/etc',       '/mnt/my-data/etc.workdir',       '/etc'),
        ('/mnt/root/var/tmp',   '/mnt/my-data/var/tmp',   '/mnt/my-data/var/tmp.workdir',   '/var/tmp'),
        ('/mnt/root/usr/local', '/mnt/my-data/usr/local', '/mnt/my-data/usr/local.workdir', '/usr/local')
    ),
    # unclobber some directories
    BindMounts(
        ('/mnt/root/etc/apt',       '/etc/apt'),
        ('/mnt/root/home/user',     '/home/user')  # assume `user` was the only user in the base system
    ),
    RestartServices(
        'syslog',
        'systemd-journald',
        'autofs',
        'display-manager'
    )
)

# create necessary directories in /mnt, which is a tmpfs now
invoke.run('mkdir -p /mnt/myserver')
invoke.run('mkdir -p /mnt/usb')
invoke.run('mkdir -p /mnt/temp')

# assume we don't have much changes in replaced /etc, just firewall rules and...
invoke.run('systemctl restart nftables')

# ...and we have wireguard config now.
# Note we don't use wg-quick for VPN, it breaks ip rules and makes per-user routing impossible
invoke.run('ip link add dev wg0 type wireguard')
invoke.run('ip address add 10.0.0.2 dev wg0 peer 10.0.0.1')
invoke.run('/bin/bash -c "wg setconf wg0 <(wg-quick strip /etc/wireguard/wg0.conf)"', shell=True)
invoke.run('ip link set up dev wg0')
invoke.run('ip route replace default dev wg0')

# add direct route to the VPN server itself
invoke.run('ip route add 1.2.3.4 via 192.168.0.1')

# sans-vpn, assume uid is 1003:
invoke.run('ip rule add uidrange 1003-1003 table 1 pref 100')
invoke.run('ip route add default via 192.168.0.1 table 1')

# assuming the bootstrap volume was mounted on /tmp/boot, unmount it:
if not remote:
    invoke.locrypt_unmount('/tmp/boot')

# start emergency switch
invoke.run('python3 emergency-switch.py &', shell=True, capture_output=False)
```

### Bootstrap script for server

Assuming this script is run from a central point to bootstrap the entire infrastructure via `ssh`,
the name of the server, `myserver`, is hardcoded.

The basic system simply runs LXC containers from hidden volume.

```python
#!/usr/bin/env python3

import os
base_dir = os.path.dirname(os.path.abspath(__file__))

import sys
sys.path.insert(0, os.path.dirname(base_dir))

from dpdt_base import read_config, setup, Invoke
from dpdt_tasks import *

config = read_config(base_dir)
invoke = Invoke(remote='myserver')
invoke.set_devices(config)

invoke.run('systemctl stop nfs-kernel-server')

setup(
    config, invoke,

    TmpfsMounts('/mnt'),  # re-mount /mnt to tmpfs
    MountRoot,            # get access to the root device via /mnt/root
    MountVolumes,         # mount all configured volumes
    BindMounts(           # replace /root directory
        ('/mnt/hidden/root', '/root')
    )
)

# allowed SSH key has changed after replacing /root so we need to re-instantiate Invoke
invoke = Invoke(remote='myserver', ssh_key='~/.ssh/secret_id_ecdsa')

setup(
    config, invoke,

    # re-mount essential directories using overlayfs
    OverlayMounts(
        ('/mnt/root/etc',       '/mnt/hidden/etc',       '/mnt/hidden/etc.workdir',       '/etc'),
        ('/mnt/root/var/tmp',   '/mnt/hidden/var/tmp',   '/mnt/hidden/var/tmp.workdir',   '/var/tmp'),
        ('/mnt/root/usr/local', '/mnt/hidden/usr/local', '/mnt/hidden/usr/local.workdir', '/usr/local')
    ),
    BindMounts(
        # more lxc containers
        ('/mnt/hidden/lxc',   '/var/lib/lxc'),
        # unclobber some directories
        ('/mnt/root/etc/apt', '/etc/apt')
    )
)

# too many changes in /etc...
invoke.run('systemctl daemon-reexec')
invoke.run('systemctl start nfs-kernel-server')

# start LXC containers
for container_name in config['containers']:
    invoke.run(f'lxc-start {container_name}')
```

## Anything else?

Let me know if anything is unclear.

If you found this useful, I won't refuse your satoshis:
`bitcoin:bc1qwksstkxqu8a7e484k70fpa5euxv55f87fps53m`.
Basically, I don't need your money but I do need to thank my angel.

## See also

* [Previous version](https://declassed.art/en/blog/2022/06/23/a-way-to-hide-your-secrets-and-denial-plausibility)
* [Main article](https://declassed.art/en/blog/2022/12/03/declassed-plausible-deniability-toolkit)
