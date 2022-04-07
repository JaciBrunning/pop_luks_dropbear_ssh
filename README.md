# Decrypting LUKS on Pop_OS! over SSH
If you're like me and encrypt your boot drive with LUKS, but also boot your computer remotely using Wake-on-LAN, you've run into the issue of not being able to access your machine until the boot disk is unlocked. With these instructions, you can open a remote SSH connection to your machine to unlock the boot disk and allow the system to boot.

We do this by running the dropbear SSH server in the `initramfs`, which we can use to open the boot disk with `cryptroot-unlock`.

This guide assumes you're running Pop_OS! (and hence use systemd-boot and netplan), but there are similar guides for Ubuntu and other distros, including the primary guide I used for this: https://hamy.io/post/0009/how-to-install-luks-encrypted-ubuntu-18.04.x-server-and-enable-remote-unlocking/

## Installing Dropbear
First, make sure you're up to date:
```
apt update
apt upgrade
```

Next, install dropbear in the initramfs.
```
apt install dropbear-initramfs
```

## Configuring Dropbear
Now you can configure dropbear by editing `/etc/dropbear-initramfs/config` as root:
`/etc/dropbear-initramfs/config`
```
# ...
DROPBEAR_OPTIONS="-s -p 2222 -j -k -I 120"
# ...
```
These are the options that I use and recommend. You can find them for yourself by running `man dropbear`
- `-s`: Disable password logins for root, since we'll be using key-based authentication
- `-p 2222`: Run the server on port 2222. I don't run it on 22 since otherwise clients can get confused since dropbear will have a different host private key than the SSH server running on your system.
- `-j -k`: Disable local and remote port forwarding respectively
- `-I 120`: Timeout clients after 120s of inactivity, in case you leave your machine unlocked.

## Adding Public Keys
Now you can specify what machines can SSH in using key-based auth. These are the public keys of your clients, stored in `~/.ssh/id_rsa.pub`. Place one-per-line in `/etc/dropbear-initramfs/authorized_keys`.

I add some extra options, which you can find below.
`/etc/dropbear-initramfs/authorized_keys`
```
no-port-forwarding,no-agent-forwarding,no-x11-forwarding,command="/bin/cryptroot-unlock" ssh-rsa ...
```

`no-XXXXX-forwarding` are all self-explanatory, but `command="/bin/cryptroot-unlock` is specified to drop the user straight to the password prompt for the boot disk once connected over SSH. If you want to give them access to the whole initramfs, you can neglect this.

## Configuring IP Address during boot
To access the machine over SSH, it needs to have an IP address before entering userspace. We can set this as a kernel boot option using `kernelstub`

```
$ kernelstub -a "ip=<ip addr>::<gateway ip>:<netmask>:<hostname>:<interface>"
# Or, if you're using VLANs
$ kernelstub -a "vlan=<interface>.<tag>:<interface>     # e.g. eth0.10=eth0 for VLAN ID 10 on eth0
$ kernelstub -a "ip=<ip addr>::<gateway ip>:<netmask>:<hostname>:<interface>.<tag>"
```

It's important to specify the interface (e.g. `eth0`) since otherwise it will apply to every interface and maintain during boot

Now, update your initramfs
```
$ update-initramfs -u
```

## Dropping IP config prior to boot
Usually, these IP settings will be maintained after boot through generated config files in `/run/netplan/`. This is fine, but if you'd prefer your network configuration to be set by userspace (which you probably do), you can drop these kernel IPs after the system has unlocked the root partition by adding the following script:

`/etc/initramfs-tools/scripts/init-bottom/netplan-revert.sh`
```
#!/bin/sh
rm -f /run/netplan/<interface>.yaml  # e.g. rm -f /run/netplan/eth0.yaml, or rm -f /run/netplan/eth0.10.yaml for a VLAN

# If you ARE NOT using vlans:
ip -f inet address flush dev <interface>
# If you ARE using vlans:
ip link delete <interface>.<tag>
```

Make the script executable and ownable by root, then update your initramfs
```
$ chown root:root /etc/initramfs-tools/scripts/init-bottom/netplan-revert.sh
$ chmod +x /etc/initramfs-tools/scripts/init-bottom/netplan-revert.sh
$ update-initramfs -u
```

### And you're done!
You can SSH into your system with `ssh -p 2222 root@<ip>`, which should drop you straight to a password prompt. Put in your LUKS password, and the SSH session will terminate while your system unlocks. SSH to your regular user (`ssh jaci@<ip>`) and you should be all set!
