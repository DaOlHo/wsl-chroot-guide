# Guide to using Chroot on WSL with an actual install

## Mount your drives into WSL

On powershell: `GET-CimInstance -query "SELECT * from Win32_DiskDrive"`

This should give you something back that looks like this:
```
DeviceID           Caption                         Partitions Size          Model
--------           -------                         ---------- ----          -----
\\.\PHYSICALDRIVE0 ST2000DM008-2FR102              1          2000396321280 ST2000DM008-2FR102
\\.\PHYSICALDRIVE3 KBG30ZMS256G NVMe TOSHIBA 256GB 2          256052966400  KBG30ZMS256G NVMe TOSHIBA 256GB
\\.\PHYSICALDRIVE2 Samsung SSD 970 EVO 500GB       2          500105249280  Samsung SSD 970 EVO 500GB
\\.\PHYSICALDRIVE1 ST4000DM004-2CV104              1          4000784417280 ST4000DM004-2CV104
```

Find the drive that you have your Linux install on, for the purposes of this guide I will call it `\\.\PHYSICALDRIVEX` (always remember to replace the X with your actual device ID!).

Mount both EFI and Root partitions into WSL: `wsl --mount \\.\PHYSICALDRIVEX -p 2 & wsl --mount \\.\PHYSICALDRIVEX -p 1 -t vfat` (This guide assumes the EFI partition is partition 1 and the root partition is partition 2. This needs to be on a seperate drive, as WSL will not mount and EXT4 filesystems on disks that already have NTFS filesystems)

## Use Systemd in WSL (if you use Systemd as an init system on your actual install)

For whatever reason, internet refused to work for me unless I did this.

Follow this guide to install Genie onto WSL: https://gist.github.com/djfdyuruiry/6720faa3f9fc59bfdf6284ee1f41f950

If when you start up Genie, it prints `Waiting for WSL!` and stops to print more exclemation marks, you'll need to edit your timeout to be 0. Genie is started, systemctl works, and if you `ctrl+c` and try to start it again it'll say it's already been started. For the purposes of things being just a tad less painful:

edit `/etc/genie.ini`, and change the `systemd-timeout=240` line to `systemd-timeout=0`. Reboot WSL and there should no longer be a timeout.

## Ensuring internet works

WSL by default auto-generates the resolv.conf with some IP that never works, at least for me. Here's how to fix that (this matters because you'll be binding this file to your new root later).

```bash
$ sudo rm /etc/resolv.conf
$ sudo bash -c 'echo "nameserver 8.8.8.8" > /etc/resolv.conf'
$ sudo bash -c 'echo "[network]" > /etc/wsl.conf'
$ sudo bash -c 'echo "generateResolvConf = false" >> /etc/wsl.conf'
$ sudo chattr +i /etc/resolv.conf
```

The nameserver 8.8.8.8 is google's DNS, which actually works.

## Creating the Chroot script

There are a couple things you need to do every time you chroot to make sure *most* stuff works.

Create a new file in a text editor, in my case it was `~/chroot.sh`:

```bash
#!/usr/bin/env bash

mount --bind /dev /mnt/wsl/PHYSICALDRIVEXp2/dev                         # These are not present on a system you don't boot,
mount --bind /tmp /mnt/wsl/PHYSICALDRIVEXp2/tmp                         # so for everything to work right you need these.
mount --bind /proc /mnt/wsl/PHYSICALDRIVEXp2/proc                       # ^
mount --bind /etc/resolv.conf /mnt/wsl/PHYSICALDRIVEXp2/etc/resolv.conf # This is needed so your internet works
mount --bind /mnt/wslg /mnt/wsl/PHYSICALDRIVEXp2/mnt/wslg               # This is so you can pass through windows to windows
mount --bind /mnt/wsl/PHYSICALDRIVEXp1 /mnt/wsl/PHYSICALDRIVEXp2/boot   # Mount EFI partition

cd /mnt/wsl/PHYSICALDRIVEXp2/

# Delete existing X socket (if exists) and use the one generated by WSL
[ -S tmp/.X11-unix ] && sudo rm tmp/.X11-unix
[ -f tmp/.X11-unix ] && sudo rm tmp/.X11-unix
[ -L tmp/.X11-unix ] && sudo rm tmp/.X11-unix
sudo ln -rs mnt/wslg/.X11-unix/ tmp/.X11-unix

# Finally chroot into your install, passing through environment variables needed for window pass through, and lastly switching to your user.
# (Of course, replace USER with your username on your linux install)
chroot /mnt/wsl/PHYSICALDRIVEXp2 /usr/bin/env WSL_INTEROP="$WSL_INTEROP" DISPLAY="$DISPLAY" WAYLAND_DISPLAY="$WAYLAND_DISPLAY" XDG_RUNTIME_DIR="$XDG_RUNTIME_DIR" /bin/bash -c "su USER"
```

You should now be able to chroot into your install by running `$ sudo ./chroot.sh`

## Creating a windows terminal profile for this to make the chroot process less annoying

Create a new WSL profile, set icons, name, and whatnot to your liking. In the "command line" section, paste the following, replacing USER and X appropriately:

`cmd /c wsl --mount \\.\PHYSICALDRIVEX -p 2 >nul & wsl --mount \\.\PHYSICALDRIVEX -p 1 -t vfat >nul & wsl -d Ubuntu -u root genie -i >null & wsl -u root /home/USER/chroot.sh & wsl --shutdown`

The `cmd /c` is there to execute the entire thing, because you can technically only execute one command when starting up a new WT profile. The `/c` also serves to exit the terminal when you exit chroot, which is pretty handy.

Next you mount both drives, with the `>nul` there to hide the mount messages (this is to my taste, you can remove them if you'd rather see them).

Next you start genie on Ubuntu (replace this if you're using a different WSL distro), same deal as above with the `>nul`.

Next, start wsl as root (so you don't need to type in your password when chrooting), and run the script you created.

Lastly, shutdown WSL before closing the WT tab (This is optional, if you want to keep WSL open).
