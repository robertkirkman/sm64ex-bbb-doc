# How to play Super Mario 64 PC port on BeagleBone Black
#### This is also, by necessity, a guide on how to use the BBB's Imagination Technologies PowerVR SGX530 GPU

### Prerequisites:
* GNU/Linux PC with an internet connection, a MicroSD card slot and a USB A Female port
* Original BeagleBoard BeagleBone Black - YMMV with variants
* 2GB+ MicroSD card
* USB A Male to Mini-B Male cord
* Micro-HDMI Male to HDMI Male cord
* Display with an HDMI Female port
* USB gamepad or keyboard of choice - Nintendo GameCube controller with official USB adapter is used as example

### Installation:
1. On the PC, download and extract the newest Debian 11 minimal monthly snapshot image for AM335x from https://rcn-ee.com, a domain found in the `sources.list` of the official 2020-04-06 compressed image of Debian 10 for the BeagleBone Black from the official [beagleboard.org downloads page](https://beagleboard.org/latest-images):

```
$ export newest=$(curl https://rcn-ee.com/rootfs/bb.org/testing/ | grep -Eo "[0-9]+-[0-9]+-[0-9]+/" | tr -d '/' | tail -n1)
$ wget https://rcn-ee.com/rootfs/bb.org/testing/$newest/bullseye-minimal/am335x-debian-11.2-minimal-armhf-$newest-2gb.img.xz
$ wget https://rcn-ee.com/rootfs/bb.org/testing/$newest/bullseye-minimal/am335x-debian-11.2-minimal-armhf-$newest-2gb.img.xz.sha256sum
$ sha256sum --check am335x-debian-11.2-minimal-armhf-$newest-2gb.img.xz.sha256sum
$ xz -vd am335x-debian-11.2-minimal-armhf-$newest-2gb.img.xz
```

2. Insert the MicroSD card and run `lsblk` to identify its block device name, `X` where `X` is a file in `/dev/`, then write the extracted image to the card:

**WARNING: THIS DELETES PREEXISTING CONTENT ON THE CARD, AND IF YOU CHOOSE THE WRONG DEVICE, YOU WILL LOSE DATA**

>NOTE: For the following command requiring root, indicated by `#`, use `sudo -E` for a single command or `sudo -E su` to obtain the root shell to preserve the `$newest` environment variable, for convenience.

```
$ lsblk
# dd if=am335x-debian-11.2-minimal-armhf-$newest-2gb.img of=/dev/X bs=1M status=progress oflag=sync
```

3. Use `lsblk` again to identify the root partition of the newly flashed MicroSD card `/dev/XY`, mount it, enable the auto-flasher, and set the display resolution to 640x480, both of which can help performance later, then unmount `/dev/XY`:

**WARNING: THIS WILL DELETE PREEXISTING CONTENT ON THE BBB WHEN IT TURNS ON, TO RUN DEBIAN FROM THE SD CARD SKIP THIS STEP**

>NOTE: If you use auto-mounting services, skip `mount` and replace `tmproot` with the auto-mounted mountpoint for `/dev/XY`.

```
$ lsblk
# mkdir tmproot
# mount /dev/XY tmproot
# sed -i -e 's/#cmdline/cmdline/g' -e 's/1024x768/640x480/g' -e '/tmpfs/s/^/#/' tmproot/boot/uEnv.txt
# umount tmproot
```

4. Remove the MicroSD card from the PC, insert it into the BBB, and plug the BBB into the PC using the USB cord. The BBB will automatically turn on, boot from the MicroSD card, install Debian into the BBB's internal flash, and turn off again. Once it turns off, remove the MicroSD card and push the BBB's power button to turn it back on. If you skipped step 3, the BBB will not overwrite the internal flash or turn off, instead leave the MicroSD card inserted and proceed.

5. **[TODO: clean up this step]** The BBB will take some time to boot. Once it finishes, enable ip forwarding and set the RNDIS network adapter to which address 192.168.7.1 has been assigned to shared to other computers.

```
$ ip a
# sysctl net.ipv4.ip_forward=1
# iptables -t nat -A POSTROUTING -o eno1 -j MASQUERADE
# iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
# iptables -A FORWARD -i enp0s20u12u1 -o eno1 -j ACCEPT
```

6. Log into the BBB, using the default password `temppwd`:

>NOTE: Each time the BBB boots a clean image, it regenerates its `ssh` host keys. If you reach this step multiple times on the same PC, you might need to manage your `~/.ssh/known_hosts` file accordingly.

```
$ ssh debian@192.168.6.2
```

7. Now on the BBB, obtain a root shell and choose a new secure password for the `debian` user:

```
$ sudo su
# passwd debian
```

8. Route internet traffic over the other virtual adapter and apply a DNS server address, replacing `1.1.1.1` with the DNS address of your choice:

```
# sed -i '/^Addr.*/a Gateway=192.168.7.1\nDNS=1.1.1.1' /etc/systemd/network/usb0.network
# systemctl restart systemd-networkd
```

9. Set the system time zone, replacing `America/Chicago` with your preferred choice, and sync the system time with 0.debian.pool.ntp.org:

```
# timedatectl set-timezone America/Chicago
# systemctl restart systemd-timesyncd
```

10. Check for updates, update Debian, and install all the precompiled dependencies needed, then reboot the BBB; These include the open-source kernel-space portion of the PowerVR SGX530 GPU's graphics driver and build dependencies for SDL2:

```
# apt-get update
# apt-get upgrade -y
# apt-get install -y bbb.io-kernel-5.10-ti-am335x build-essential cmake pkg-config \
                     libdrm-dev libwayland-server0 libwayland-client0 libasound2-dev \
                     libudev-dev libevdev-dev libusb-dev libxkbcommon-dev python3
# reboot
```

11. Log into the BBB again, using your new password:

```
$ ssh debian@192.168.6.2
```

12. Now on the BBB again, download and install the proprietary user-space portion of the PowerVR SGX530 GPU's graphics driver:

>NOTE: This driver provides OpenGL ES 2.1, EGL 1.5 and a hard-forked GBM that only functions with the RGB565 pixel format due to hardware limitations, for strictly KMS/DRM contexts, and silently conflicts with any other implementation of those libraries (such as those in the Debian official repositories). No open-source user-space driver for any PowerVR GPU has ever existed.

```
$ git clone -b 1.17.4948957-next git://git.ti.com/graphics/omap5-sgx-ddk-um-linux.git
$ cd omap5-sgx-ddk-um-linux
# TARGET_PRODUCT=ti335x make install
```

13. Download the SDL development repository and patch it with changes of my own design that exclusively make it compatible with BeagleBone Black:

```
$ cd
$ git clone https://github.com/libsdl-org/SDL.git
$ cd SDL
$ curl https://raw.githubusercontent.com/robertkirkman/sm64ex-bbb-doc/main/sdl-bbb.patch | git apply -v
```

14. Compile and install SDL:

```
$ mkdir build
$ cd build
$ CFLAGS="-O3 -march=armv7-a -marm -mfpu=neon -mtune=cortex-a8" ../configure \
      --prefix=/usr \
      --enable-video-kmsdrm \
      --enable-video-opengles \
      --enable-video-opengles2 \
      --enable-alsa \
      --disable-video-vulkan \
      --disable-video-opengl \
      --disable-video-opengles1 \
      --disable-video-dummy \
      --disable-video-x11 \
      --disable-pulseaudio \
      --disable-diskaudio \
      --disable-dummyaudio \
      --disable-oss \
      --disable-kmsdrm-shared \
      --disable-alsa-shared
$ make
# make install
```

15. Download the Super Mario 64 PC port and place your Super Mario 64 ROM in the repository root directory, replacing `baserom.us.z64` with your localization's filename:

>NOTE: For information on how to obtain a Super Mario 64 ROM, see here: https://github.com/sanni/cartreader

```
$ cd
$ git clone https://github.com/sm64pc/sm64ex.git
$ cp baserom.us.z64 sm64ex
```

16. Patch the Super Mario 64 PC port with minor changes that improve the compilation experience on BBB, then compile it:

```
$ cd sm64ex 
$ curl https://raw.githubusercontent.com/robertkirkman/sm64ex-bbb-doc/main/sm64ex-bbb.patch | git apply -v
$ BETTERCAMERA=1 EXTERNAL_DATA=1 TEXTURE_FIX=1 make
```

17. Prepare an input device. As an example I use an official Nintendo GameCube controller with the official Nintendo USB GameCube Controller Adapter and ToadKing's open-source user-space driver. Plug the controller into the adapter, Connect the adapter to the BBB's USB A port via the black connector, then download, compile and run the driver:

```
$ cd
$ git clone https://github.com/ToadKing/wii-u-gc-adapter.git
$ cd wii-u-gc-adapter
$ make
$ ./wii-u-gc-adapter &
```

18. Connect the display to the BBB using the HDMI cord, then initialize the user-space graphics driver and run the Super Mario 64 PC port:

NOTE: To hear the game audio, your display must have speakers or a 3.5mm jack for HDMI audio.

```
$ cd
# pvrsrvctl --start --no-module
$ sm64ex/build/us_pc/sm64.us.f3dex2e
```

>CREDITS:
>Huge thanks to zmatt on #beagle at irc.libera.chat, vanfanel, Rob Clark, Remi Avignon, Robert Nelson, and all the developers at Texas Instruments and Imagination Technologies, without whom playing Super Mario 64 on BeagleBone Black would not be possible.
