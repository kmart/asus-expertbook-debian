# Setup of Debian testing (forky) on ASUS ExpertBook Ultra (B9406CAA)

Describes how to install Debian testing (forky) using the netinst CD image
downloadable from the [Network install from a minimal USB,
CD](https://www.debian.org/CD/netinst/) page. Direct link to the amd64 ISO image:
[debian-13.5.0-amd64-netinst.iso](https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-13.5.0-amd64-netinst.iso)

There are a few descriptions on how to install Linux on this ASUS model around, specifically:

- [asus-expertbook-linux](https://github.com/burakgon/asus-expertbook-linux)
- [Omarchy issue #5432](https://github.com/basecamp/omarchy/issues/5423)

Both are for [archlinux](https://archlinux.org/), but the descriptions are so good that it
was easy to adapt the suggestions/steps to the Debian setup. The kernel itself have also catched up a bit on
support for this specific model and some things that that earlier required fixes now works out of the box.

## Debian installation

Two USB sticks will be required. One for the ISO image itself and optionally
one for the `wpa_supplicant.conf` file required to get the wireless network working.

### Prepare a USB stick with the image

Download the image and write it to the USB stick with:

```
dd if=/tmp/debian-testing-amd64-netinst.iso of=/dev/sdd bs=1M
```

### Boot from the USB stick

Hold down the Esc key and the press the power up key. Keep on holding down the Esc key until the
boot menu show up.

Inside the BIOS change the following:

- Fast boot: disable
- Secure boot: disable

Insert the USB stick in one of the USB ports and press F10 to save the BIOS changes (or use the menu).

Again hold down the Esc key while the machine is booting. This should bring up the boot menu.
From the boot menu select the USB stick.

### Debian installation

Should be straight forward, but there are a few important steps:

#### `root` and user setup

Skip defining a password for `root`. This will cause the `sudo` command to be installed,
which is a "better way" for doing administrative task.

#### Network device detection

The installer will complain about missing firmware files (`intel/ish/ish_ptl_*`).
Can be ignored. The missing files are related to Intel Sensor Hub (ISH).

#### Wireless network setup

Network setup will fail on the get IP address step (DHCP). The reason is that the
installer don't detects that the network interface is actually a wireless card (WiFi).

To get around this there are two option.

1. Use an ethernet USB dongle to provide network connection.

2. Manually set up the wireless card.

The steps for the second option.

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

Select "Go back" which will bring up the main menu. Select "shell" from the menu.

4. Mount the USB stick on the ASUS with the command:

```
mount /dev/sdb1 /mnt
```

(The USB stick should be on `/dev/sdb`. The installation USB stick is on `/dev/sda`.)

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
