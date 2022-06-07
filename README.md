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

>NOTE: You should now be able to follow any of the [other guides](#other-software-made-usable-by-the-patched-sdl) in this document. If you're looking for one of them instead of the Super Mario 64 PC port, skip there.

15. Download the Super Mario 64 PC port and place your Super Mario 64 ROM in the repository root directory, replacing `baserom.us.z64` with your ROM's filename (any localization):
>NOTE: For information on how to obtain a Super Mario 64 ROM, see [here](https://github.com/sanni/cartreader).

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
>NOTE: **You should now be able to see and play the game.** If the performance isn't satisfactory, or you want to use this patched SDL for other purposes, keep reading.

## Other software made usable by the patched SDL:
### mupen64plus
>NOTE: For reasons beyond my current understanding, with the steps in this guide, and all other things being equal, Super Mario 64 performs better on BBB in `mupen64plus-video-gles2n64` than it does in `sm64ex`, making mupen64plus a good choice as long as there is no need for mods that only work on PC port, not real console - or if there is need for mods that only work on console and emulator.

1. Download, compile and install mupen64plus and the `gles2n64` video plugin for it:
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

2. Run mupen64plus, where `baserom.us.z64` is the path to your Super Mario 64 ROM. `pvrsrvctl` only needs to be run once since the last time it was manually stopped or the BBB was rebooted. If you are using a GameCube Controller Adapter with vendor and device ID `057e:0337`, run `wii-u-gc-adapter` from step 17 above:
>NOTE: for more info and troubleshooting the `pvrsrvctl` command, scroll [here](#initializing-powervr-driver).
```
$ lsusb
# ~/wii-u-gc-adapter/wii-u-gc-adapter &
# pvrsrvctl --start --no-module
$ mupen64plus --fullscreen --gfx mupen64plus-video-n64 baserom.us.z64
```

### Half-Life 2
#### Prerequisites:
* GNU/Linux PC with an internet connection and a MicroSD card slot or USB A Female port
* BeagleBone Black prepared as directed in steps 1-14 of the above guide
* 10GB+ MicroSD card or USB flash drive depending on the ports available on the PC
* USB A Male to Mini-B Male cord
* Micro-HDMI Male to HDMI Male cord
* Display with an HDMI Female port
* 2+ port powered USB hub
* USB mouse and keyboard

**WARNING: BE CAREFUL WITH UNPOWERED USB HUBS! THE BBB CANNOT SUPPLY MUCH CURRENT. RATHER THAN TRYING TO CALCULATE THAT AND PUSH THE LIMIT, IT IS SAFER TO USE A POWERED HUB.**

#### Installation:
1. Copy the `ep2`, `episodic`, `hl2`, `lostcoast`, `platform` and `steam_input` folders from your Half-Life 2 installation on PC into a new folder named `halflife2` on an empty USB flash drive or SD card formatted with a UNIX-like filesystem (such as `ext4`) with at least 10GB of free space, then unmount the drive, `sync`, remove it from your PC and plug it into the BBB. 
>NOTE: For information on how to obtain a copy of Half-Life 2, see [here](https://store.steampowered.com/app/220/HalfLife_2/).

2. On the BBB, identify and mount the drive, where `/dev/XY` is the block device name of your drive's UNIX-like partition:
```
$ lsblk
# mount /dev/XY /mnt
$ cd /mnt
$ mkdir -p halflife2/custom/nillerusr
$ wget http://nillerusr.fvds.ru/shaders.tar.xz
$ tar xvf shaders.tar.xz -C halflife2/custom/nillerusr
```

3. Download, patch, compile and install nillerusr's actively-maintained leaked-source Source Engine into the Half-Life 2 assets folder:
```
# apt-get install -y libfontconfig-dev libbz2-dev libcurl4-openssl-dev libjpeg-dev libopenal-dev libopus-dev python2.7-dev python-is-python2
$ git clone --recursive --depth 1 -b sanitize https://github.com/nillerusr/source-engine
$ cd source-engine
$ curl https://raw.githubusercontent.com/robertkirkman/sm64ex-bbb-doc/main/source-engine-bbb.patch | git apply -v
$ ./waf configure -T debug --prefix=/mnt/halflife2
$ ./waf install
```

4. Prepare input devices (keyboard and mouse), navigate to the new Half-Life 2 installation, start the PowerVR userspace daemon if it's not already running, and run `hl2_launcher`:
```
$ cd /mnt/halflife2
# pvrsrvctl --start --no-module
$ export LD_LIBRARY_PATH=/mnt/halflife2/bin
$ ./hl2_launcher
```

### Bad Apple!!
>The famous Touhou music video

#### Prerequisites:
* BeagleBone Black prepared as directed in steps 1-14 of the above guide
* 10GB+ MicroSD card or USB flash drive
* USB A Male to Mini-B Male cord
* Micro-HDMI Male to HDMI Male cord
* Display with an HDMI Female port

#### Installation:
1. On the BBB, identify and mount the drive, where `/dev/XY` is the block device name of your drive's UNIX-like partition:
```
$ lsblk
# mount /dev/XY /mnt
```

2. Patch and compile FFmpeg, which requires patching to detect and link against our graphics driver, being very careful not to accidentally install any packages that conflict with the BBB's graphics driver:
```
# apt-get install -y libass-dev libgnutls28-dev libmp3lame-dev libvorbis-dev meson ninja-build texinfo yasm libvpx-dev python3-pip
$ cd /mnt
$ git clone https://github.com/FFmpeg/FFmpeg.git
$ cd FFmpeg
$ curl https://raw.githubusercontent.com/robertkirkman/sm64ex-bbb-doc/main/ffmpeg-bbb.patch | git apply -v
$ PATH="$HOME/bin:$PATH" PKG_CONFIG_PATH=/mnt/FFmpeg/ffmpeg_build/lib/pkgconfig ./configure \
  --prefix=/mnt/FFmpeg/ffmpeg_build \
  --pkg-config-flags="--static" \
  --extra-cflags="-I/mnt/FFmpeg/ffmpeg_build/include" \
  --extra-ldflags="-L/mnt/FFmpeg/ffmpeg_build/lib" \
  --extra-libs="-lpthread -lm" \
  --ld="g++" \
  --bindir="$HOME/bin" \
  --enable-gpl \
  --enable-libvpx \
  --enable-libopus \
  --enable-libvorbis \
  --enable-opengl
$ PATH="$HOME/bin:$PATH" make
$ make install
```

3. Install yt-dlp, download the official Bad Apple!! video, convert it to grayscale raw bitmap format, and play it using OpenGL (ES 2.0) backend in an SDL window:
```
$ cd /mnt
$ python3 -m pip install -U yt-dlp
$ PATH="$HOME/.local/bin:$PATH" yt-dlp -o badapple.mp4 -f 137 https://www.youtube.com/watch?v=i41KoE0iMYU
$ PATH="$HOME/bin:$PATH" ffmpeg -i badapple.mp4 -vsync drop -vf scale=640:480 -c:v rawvideo -pix_fmt gray badapple_480_gray.yuv
# pvrsrvctl --start --no-module
$ PATH="$HOME/bin:$PATH" ffmpeg -video_size 640x480 -re -pix_fmt gray -i badapple_480_gray.yuv -filter:v fps=30 -f opengl "badapple"
```

### GZDoom

1.
>NOTE: `libopus-dev` conflict with Half-Life 2
```
$ cd
$ wget -O ~/.config/gzdoom/doom.wad https://distro.ibiblio.org/slitaz/sources/packages/d/doom1.wad
# apt-get install -y libgme-dev libmpg123-dev timidity
$ git clone https://github.com/coelckers/ZMusic.git
$ cd ZMusic
$ mkdir build
$ cd build
$ export CFLAGS="-O3 -flto -march=armv7-a -marm -mfpu=neon -mfloat-abi=hard -mtune=cortex-a8" 
$ cmake .. -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=/usr
$ make
# make install
```

2.
```
$ cd
$ git clone https://github.com/coelckers/gzdoom.git
$ cd gzdoom
$ mkdir build
$ cd build
$ export CFLAGS="-O3 -flto -march=armv7-a -marm -mfpu=neon -mfloat-abi=hard -mtune=cortex-a8" 
$ cmake .. -DNO_FMOD=ON -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=/usr
$ make
# make install
```

3.
```
# pvrsrvctl --start --no-module
$ gzdoom -gles2_renderer
```

## Other software that can work with SGX530

### [TODO: try to update guide] Qt/Quick EGLFS and Weston Wayland compositor
>NOTE: The guide at the link below is very outdated and uses much older versions of the graphics driver and other software than I use in this repository, which are all completely incompatible, and if you try to follow the steps there, you will have to start over again before anything here will work again; however, until I create a replacement, it is very good for historical reference and helped me out a lot when I started trying to use the BBB.

Years ago, Remi Avignon's eLinux wiki page was the best SGX530 tutorial, before I created this repository. It includes directions for using Qt/Quick EGLFS and Weston compositor (separately) on BBB, which I have verified do work if you carefully use the exact versions he mentions of all the software involved. If you are interested, you can read it [here](https://elinux.org/BeagleBoneBlack/SGX_%2B_Qt_EGLFS_%2B_Weston).

### [TODO: fill with all working demos] Small OpenGL ES 1.1/2.0 Demos
>Very useful for troubleshooting, learning or just using as screensavers
1. kmscube
```
$ cd
$ git clone https://github.com/robertkirkman/kmscube.git
$ cd kmscube
$ meson setup build
$ meson compile -C build
$ cd build
# pvrsrvctl --start --no-module
$ ./kmscube
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

### Initializing PowerVR Driver
Most graphics drivers handle their equivalents to this program robustly and automatically, and if you would like, you may automate this with systemd; however, `pvrsrvctl` is very buggy, and I think it's important to be familiar with this process for troubleshooting and understanding purposes. Here are some common errors that a lot of OpenGL ES 2.0 software will print if you try to execute it before the PowerVR driver is initialized:
```
PVR:(Error): PVRSRVBridgeCall: Failed to access device.  Function ID:3223086849 (strerror returns no value.). [0, ]
PVR:(Error): PVRSRVEnumerateDevices: BridgeCall failed [0, ]
PVR:(Error): PVRSRVConnect: Unable to enumerate devices. [0, ]
PVR:(Error): Couldn't connect to services [0, ]
PVR:(Error): PVRDRIEGLGlobalDataInit: PVR Services initialisation failed [0, ]
PVR:(Error): PVRDRICreateScreenImpl: Couldn't create EGL global data [0, ]
MESA-LOADER: failed to open kms_swrast (search paths /usr/lib/dri)
failed to load driver: kms_swrast
MESA-LOADER: failed to open swrast (search paths /usr/lib/dri)
failed to load swrast driver
Unable to create default window: EGL not initialized
Cannot create default SDL window.
```

To check whether the SGX530 is ready to execute EGL and OpenGL ES 2.0 calls, use this:
```
# dmesg | grep -e PVR -e pvr
```
The first thing you should see is this:
```
pvrsrvkm: loading out-of-tree module taints kernel.
[drm] Initialized pvr 1.17.4948957 20110701 for 56000000.gpu on minor 1
```
If you don't see this or the numbers are different, unfortunately you might have the wrong boot image, device tree, kernel, or kernel module installed. In that case, review steps 1-12 again and make sure you didn't miss anything.

The next thing you should see is this:
```
PVR_K: UM DDK-(4948957) and KM DDK-(4948957) match. [ OK ]
```
If you don't, then `pvrsrvctl` has not been run yet. It needs to be run only once each time the BBB boots. Use this command to run it:
```
# pvrsrvctl --start --no-module
```
These arguments work the best for me, and if you try to use `pvrsrvctl` a different way, unfortunately it might not work correctly. If you want to experiment or improve on this method, though, feel free to. After doing this, you should now see the "`UM and KM match OK`" message in your kernel log, and can proceed to use the GPU for as long as this BBB stays on. If you don't, you might have the wrong user-space driver installed, so you should carefully review step 12.

### All I want to do is turn off the stupid irritating flashing LEDs
```
# bash -c 'for i in $(seq 0 3); do echo 0 > "/sys/class/leds/beaglebone:green:usr${i}/brightness"; done'
```

# CREDITS:

>Huge thanks to zmatt on #beagle at irc.libera.chat, vanfanel, Rob Clark, Remi Avignon, Robert Nelson, and all the engineers at Texas Instruments and Imagination Technologies, without whom playing Super Mario 64 on BeagleBone Black would not be possible.

>Thank you to benedani on The Cult Discord guild for the B3313 mod for helping with the mupen64plus setup

>Thank you very much to nillerusr for developing the leaked-source version of Half-Life 2 and helping me install it