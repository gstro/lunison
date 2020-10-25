# lunison
Secure, nightly file sync.

- USB Hard Drive Encryption
- VPN Setup
- Unison Sync
- Systemctl Scheduling

These are instructions for how I set up a Raspberry Pi with an external hard
drive to securely sync files with another system. I also run a separate script
that makes nightly web requests for files over a VPN, so I included my settings
for the VPN and for setting up the schedule.

### USB Hard Drive Encryption

Mostly inspired by [USB Hard Drive Encryption on a Raspberry Pi](http://longsteve.com/wiki/index.php/USB_Hard_Drive_Encryption_on_a_Raspberry_Pi).

1. Use `lsblk` to identify the HDD partition. NOTE: this output is _after_ the
drive had been configured as an encrypted device.
```
$ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
sda           8:0    0 931.5G  0 disk
└─sda1        8:1    0 931.5G  0 part
  └─vault   254:0    0 931.5G  0 crypt /media/vault
mmcblk0     179:0    0  29.8G  0 disk
├─mmcblk0p1 179:1    0   2.2G  0 part
├─mmcblk0p2 179:2    0     1K  0 part
├─mmcblk0p5 179:5    0    32M  0 part
├─mmcblk0p6 179:6    0   256M  0 part  /boot
└─mmcblk0p7 179:7    0  27.3G  0 part  /
```

2. Format the `/dev/<PART>` partition with `mkfs.ext4`, e.g.
```
sudo mkfs.ext4 /dev/sda1
```

3. Install `cryptsetup` and start the dm-crypt modules. This lets us avoid
rebooting later.
```
sudo apt-get install cryptsetup
sudo modprobe dm-crypt sha256 aes
```

4. Use `cryptsetup` to setup the encrypted partition. The flags:
- `c`: the cipher (`aes`); (`cat /proc/crypto` for available ciphers)
- `s`: the keysize in bits (256 bits)
- `h`: the hash used to create the encryption key from the passphrase
```
sudo cryptsetup --verify-passphrase luksFormat /dev/sda1 -c aes -s 256 -h sha256
```

5. Open the encrypted partition, giving it a `<NAME>` e.g. 'vault'
```
sudo cryptsetup luksOpen /dev/sda1 vault
```

6. Format the encrypted partition.
```
sudo mkfs -t ext4 -m 1 /dev/mapper/vault
```

7. Create a mount point, mount the encrypted partition, and make it available to
the non-root user.
```
sudo mkdir /media/vault
sudo mount /dev/mapper/vault /media/vault/
sudo chown pi:pi /media/vault/
```

8. Create a keyfile from `urandom`.
- `if`: input file (`urandom`)
- `of`: output file (`/root/key/file`)
- `bs`: read and write number of bytes (`1024` bytes)
- `count`: copy only this many input blocks (`4`)
This creates a 4KB keyfile:
```
sudo dd if=/dev/urandom of=/root/key/file bs=1024 count=4
```

9. Make the keyfile read-only to root
```
sudo chmod 0400 /root/key/file
```

10. Add the keyfile to the encrypted partition
```
sudo cryptsetup luksAddKey /dev/sda1 /root/key/file
```

11. Add an entry to the `/etc/crypttab` file with the following format
```
<target name>    <source device>    <key file>    <options>
```
For example:
```
vault    /dev/sda1    /root/key/file    luks
```

12. Mount the device in fstab -- add an entry to `/etc/fstab`:
```
/dev/mapper/vault   /media/vault ext4   defaults,rw       0       0
```

Once these steps are completed, the USB hard drive should auto-mount on system
startup. It will be an encrypted drive, so if it was connected to another system
(without the keyfile or password) the contents would not be accessible.

##### Sources

- [USB Hard Drive Encryption on a Raspberry Pi](http://longsteve.com/wiki/index.php/USB_Hard_Drive_Encryption_on_a_Raspberry_Pi)
- [How to encrypt a LUKS container using a smart card or token](https://blog.g3rt.nl/luks-smartcard-or-token.html)
- [HOWTO: Automatically Unlock LUKS Encrypted Drives With A Keyfile](https://www.howtoforge.com/automatically-unlock-luks-encrypted-drives-with-a-keyfile)
- [Keyfile-based LUKS encryption in Debian](https://www.finnie.org/2009/07/26/keyfile-based-luks-encryption-in-debian/)

### VPN Setup

Mostly inspired by [NordVPN on the Raspberry Pi](https://pimylifeup.com/raspberry-pi-nordvpn/).
This assumes a NordVPN account already. Additional instructions found at the
[NordVPN site](https://support.nordvpn.com/Connectivity/Linux/1047409422/How-can-I-connect-to-NordVPN-using-Linux-Terminal.htm)

1. Login to the Raspberry Pi via SSH, and update to the latest packages
```
sudo apt-get update
sudo apt-get upgrade
```

2. Install the OpenVPN package, if it's not already installed, and navigate to
its directory
```
sudo apt-get install openvpn
cd /etc/openvpn
```

3. Download the OpenVPN config files here:
```
sudo wget https://downloads.nordcdn.com/configs/archives/servers/ovpn.zip
```

4. Extract the files.
```
sudo unzip ovpn.zip
```

5. Go to [NordVPN Tools](https://nordvpn.com/servers/tools/) to determine the
best server for your requirements.

6. Start OpenVPN with the config for the selected server.
```
sudo openvpn [file name]
```

7. OpenVPN will ask you for your NordVPN credentials which can be found in the
[Nord Account dashboard](https://my.nordaccount.com/dashboard/nordvpn/).


The VPN will now be active. You can test with
```
curl -ks https://ipv4.ipleak.net/json/
```

Now, we want to setup the VPN to connect on boot.

8. Store the username and password in an `auth.txt` file
```
sudo vim auth.txt
```

9. Store the username and password on separate lines,
```
username
password
```

10. Copy the specified server config and simplify the name, e.g.
```
sudo cp au151.nordvpn.com.tcp443.ovpn au151.conf
```

11. Edit the file to instruct it to use the `auth.txt` file.
```
sudo vim au151.conf
```
Replace
```
auth-user-pass
```
with
```
auth-user-pass auth.txt
```

12. Reboot the Pi and reconnect via SSH
```
sudo reboot
```

The VPN should still be active, check again with
```
curl -ks https://ipv4.ipleak.net/json/
```

Now we want to ensure that DNS does not leak.

13. Edit `dhcpcd.conf` to set the static DNS to [Cloudflare's `1.1.1.1`](https://www.cloudflare.com/learning/dns/what-is-1.1.1.1/).
```
sudo vim /etc/dhcpcd.conf
```
Replace
```
#static domain_name_servers=192.168.0.1
```
with
```
static domain_name_servers=1.1.1.1
```
and reboot
```
sudo reboot
```

We can confirm that the DNS leak is closed,

```
time curl -ks https://www.dnsleaktest.com/ | grep flag
```

Now the Raspberry Pi has an encrypted external drive, and an encrypted
connection to the web. Next we want to setup a simple file sync.

### Unison Sync

We'll use [Unison](https://www.cis.upenn.edu/~bcpierce/unison/) as a syncing
utility. From [ArchWiki](https://wiki.archlinux.org/index.php/unison):
> Unison is a bidirectional file synchronization tool that runs on Unix-like operating systems (including Linux, macOS, and Solaris) and Windows. It allows two replicas of a collection of files and directories to be stored on different hosts (or different disks on the same host), modified separately, and then brought up to date by propagating the changes in each replica to the other.

The trickiest part of this setup is that both systems (for us, the Raspberry Pi
and a macOS) have to use almost identical `unison` versions. This means that
either the `apt` and `brew` versions must be aligned, or, more likely, that
we'll need to manually install one side to match the other. Some hints can be
found [here](https://dvj.me.uk/blog/install-unison-mac-raspbian).

I preferred to use `brew` on the macOS system, and then manually match that
`unison` version with the Raspberry Pi.

1. Install `unison` on macOS via `brew`
```
brew update && brew upgrade && brew cleanup && brew install unison
```

2. Check the `unison` version to determine which to match
```
unison -version
unison version 2.51.3 (ocaml 4.10.0)
```
This will tell you which `unison` and `ocaml` versions you'll need on your
other system.

3. On the Raspberry Pi, install the OCaml package manager `opam`
```
sudo apt-get install opam m4
```

4. Set the OCaml version to match the macOS unison
```
opam init -y --compiler=4.10.0
eval `opam config env`
```

5. Download the source for the unison version to match macOS and build
```
wget http://www.cis.upenn.edu/\~bcpierce/unison/download/releases/unison-2.51.3/unison-2.51.3.tar.gz
tar -xvzf unison-2.51.3.tar.gz
cd src
make UISTYLE=text
```

6. Copy the utility to the path `/usr/local/bin`
```
sudo cp ./unison /usr/local/bin
```

Now we need to setup the necessary keys.

7. On the macOS, generate an ssh key, _without a passphrase_.
```
ssh-keygen -t ed25519
```

8. Copy the pubkey to the Raspberry Pi
```
ssh-copy-id -i ~/.ssh/id_ed25519.pub pi@raspberrypi
```

9. Login to the Raspberry Pi and check for the correct key
```
cat .ssh/authorized_keys
ssh-ed25519 <pubkey> <macOS>
```

10. On both systems, setup a test directory at `~/UnisonTest/`. Test the
connection with a quick `-testserver` flag. On macOS,
```
unison ~/UnisonTest ssh://pi@raspberrypi//home/pi/UnisonTest -testserver
Unison 2.51.3 (ocaml 4.10.0): Contacting server...
Connected [//MBP//Users/username/UnisonTest -> //raspberrypi//home/pi/UnisonTest]
```

11. On macOS, create a test file and test the sync
```
echo "testing unison" > ~/UnisonTest/test.txt
```
```
unison ~/UnisonTest ssh://pi@raspberrypi//home/pi/UnisonTest
Unison 2.51.3 (ocaml 4.10.0): Contacting server...
Connected [//MBP//Users/username/UnisonTest -> //raspberrypi//home/pi/UnisonTest]
Looking for changes
Warning: No archive files were found for these roots, whose canonical names are:
	/Users/username/UnisonTest
	//raspberrypi//home/pi/UnisonTest
This can happen either
because this is the first time you have synchronized these roots,
or because you have upgraded Unison to a new version with a different
archive format.
...
```

12. Check that the file was synced. On the Raspberry Pi,
```
pi@raspberrypi:~ $ ls UnisonTest/
test.txt
```

13. See the `default.prf` file in this repo to see some custom configuration
options. This will let you just run `unison` without specifying any parameters.
See also [this Unison tutorial](https://www.howtoforge.com/tutorial/unison-file-sync-between-two-servers-on-debian-10/)
for additional config file details.


### Systemctl Scheduling

I wanted a script that I could run nightly, regardless of whether my laptop was
on and connected. So I setup the Raspberry Pi to run the script. It makes web
requests so I wanted it protected with the VPN. The script sometimes generates
output files so I wanted to sync those back to my macOS system when I wanted, so
I setup `unison`. Lastly, I wanted the script to run nightly, at least when the
Raspberry Pi was on and connected, so I setup `systemctl` to run my `nightly.py`
script. `Systemctl` is a more modern scheduler than `cron`, and fits my use-case
well.

I've included my files in the repo and below.

1. The first step is to create a `service`.
```
sudo vim /lib/systemd/system/nightly.service
```

2. Add the following content
```
[Unit]
Description=Run all nightly tasks
# Prevent the script from triggering before the network is available
After=network.target

[Service]
Type=simple
ExecStart=/home/pi/scripts/nightly.py
User=pi

[Install]
# Allow the script to run without needing a user to be logged in
WantedBy=multi-user.target
```

3. Next, create a `timer`.
```
sudo vim /lib/systemd/system/nightly.timer
```

4. Add the following content
```
[Unit]
Description=Schedule for nightly scraping
RefuseManualStart=no        # Allow manual starts
RefuseManualStop=no         # Allow manual stops

[Timer]
# Execute job if it missed a run due to machine being off
Persistent=true
# Run 5 min (300 seconds) after boot for the first time
OnBootSec=300
# Run every day at midnight
OnCalendar=daily
# File describing job to execute
Unit=nightly.service

[Install]
WantedBy=timers.target
```

To interface with `systemctl`:

#### Enable Service
`systemctl enable nightly.service`
`systemctl enable nightly.timer`

#### Schedule Service
`systemctl start nightly.timer`

### Check Status
`systemctl status nightly`
`systemctl list-unit-files`

### Manually Start and Stop Service
`systemctl start nightly.service`
`systemctl stop nightly.service`
