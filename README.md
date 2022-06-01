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
$ wget https://rcn-ee.com/rootfs/bb.org/testing/$newest/bullseye-minimal/am335x-debian-11.3-minimal-armhf-$newest-2gb.img.xz
$ wget https://rcn-ee.com/rootfs/bb.org/testing/$newest/bullseye-minimal/am335x-debian-11.3-minimal-armhf-$newest-2gb.img.xz.sha256sum
$ sha256sum --check am335x-debian-11.3-minimal-armhf-$newest-2gb.img.xz.sha256sum
$ xz -vd am335x-debian-11.3-minimal-armhf-$newest-2gb.img.xz
```

2. Insert the MicroSD card and run `lsblk` to identify its block device name, `X` where `X` is a file in `/dev/`, then write the extracted image to the card:

**WARNING: THIS DELETES PREEXISTING CONTENT ON THE CARD, AND IF YOU CHOOSE THE WRONG DEVICE, YOU WILL LOSE DATA**

>NOTE: For the following command requiring root, indicated by `#`, use `sudo -E` for a single command or `sudo -E su` to obtain the root shell to preserve the `$newest` environment variable, for convenience. **Throughout this guide, continue to do one of the following, choose your preference:**<br>
**A. use `sudo -E` or `sudo -E su`**<br>
**B. keep track of the environment variables**<br>
**C. change PREFIX/install paths to avoid switching users to the degree that you prefer**<br>

```
$ lsblk
# dd if=am335x-debian-11.3-minimal-armhf-$newest-2gb.img of=/dev/X bs=1M status=progress oflag=sync
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
# iptables -A FORWARD -i enp0s20u12u4 -o eno1 -j ACCEPT
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

10. Check for updates, update Debian, and install all the precompiled dependencies needed, then reboot the BBB; these include the open-source kernel-space portion of the PowerVR SGX530 GPU's graphics driver, build dependencies for SDL2, and runtime dependencies of the user-space portion of the graphics driver:

```
# apt-get update
# apt-get upgrade -y
# apt-get install -y bbb.io-kernel-5.10-ti-am335x build-essential cmake pkg-config \
                     libdrm-dev libwayland-server0 libwayland-client0 libasound2-dev \
                     libudev-dev libevdev-dev libusb-dev libusb-1.0-0-dev \
                     libxkbcommon-dev usbutils
# reboot
```

11. Log into the BBB again, using your new password:

```
$ ssh debian@192.168.6.2
```

12. Now on the BBB again, download and install the proprietary user-space portion of the PowerVR SGX530 GPU's graphics driver:

>NOTE: This driver provides OpenGL ES 2.0, EGL 1.5 and a hard-forked GBM that only functions with the RGB565 pixel format due to [hardware limitations](https://software-dl.ti.com/processor-sdk-linux/esd/docs/latest/linux/Foundational_Components/Graphics/AM3_Beagle_Bone_Black_Configuration.html) (dont follow the directions there, they are outdated), for strictly KMS/DRM contexts, and silently conflicts with any other implementation of OpenGL, EGL or GBM (such as those in the Debian official repositories). In the present day, open-source user-space drivers for [PowerVR Rogue GPUs](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/imagination) and [PowerVR Series1 3D accelerators](https://github.com/powervr-graphics/PowerVR-Series1) exist, but these were designed in the 2010s and 1990s, respectively. The PowerVR SGX530 was designed in the 2000s, and as of yet no user-space source code for its drivers is publicly accessible.

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
$ export CFLAGS="-O3 -march=armv7-a -marm -mfpu=neon -mtune=cortex-a8" 
$ ../configure \
      --prefix=/usr \
      --enable-video-kmsdrm \
      --enable-video-opengles \
      --enable-video-opengles1 \
      --enable-video-opengles2 \
      --enable-alsa \
      --disable-video-vulkan \
      --disable-video-opengl \
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

15. Download the Super Mario 64 PC port and place your Super Mario 64 ROM in the repository root directory, replacing `baserom.us.z64` with your ROM's filename (any localization):

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
$ TARGET_BBB=1 BETTERCAMERA=1 EXTERNAL_DATA=1 TEXTURE_FIX=1 make
```

17. Prepare an input device. As an example I use an official Nintendo GameCube controller with the official Nintendo USB GameCube Controller Adapter and ToadKing's open-source user-space driver. Plug the controller into the adapter, Connect the adapter to the BBB's USB A port via the black connector, then download, compile, and run the driver. You only need to do this if you are using a GameCube Controller Adapter with vendor and device ID `057e:0337` as it appears in `lsusb`:

```
$ lsusb
$ cd
$ git clone https://github.com/ToadKing/wii-u-gc-adapter.git
$ cd wii-u-gc-adapter
$ export CFLAGS="-O3 -flto -march=armv7-a -marm -mfpu=neon -mfloat-abi=hard -mtune=cortex-a8" 
$ make
# ./wii-u-gc-adapter &
```

18. Connect the display to the BBB using the HDMI cord, then initialize the user-space graphics driver and run the Super Mario 64 PC port:

>NOTE: To hear the game audio, your display must have speakers or a 3.5mm jack for HDMI audio.

```
$ cd
# pvrsrvctl --start --no-module
$ sm64ex/build/us_pc/sm64.us.f3dex2e
```
**You should now be able to see and play the game.** If the performance isn't satisfactory, or you want to use this patched SDL for other purposes, keep reading.

## Other software made usable by the patched SDL:
### mupen64plus
>NOTE: For reasons beyond my current understanding, with the steps in this guide, and all other things being equal, Super Mario 64 performs better on BBB in `mupen64plus-video-gles2n64` than it does in `sm64ex`, making mupen64plus a good choice as long as there is no need for mods that only work on PC port, not real console - or if there is need for mods that only work on console and emulator.

1. Install mupen64plus
>NOTE: I have deliberately unrolled the intended Bash loops to call attention to mupen64plus' not-fully-sane build system and make it easier to understand.
```
# apt-get install -y zlib1g-dev libpng-dev libfreetype-dev nasm libboost-dev libboost-filesystem-dev
$ cd
$ git clone https://github.com/mupen64plus/mupen64plus-core.git
$ git clone https://github.com/mupen64plus/mupen64plus-ui-console.git
$ git clone https://github.com/mupen64plus/mupen64plus-audio-sdl.git
$ git clone https://github.com/mupen64plus/mupen64plus-input-sdl.git
$ git clone https://github.com/mupen64plus/mupen64plus-rsp-hle.git
$ git clone https://github.com/ricrpi/mupen64plus-video-gles2n64.git
$ export USE_GLES=1 NEON=1 HOST_CPU=armv7 PREFIX=/usr OPTFLAGS="-O3 -flto -march=armv7-a -marm -mfpu=neon -mfloat-abi=hard -mtune=cortex-a8" 
$ cd ~/mupen64plus-core/projects/unix
$ make all
# make install
$ cd ~/mupen64plus-ui-console/projects/unix
$ make all
# make install
$ cd ~/mupen64plus-audio-sdl/projects/unix
$ make all
# make install
$ cd ~/mupen64plus-input-sdl/projects/unix
$ make all
# make install
$ cd ~/mupen64plus-rsp-hle/projects/unix
$ make all
# make install
$ cd ~/mupen64plus-video-gles2n64/projects/unix
$ make all
# make install
# sed -i -e 's/window\ width=400/window\ width=640/g' -e 's/window\ height=240/window\ height=480/g' /usr/share/mupen64plus/gles2n64.conf
```

2. Run mupen64plus, where `baserom.us.z64` is the path to your Super Mario 64 ROM. If you are using a GameCube Controller Adapter with vendor and device ID `057e:0337`, run `wii-u-gc-adapter` from step 17 above:
```
$ lsusb
# ~/wii-u-gc-adapter/wii-u-gc-adapter &
# pvrsrvctl --start --no-module
$ mupen64plus --fullscreen --gfx mupen64plus-video-n64 baserom.us.z64
```

### Half-Life 2
```
$ cd
$ git clone https://github.com/nillerusr/source-engine --recursive --depth 1
$ cd source-engine
$ ./waf configure -T debug
$ ./waf build
# apt-get install -y libfontconfig-dev libbz2-dev libcurl4-openssl-dev libjpeg-dev libopenal-dev libopus-dev python2.7-dev python-is-python2
```

### Bad Apple
>The famous Touhou music video

```
# apt-get install -y libass-dev libgnutls28-dev libmp3lame-dev libvorbis-dev meson ninja-build texinfo yasm libvpx-dev
$ cd
$ mkdir ffmpeg_sources
$ cd ffmpeg_sources
$ wget -O ffmpeg-snapshot.tar.bz2 https://ffmpeg.org/releases/ffmpeg-snapshot.tar.bz2
$ tar xjvf ffmpeg-snapshot.tar.bz2
$ cd ffmpeg
$ PATH="$HOME/bin:$PATH" PKG_CONFIG_PATH="$HOME/ffmpeg_build/lib/pkgconfig" ./configure \
  --prefix="$HOME/ffmpeg_build" \
  --pkg-config-flags="--static" \
  --extra-cflags="-I$HOME/ffmpeg_build/include" \
  --extra-ldflags="-L$HOME/ffmpeg_build/lib" \
  --extra-libs="-lpthread -lm" \
  --ld="g++" \
  --bindir="$HOME/bin" \
  --enable-gpl \
  --enable-libvpx \
  --enable-libopus \
  --enable-libvorbis 
$ PATH="$HOME/bin:$PATH" make
$ make install
$ yt-dlp https://www.youtube.com/watch?v=i41KoE0iMYU
$ ffmpeg -loop 1 -re -i badapple.mp4 -f kmsdumb /dev/dri/card0
```

## Other... stuff ig

### Stream audio from BBB to PC using pulseaudio `module_native_protocol_tcp`
>NOTE: This uses a lot of CPU. Not a good idea.

1. BeagleBone Black:
```
# apt-get install -y pulseaudio
$ systemctl --user stop pulseaudio.service
```

2. Server (GNU/Linux system with speakers), where `192.168.6.2` is your BBB's IP address:
```
$ systemctl --user stop pulseaudio.service
$ ssh -R 4713:localhost:4713 debian@192.168.6.2
```

3. Server (leave the shell above running and launch a new, separate shell):
```
$ systemctl --user start pulseaudio.service
$ pactl load-module module-native-protocol-tcp auth-anonymous=1
```

4. BeagleBone Black:
```
$ systemctl --user start pulseaudio.service
$ pactl load-module module-native-protocol-tcp
$ export PULSE_SERVER=localhost
$ sm64ex/build/us_pc/sm64.us.f3dex2e
```

### All I want to do is turn off the stupid irritating flashing LEDs
```
# bash -c 'for i in $(seq 0 3); do echo 0 > "/sys/class/leds/beaglebone:green:usr${i}/brightness"; done'
```

# CREDITS:

>Huge thanks to zmatt on #beagle at irc.libera.chat, vanfanel, Rob Clark, Remi Avignon, Robert Nelson, and all the engineers at Texas Instruments and Imagination Technologies, without whom playing Super Mario 64 on BeagleBone Black would not be possible.

>Thank you to benedani on The Cult Discord guild for the B3313 mod for helping with the mupen64plus setup