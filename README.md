# Setup of Debian testing (forky) on ASUS ExpertBook Ultra (B9406CAA)

Describes how to install Debian testing (forky) using the netinst CD image
downloadable from the [Network install from a minimal USB,
CD](https://www.debian.org/CD/netinst/) page. Direct link to the amd64 ISO image:
[debian-13.5.0-amd64-netinst.iso](https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-13.5.0-amd64-netinst.iso)

There are a few descriptions on how to install Linux on this ASUS model around, specifically:

- [burakgon/asus-expertbook-linux](https://github.com/burakgon/asus-expertbook-linux)
- [Omarchy issue #5432](https://github.com/basecamp/omarchy/issues/5423)

Both descriptions are for [Arch Linux](https://archlinux.org/), but the descriptions are so good that it
was easy to adapt the suggestions/steps to the Debian setup. The kernel itself have also catched up a bit on
support for this specific model and some things that that earlier required fixes now works out of the box.

---

**It looks like that there are different versions of the machine on
the market, with different configurations (LCDs etc.). The following
installation and setup description might therefore not be valid in all
cases. Use at own risk.**

---

## Debian installation

Two USB sticks will be required. One for the ISO image itself and optionally
one for the `wpa_supplicant.conf` file required to get the wireless network working.

### Prepare a USB stick with the installation image

Download the image and write it to the USB stick with:

```
dd if=/tmp/debian-testing-amd64-netinst.iso of=/dev/sdd bs=1M
```

### Boot from the USB stick

Hold down the `Esc` key and start booting by pressing the power key. Keep on holding down the `Esc` key until the
boot menu show up.

Inside the BIOS change the following:

- Fast boot: disable
- Secure boot: disable

Insert the USB stick in one of the USB ports and press `F10` to save the BIOS changes (or use the menu).

Again hold down the `Esc` key while the machine is booting. This should again bring up the boot menu.
From the boot menu select the USB stick.

### Installation

Should be straight forward, but there are a few important steps:

#### `root` and user setup

Skip defining a password for `root`. This will cause the `sudo` command to be installed,
which is a "better" command for doing administrative task.

#### Network device detection

The installer will complain about missing firmware files (`intel/ish/ish_ptl_*`).
Can be ignored. The missing files are related to the Intel Sensor Hub (ISH).

#### Wireless network setup

Network setup will fail on the get IP address step (DHCP). The reason is that the
installer don't detect that the network interface is actually a wireless card.

To get around this there are two options.

1. Use an ethernet USB dongle to provide network connection.

2. Manually set up the wireless card.

The steps for the second option:

1. Prepare a (second) `fat32`/`vfat` formatted USB stick.

2. Create a file name `wpa_supplicant.conf` on the USB stick with the following content:

```
ctrl_interface=/var/run/wpa_supplicant
update_config=1
country=US

network={
    ssid="Your_WiFi_Name"
    psk="Your_WiFi_Password"
    key_mgmt=WPA-PSK
    priority=100
}
```

Set the `ssid` and `psk` values to the correct values for the network to connect to.

(This is for a WPA2/WPA3 network. Modify the configuration file as required for other types of
wireless networks.)

3. Go to the shell

Select "`<Go Back>`" one or two times. This should bring up the main menu. Select "`Execute a shell`" from the menu.

4. Mount the USB stick on the ASUS with the command:

```
mount /dev/sdb1 /mnt
```

(The USB stick should be on `/dev/sdb`. The installation USB stick should be on `/dev/sda`.)

5. Enable the wireless network card with the command:

```
wpa_supplicant -B -i wlo1 -c /mnt/wpa_supplicant.conf

```

Should return no errors.

Exit the shell and continue with the network setup, which now should succeed.

#### Package selection

After having selected the packages to be installed and completed the installation of the
packages, go back to the shell and run the command:

```
apt install firmware-intel-graphics
```

This will install the Intel Arc/Xe graphics driver required for the LCD to work. Without the `xe` driver
installed the LCD will turn black (nothing visible) after boot.

Next complete the installation and boot into the newly installed system.

## Fixes

After boot and with a DM (Gnome/KDE/etc) installed and working some remaining issues requires fixes.
These are:

- LCD brightness can't be adjusted. Stays on 100%.
- No sound.
- Thouchpad not working.

### LCD brightness

Add `xe.enable_dpcd_backlight=1` to the `GRUB_CMDLINE_LINUX_DEFAULT` line in `/etc/default/grub`.

The line should now look like this:

```
GRUB_CMDLINE_LINUX_DEFAULT="xe.enable_dpcd_backlight=1 quiet"
```

Run `update-grub`.

After reboot it should now be possible to adjust the LCD brightness.

### Sound

Install latest version of the `firmware-sof-signed` package. The current version of this package
in testing is old (ver. 2025.05.1) and don't provide the firmware required for enabling sound.

Debian unstable (sid) has the latest version (2025.12.2). Download and install this package with:

```
cd /tmp
wget http://ftp.us.debian.org/debian/pool/non-free-firmware/f/firmware-sof/firmware-sof-signed_2025.12.2-2_all.deb
sudo dpkg -i /tmp/firmware-sof-signed_2025.12.2-2_all.deb
```

After reboot sound (and microphone) should work.

Note that if the old version of the package is installed, the package should probably be removed first.
Can be done with:

```
apt uninstall firmware-sof-signed
```

before installing the new version of the package.

(An alterative to installing the package as described, is to make packages in "unstable" available
through "apt pinning". See Debian documentation for details.)

### Touchpad

Create the file `/etc/udev/hwdb.d/61-pixart-4f05-pressure-fix.hwdb` with the following content:

```
evdev:input:b0018v093Ap4F05*
 EVDEV_ABS_18=0:100:0:0
 EVDEV_ABS_3A=0:100:0:0
```

Update the hardware database with the command:

```
systemd-hwdb update
```

Next create the directory `/etc/libinput` if it don't exists and add the file `/etc/libinput/asus-expertbook-b9406.quirks`
with the following content:

```
[ASUS ExpertBook Ultra B9406 Touchpad]
MatchUdevType=touchpad
MatchBus=i2c
MatchVendor=0x093A
MatchProduct=0x4F05
MatchDMIModalias=dmi:*svnASUS*:pn*B9406*
AttrEventCode=-ABS_MT_PRESSURE;-ABS_PRESSURE;
```

After reboot the touchpad should work.

See [burakgon - touchpad-fix](https://github.com/burakgon/asus-expertbook-linux/tree/main/touchpad-fix) for details.

## Enabling Secure Boot

The ASUS ExpertBook Ultra omits the **Microsoft Corporation UEFI CA 2011** CA certificate required for enabling secure boot using Debian's shims.

To make secure boot work this certificate must be added to the list of valid certificates (enroll).

First obtain the certificate and add it to the boot directory.

```
wget https://www.microsoft.com/pkiops/certs/MicCorUEFCA2011_2011-06-27.crt
sudo mkdir -p /boot/efi/keys
sudo cp MicCorUEFCA2011_2011-06-27.crt /boot/efi/keys/
```

Reboot into BIOS:

1. Go to `Security` and enable "Secure boot". The menues referred to below should now become available.
2. Go to `Key Management -> Authorized Signatures (db)`.
3. Select **Append** (not Replace) -> Browse ESP and select `keys/MicCorUEFCA2011_2011-06-27.crt` and add the certificate.

`F10` to save and boot.

Verify with:

```
mokutil --sb-state
```

Should say that secure boot is enabled.
