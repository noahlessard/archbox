# INSTALLING ARCH ON WEIRD OLD MACHINES

**based on https://gist.github.com/mjnaderi/28264ce68f87f52f2cabb823a503e673**


Start with the partioning of the disk. We will assume a non-uefi device.
Remember to use fdisk for non UEFI devices (look /sys/firmware/efi to check which type you have).  

Start with fdisk, and create MBR table.  

```
fdisk /dev/sda
(fdisk) o
```

Here are the paritions to create 
```
Device     Boot     Start       End   Sectors   Size Id Type
/dev/sda1            2048    526335    524288   256M 83 Linux      /boot
/dev/sda2          526336 765986815 765460480   365G 83 Linux      Encrypted with LUKS, 3 LVM partitions:
    swap  vg0 -wi-ao----   8.00g                                   swap
    root  vg0 -wi-ao----  80.00g                                   /
    anbar vg0 -wi-ao---- 277.00g                                   for user use...
/dev/sda3       765986816 976773167 210786352 100.5G 83 Linux      (Optional) Other partitions if you need... You can encrypt them separately with another password
```

Create paritions in fdisk by doing:
```
(fdisk) n
(fdisk) p
(fdisk) 1
(fdisk) <Enter>
(fdisk) +256M
(fdisk) t
(fdisk) 83

(fdisk) n
(fdisk) p
(fdisk) 2
(fdisk) <Enter>
(fdisk) +365G
(fdisk) t
(fdisk) 83


#you might not have to do this if you are not creating another user that isn't root (sda3)
(fdisk) n
(fdisk) p
(fdisk) 3
(fdisk) <Enter>
(fdisk) <Enter>
(fdisk) t
(fdisk) 83

(fdisk) w (Write Changes)
```

Then format the first parition in /dev/sda, the one that will become boot
```
mkfs.ext2 /dev/sda1
```

Next, encrypt the parition that will become root. Don't worry about password, that can be changed later. Just use something simple that you will not forget
```
cryptsetup -c aes-xts-plain64 -y --use-random luksFormat /dev/sda2
cryptsetup luksOpen /dev/sda2 luks
```

Create LVM Partitions: This creates one partions for root, modify if /home or other partitions should be on separate partitions
```
pvcreate /dev/mapper/luks
vgcreate vg0 /dev/mapper/luks
lvcreate --size 8G vg0 --name swap (ensure this G values match what you allocated earlier)
lvcreate --size 80G vg0 --name root
lvcreate -l +100%FREE vg0 --name anbar
```

Then format the LVM partitions:
```
mkfs.ext4 /dev/mapper/vg0-root
mkfs.ext4 /dev/mapper/vg0-anbar
mkswap /dev/mapper/vg0-swap
```

Then mount all the partitions:

```
mount /dev/mapper/vg0-root /mnt
mkdir /mnt/boot
mount /dev/sda1 /mnt/boot
swapon /dev/mapper/vg0-swap
```

Install the inital system with pacstrap (different than pacman):
```
pacstrap -i /mnt base base-devel linux linux-firmware openssh git vim
```


Generate Fstab, and edit it for tmp to be a ram folder if you want. 
```
# genfstab -pU /mnt >> /mnt/etc/fstab
tmpfs	/tmp	tmpfs	defaults,noatime,mode=1777	0	0
```


Enter the new system. 
```
arch-chroot /mnt /bin/bash
```
Now try to install mkinitpico and lvm2 (they might be under different names but use pacman -S)


Set timezone:
```
See available timezones:
ls /usr/share/zoneinfo/

Set timezone:
ln -s /usr/share/zoneinfo/Asia/Tehran /etc/localtime
```

Set locale
```
vim /etc/locale.gen (uncomment en_US.UTF-8 UTF-8)
locale-gen
echo LANG=en_US.UTF-8 > /etc/locale.conf
export LANG=en_US.UTF-8
```

Sync the clocks between the two OSs.
```
hwclock --systohc --utc
```

Set the hostname (change from the literal string hostname)
```
echo myhostname > /etc/hostname
```

Add this stuff to /etc/hosts (change only the myhostname)
```
127.0.0.1	localhost
::1		localhost
127.0.1.1	myhostname.localdomain	myhostname
```
Create a user
```
useradd -m -g users -G wheel -s myusername (try without the -s if this fails)
passwd myusername
visudo
# uncomment %wheel ALL=(ALL) ALL
```

Configure mkinitpico and add modules.
```
vim /etc/mkinitcpio.conf
#Add 'ext4' to MODULES (if failing to boot try adding "nvme" and "nvme_core")
#Add 'encrypt' and 'lvm2' to HOOKS before 'filesystems'
```

Regenerate the image, do this every time you change the .conf file:
```
mkinitcpio -p linux (you could also try mkinitcpio --allpresets, or delete the linux.preset in /etc/mkinitcpio.d and then running pacman -S linux)
```

Then install grub:
```
pacman -S grub
grub-install --target=i386-pc --recheck /dev/sda
```

Then configure grub by vim /etc/default/grub:
```
GRUB_CMDLINE_LINUX="root=/dev/mapper/vg0-root cryptdevice=/dev/sda2:luks:allow-discards"
```

and then generate grub config file by doing:
```
grub-mkconfig -o /boot/grub/grub.cfg
```

Now turn everything off and exit:
```
exit
umount -R /mnt
swapoff -a

REMOVE THE ARCH INSTALL USB KEY HERE!

reboot
```

This (should) now be a functioning arch linux system. It might not have interent or grahpics yet though. The rest of this guide will guide you through some extra installation steps that will help you secure and use your new arch system a little more easily. 

## Setup internet:

You can create an generic network file by doing:
```
vim /etc/systemd/network/20-wired.network (will create new file)
```

Then filling it with
```
[Match]
Name=en*
Name=eth*

[Network]
DHCP=yes
```

For a more specific file with setup static ip on ethernet:
```
vim /etc/systemd/network/eth0.network
```
Then fill with this stuff (example values)
```
[Match]
# You can also use wildcards. Maybe you want enable dhcp
# an all eth* NICs
Name=eth0
[Network]
#DHCP=v4
# static IP
# 192.168.100.2 netmask 255.255.255.0 (change 192.168.100.2 to something)
Address=192.168.100.2/24
Gateway=192.168.100.1
DNS=192.168.100.1
```

Next, Restart systemd-networkd and systemd-resolved. Restart systemd-networkd and systemd-resolved again if required:
```
# systemctl restart systemd-networkd systemd-resolved
# ping archlinux.org (shoud work)
```

## Gnome Stuff + Key signing:

Gnome is how arch has a graphical interface. You will need to install it manually.

If you are having issue with keys, do this as root (might be good pratice to just do it anyways):
```
rm -r /etc/pacman.d/gnupg/
pacman-key --init
pacman-key --populate archlinux
pacman -Sc
pacman -Syyu
```

First install xorg:
```
sudo pacman -S xorg xorg-server
```

Next install graphic drivers (this is generic one but might work):
```
# pacman -S xf86-video-intel
```

Then install gnome (names have changed a little)
```
# pacman -S gnome gdm (If these don't work just do "pacman -S gnome")
# pacman -S gnome-extra gnome-system-tools  (Optional)
```

Then enable the serivce for gnome (only if you want it to start every time):
```
sudo systemctl enable gdm.service
```
If you don't plan on using the graphics for interacting with the system, you can just disable the service just like you enabled it and that will help save some resources.

Finally reboot. 

## Locking with USB key:

This section is optionally, but a fun project. This section will encrypt the computer and have it automatically look for a usb key containing the keyfile. Once you go through all these steps, this is the only way it will open, so make some copies of the keyfile!

First, start with a freshly formatted USB key. Make sure it is using ext4 filesystem and give it a name you can rememeber but is not generic (the name is important). In this case we are assuming the name is usbstick. Next, put a random keyfile on it with:

```
dd bs=512 count=4 if=/dev/random of=/media/usbstick/mykeyfile iflag=fullblock
```

Then, in the main arch system, add a key to cryptsetup to use this key (make sure usb key is inserted).
```
cryptsetup luksAddKey /dev/sda2 /media/usbstick/mykeyfile
```

Now you have to edit the mkinitpico file. Make sure the ext4 module is installed, and if you are using the encrypt hook (you are if you are following this guide) then add this to the "grub linux default line", at the end after cryptdevice=/dev/sda2:luks:allow-discard.
```
cryptkey=LABEL=usbstick:ext4:/secretkey (The label should be the name of usb key, here it is usbstick. Then the filesystem, ext4, and then the path to the secret key).
```

Now make sure you saved everything and try rebooting with the usb key in. If it does not unlock, try putting in the cleartext password and then reconfigure the mkinitpico file and the grub files, then reboot. If this does unlock, the you can do the following command if you want to remove your cleartext password. 

```
# cryptsetup luksRemoveKey /dev/sdb1
Enter LUKS passphrase to be deleted: (put your password here)
```

If you want, you can make the system also shut down if it detects the key has been removed. This next section describes those steps, but it is not mandatory.

To automate the detection of a usb key, you need to make a bash script and a systemd service and time. The bash file needs to go in global /bin/. Below is an example of what that script could look like.

```
#!/bin/bash

if blkid | grep -q 'UUID="put your uuid here"';

then
	echo "USB stick inserted"
else
	echo "Key missing. Rebooting..."
	reboot
fi
```
Make this executable in bash by doing 
``` chmod u+x filename.sh ```

Next, make the service file for the systemd timer. Create a .service file in /etc/systemd/system (or /etc/systemd/user if they are user specific?).
``` vim /etc/systemd/system test_path.service ```

Fill the file with the following lines 
```
[Unit]
Description=Logs system statistics to the systemd journal
Wants=myMonitor.timer #same name as this file but with .timer

[Service]
Type=simple #could use oneshot
ExecStart=/bin/filename.sh #your sh file goes here, could try ExecStart=%h/bin/schedule-test.sh

[Install]
WantedBy=default.target #could also use multi-user.target 
```

Now, you have to create the timer file in /etc/systemd/system (or /etc/systemd/user if they are user specific?).
``` vim /etc/systemd/system test_path.timer ```

Fill it with the following
```
[Unit]
Description=Schedule a message every 1 minute
RefuseManualStart=no  # Allow manual starts
RefuseManualStop=no   # Allow manual stops

[Timer]
#Execute job if it missed a run due to machine being off
Persistent=true
#Run 120 seconds after boot for the first time
OnBootSec=120
#Run every 1 minute thereafter
OnUnitActiveSec=60
#File describing job to execute
Unit=schedule-test.service #your filename goes here

# Requires=myMonitor.service ##could also try this thing

[Install]
WantedBy=timers.target
```

Now enable the .service first with 
``` systemctl enable schedule-test.service```
or
``` systemctl --user enable schedule-test.service ```


Then if they linked successfully, try 
```
systemctl --user start schedule-test.service #leave out user if not in /user
```

Use systemctl status to check the output of that above command to make sure that the service is running correctly.
If it is running correctly, enable and start the .timer
```
systemctl --user enable schedule-test.timer #leave out user if not in /user
systemctl --user start schedule-test.timer
```
Use systemctl status to check that these are also running correctly and running at the specific interval (one minute).
You could also try 
```
journalctl -S today -f -u myMonitor.service #try with .timer as well
```

Notes for this section:
https://fedoramagazine.org/systemd-timers-for-scheduling-tasks/
https://opensource.com/article/20/7/systemd-timers

https://fedoramagazine.org/systemd-timers-for-scheduling-tasks/
https://wiki.archlinux.org/title/Systemd/Timers

https://wiki.archlinux.org/title/Dm-crypt/System_configuration#cryptkey
https://wiki.archlinux.org/title/Dm-crypt/Device_encryption#Configuring_LUKS_to_make_use_of_the_keyfile


## Set up VPN and ufw rules

This section details setting up Openvpn and a ufw killswitch.

First install the ufw and openvpn packages. You should be able to see a systemctl status for ssh, sshd, and openvpn.

First turn off wifi, then connect and ethernet if one is avaible. You should also log into your vpn service and set up a port for port forwarding, and then download a linux config file. For mullvad, it should look like mullvad_openvpn_linux_xxx.zip

We will now enable ufw and start setting up the rules.

    sudo ufw default deny outgoing
    sudo ufw default deny incoming
    sudo ufw allow out on tun0 from any to any //allows outgoing traffic to vpn
    sudo ufw allow in on tun0 from any to any //allows incoming traffic to vpn, for seeding and stuff
    sudo ufw allow out 1195/udp //for these lines you will need the port from the vpn conf file, this should be the most likly.
    sudo ufw allow in 1195/udp
    sudo ufw allow out 22
    sudo ufw allow in 22 //you will need to change these to be whatever port you have forwarded on your vpn, like 58444
    sudo systemctl enable ufw //make sure to enable this so it starts at boot!

Now that is done, so we can try connecting to the vpn. Unzip the zip file and copy the follow files to /etc/openvpn/ using sudo.

    mullvad_ca.crt
    mullvad_xx.conf
    mullvad_userpass.txt
    update-resolv-conf (may need to be chowned)

Now that that is copied, you can start openvpn with sudo openvpn --config /etc/openvpn/mullvad_xx.conf (where xx is the country or region you selected).

If you want to connect with systemctl/systemd, you might have to copy all the files into /etc/openvpn/client. In order to do this, you need to give the client folder the right permissions by doing:

    sudo chown openvpn:network client/
    sudo chown {username} client/ Then you can do sudo systemctl start openvpn-client@name_of_conf_without_.conf

Check for more info here: https://mullvad.net/en/help/linux-openvpn-installation/

To check whether or not you are connected to Mullvad, you can run curl https://am.i.mullvad.net/connected. Make sure that the openvpn service is enabled too so that it connectes on startup.



##Set up SSH and keyfiles**

This section details how to setup a secure SSH service on this system.

Arch by default uses PAM password authenticaiton for cleartext passwords, but it is not needed for keyfiles. 

Start by installing ssh with pacman.
```sudo pacman -S openssh```

Then start and enable the systemd.
```sudo systemctl enable/start sshd```

Now try sshing into the machine by using user@ip you set earlier in static ip config. If that works begin editing the sshd_config file.
```
sudo nano /etc/ssh/sshd_config
```

Start by changing the port
```

#Port 22 (change this)
#AddressFamily any
#ListenAddress 0.0.0.0
#ListenAddress ::

```

And change the default login settings
```
#LoginGraceTime 2m (could be 30s)
#PermitRootLogin yes (change to no)
#StrictModes yes
#MaxAuthTries 6 (change to 3)
#MaxSessions 10 (change to 1 or 2?)
```

Try setting this to no:
```
#X11Forwarding no (uncomment me)
```

Then save and restart sshd with systemctl restart. Make sure it still works, but while your logged in run ```ssh-keygen```.

Make sure it ran ok by checking that the ~/.ssh directory got filled.
```
ls ~/.ssh
```

Then from another machine, run ssh-copy-id:
```ssh-copy-id user@hostname.example.com -p portnumber```

Make sure both ~/.ssh directories have chmod 700 permissions (so no one else can read them). Restart both sshd daemons.
``` chmod -R 700 ~/.ssh ```

Now trying logging into the arch machine with ssh, this error might come up:
```  ```

Try:
``` ssh-add ```


## Setting up wifi hotspot

If you want to share the VPN network you just set up, or maybe just have a way to SSH into the system when its not connected to internet, this section details setting up a wifi hotspot that will relay requests through the arch system network.

To install wifi hotspot, run

```sudo pacman -S create_ap```

And now you will need to look at the [github docs](https://github.com/lakinduakash/linux-wifi-hotspot) for how to set up the kind of access point that you want. Once you have that picked up look at the kind of adapter you want to use using 

```ip address```

and test it out to make sure that it works. You might need to disable ufw and connect the vpn. If that works, then you can add this ufw rule to allow traffic through the right interface:

```sudo ufw allow in on wlan0```
```sudo ufw allow out on wlan0```

Then edit the default config file to set up the wifi interface how you want, then enable and start the service in systemd. It will be called create_ap and the config file can be found in the .service file. You also might need to make some modifications to the service file to make it restart every 12 hours, as that can stop the "Interface-disabled" problem. The timer on our system is called create-ap-restart.timer.

Make sure that the wifi interface is plugged into the same usb port every time, otherwise it will show up as a different name and not work because there will be a different name in the config file.


### The rest of this guide is setting up some default applications for media sharing that I installed to test the wifi hotspot and it's quality. If you are interested, keep reading, but they are also needed for anything.


## Setting up lanraragi

Make a git clone and unzip it until you have a place where you can find a .json file. You might have to look around in weird directories, try a complete search of all directories looking for the .json file. Then you can try running it with 

```npm start```

For troubleshooting, you might have to install a bunch of perl5 scripts and also never run this in sudo! If you are getting perl5 errors do not try to download them indivdually, try pacman or yay or something else. If you are getting a redis-server error try running 
```redis-server &```
or look into makign redis-server run in the background? 

However this is not good enough if we are looking to run it in the background. What you have to do is intsall [PM2](https://medium.com/idomongodb/how-to-npm-run-start-at-the-background-%EF%B8%8F-64ddda7c1f1). Then all you need to do is run this command in the directory that the orginal command worked in.

```pm2 --name HelloWorld start npm -- start```

To make it run automatically, run these commands in order:

```pm2 save```
```pm2 startup```
```systemctl enable pm2-username```


## Installing Jellyfin + Permissions

Install jellyfin by running 
```yay -S jellyfin-bin```

Then enable it in systemctl. Then go to the .servicefile and add this line below the user line if it is not already there.
```Groups=users```

Then added the jellyfin user to the group users on Arch. Here are some resources for doing that:
https://linuxhint.com/give-user-folder-permission-linux/
https://www.cyberciti.biz/faq/linux-list-all-members-of-a-group/
https://www.howtogeek.com/50787/add-a-user-to-a-group-or-second-group-on-linux/
https://wiki.archlinux.org/title/File_permissions_and_attributes#Changing_permissions

Also make sure that all the directories leading up to the one doing the storage has alteast a read and execute privledge. Try it with all users first, then try restricting it to just the group users. However, jellyfin does need both read and execute privledge the whole way down.
