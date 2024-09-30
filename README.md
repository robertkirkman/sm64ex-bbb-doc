# How to play Super Mario 64 on BeagleBone Black
#### A guide on how to use the BBB's Imagination Technologies PowerVR SGX530 GPU

[![demo](https://i.ytimg.com/vi_webp/OaUrnq9Qi7U/maxresdefault.webp)](https://youtu.be/OaUrnq9Qi7U?t=66 "sm64ex on BeagleBone Black with hardware 3D acceleration")

### Prerequisites:
* GNU/Linux PC with an internet connection, a MicroSD card slot and a USB A Female port
* Original BeagleBoard BeagleBone Black - YMMV with variants
* 2GB+ MicroSD card
* USB A Male to Mini-B Male cord
* Micro-HDMI Male to HDMI Male cord
* Display with an HDMI Female port
* USB gamepad or keyboard of choice - Nintendo GameCube controller with official USB adapter is used as example

### Installation:
1. On the PC, download and extract the newest Debian 11 snapshot image for AM335x from https://rcn-ee.com, a domain found in the `sources.list` of the official 2020-04-06 compressed image of Debian 10 for the BeagleBone Black from the official [beagleboard.org downloads page](https://beagleboard.org/latest-images):

> [!CAUTION]
> If you want to run directly from the SD card instead of erasing the BBB's internal storage and overwriting it, use [another image that doesn't have `eMMC-flasher` in its name](https://rcn-ee.com/rootfs/snapshot/).

```bash
wget https://rcn-ee.com/rootfs/snapshot/2024-09-07/bullseye-minimal-armhf/am335x-eMMC-flasher-debian-11.11-minimal-armhf-2024-09-07-2gb.img.xz
wget https://rcn-ee.com/rootfs/snapshot/2024-09-07/bullseye-minimal-armhf/am335x-eMMC-flasher-debian-11.11-minimal-armhf-2024-09-07-2gb.img.xz.sha256sum
sha256sum --check am335x-eMMC-flasher-debian-11.11-minimal-armhf-2024-09-07-2gb.img.xz.sha256sum
xz -vd am335x-eMMC-flasher-debian-11.11-minimal-armhf-2024-09-07-2gb.img.xz
```

2. Insert the MicroSD card and run `lsblk` to identify its block device name, `X` where `X` is a file in `/dev/`, then write the extracted image to the card:

> [!CAUTION]
> This deletes preexisting content on the card, and if you choose the wrong device, you will lose data.

```bash
lsblk
sudo dd if=am335x-eMMC-flasher-debian-11.11-minimal-armhf-2024-09-07-2gb.img of=/dev/X bs=1M status=progress oflag=sync
```

3. Remove the MicroSD card from the PC, insert it into the BBB, and plug the BBB into the PC using the USB cord. The BBB will automatically turn on, boot from the MicroSD card, install Debian into the BBB's internal flash, and turn off again. Once it turns off, remove the MicroSD card and push the BBB's power button to turn it back on. 

> [!NOTE]
> If you used a non-flasher image, the BBB will not overwrite the internal flash or turn off, instead leave the MicroSD card inserted and proceed.

4. The BBB will take some time to boot. Once it finishes, on the PC, enable ip forwarding and set the RNDIS network adapter to which address 192.168.7.1 has been assigned to shared to other computers.

> [!TIP]
> If you do this, you would need to identify your network device names and network configuration and replace this with your own, but this is optional, since you can use an ethernet cord for internet access instead if available.

```bash
ip a
sudo sysctl net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -o eno1 -j MASQUERADE
sudo iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i enp0s20u12u4 -o eno1 -j ACCEPT
```

5. Log into the BBB, using the default password `temppwd`:

> [!NOTE]
> Each time the BBB boots a clean image, it regenerates its `ssh` host keys. If you reach this step multiple times on the same PC, you might need to manage your `~/.ssh/known_hosts` file accordingly.

```bash
ssh debian@192.168.6.2
```

6. Now on the BBB, choose a new secure password for the `debian` user:

```bash
sudo passwd debian
```

7. Route internet traffic over the other virtual adapter and apply a DNS server address, replacing `1.1.1.1` with the DNS address of your choice:

```bash
sudo sed -i '/^Addr.*/a Gateway=192.168.7.1\nDNS=1.1.1.1' /etc/systemd/network/usb0.network
sudo systemctl restart systemd-networkd
```

8. Set the system time zone, replacing `America/Chicago` with your preferred choice, and sync the system time with 0.debian.pool.ntp.org:

```bash
sudo timedatectl set-timezone America/Chicago
sudo systemctl restart systemd-timesyncd
```

9. Check for updates, update Debian, and install all the precompiled dependencies needed, then reboot the BBB; these include the kernel module for the SGX530, runtime dependencies of the user-space portion of the graphics driver, and build dependencies for SDL2:

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y bbb.io-kernel-5.10-ti-am335x \
                    build-essential \
                    cmake \
                    pkg-config \
                    libdrm-dev \
                    libwayland-server0 \
                    libwayland-client0 \
                    libasound2-dev \
                    libudev-dev \
                    libevdev-dev \
                    libusb-dev \
                    libusb-1.0-0-dev \
                    libxkbcommon-dev \
                    usbutils
sudo reboot
```

10. Log into the BBB again, using your new password:

```bash
ssh debian@192.168.6.2
```

11. Now on the BBB again, download and install the proprietary user-space portion of the PowerVR SGX530 GPU's graphics driver:

> [!NOTE]
> This driver provides OpenGL ES 2.0, EGL 1.5 and a forked GBM that only functions with the RGB565 pixel format due to [hardware limitations](https://software-dl.ti.com/processor-sdk-linux/esd/docs/latest/linux/Foundational_Components/Graphics/AM3_Beagle_Bone_Black_Configuration.html) (don't follow the directions there, they are outdated), for strictly KMS/DRM contexts, and silently conflicts with any other implementation of OpenGL, EGL or GBM (such as those in the Debian official repositories). In the present day, open-source user-space drivers for [PowerVR Rogue GPUs](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/imagination) and [PowerVR Series1 3D accelerators](https://github.com/powervr-graphics/PowerVR-Series1) exist, but these were designed in the 2010s and 1990s, respectively. The PowerVR SGX530 was designed in the 2000s, and as of yet no user-space source code for its drivers is publicly accessible.

```bash
git clone -b 1.17.4948957-next git://git.ti.com/graphics/omap5-sgx-ddk-um-linux.git
cd omap5-sgx-ddk-um-linux
sudo TARGET_PRODUCT=ti335x make install
cd
```

12. Download the SDL development repository and patch it with changes I designed that make it exclusively compatible with BeagleBone Black:

```bash
git clone -b SDL2 https://github.com/libsdl-org/SDL.git
cd SDL
curl https://raw.githubusercontent.com/robertkirkman/sm64ex-bbb-doc/main/sdl-bbb.patch | git apply -v
```

13. Compile and install SDL:

```bash
mkdir build
cd build
export CFLAGS="-O3 -march=armv7-a -marm -mfpu=neon -mtune=cortex-a8" 
../configure \
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
  --enable-arm-neon \
  --enable-events \
  --enable-libudev \
  --enable-dbus \
  --enable-ibus \
  --disable-alsa-shared
make
sudo make install
cd
```

> [!NOTE]
> You should now be able to follow any of the [other guides](#other-software-made-usable-by-the-patched-sdl) in this document. If you're looking for one of them instead of the Super Mario 64 PC port, skip there. Currently, [mupen64plus](#mupen64plus) offers better in-game performance than sm64ex.

14. Download the Super Mario 64 PC port and place your Super Mario 64 ROM in the repository root directory, replacing `baserom.us.z64` with your ROM's filename (any localization):

> [!NOTE]
> To see how to obtain a Super Mario 64 ROM, click [here](https://github.com/sanni/cartreader).

```bash
git clone https://github.com/sm64pc/sm64ex.git
cp baserom.us.z64 sm64ex
```

15. Patch the Super Mario 64 PC port with minor changes that improve the compilation experience on BBB, then compile it:

```bash
cd sm64ex 
curl https://raw.githubusercontent.com/robertkirkman/sm64ex-bbb-doc/main/sm64ex-bbb.patch | git apply -v
TARGET_BBB=1 BETTERCAMERA=1 EXTERNAL_DATA=1 TEXTURE_FIX=1 make
cd
```

16. Prepare an input device. As an example I use an official Nintendo GameCube controller with the official Nintendo USB GameCube Controller Adapter and ToadKing's open-source user-space driver. Plug the controller into the adapter, Connect the adapter to the BBB's USB A port via the black connector, then download, compile, and run the driver. You only need to do this if you are using a GameCube Controller Adapter with vendor and device ID `057e:0337` as it appears in `lsusb`:

```bash
lsusb | grep '057e:0337'
git clone https://github.com/ToadKing/wii-u-gc-adapter.git
cd wii-u-gc-adapter
export CFLAGS="-O3 -flto -march=armv7-a -marm -mfpu=neon -mfloat-abi=hard -mtune=cortex-a8" 
make
sudo ./wii-u-gc-adapter &
cd
```

17. Connect the display to the BBB using the HDMI cord, then initialize the user-space graphics driver and run the Super Mario 64 PC port:

> [!NOTE]
> To hear the game audio, your display must have speakers or a 3.5mm jack for HDMI audio.

```bash
sudo pvrsrvctl --start --no-module
sm64ex/build/us_pc/sm64.us.f3dex2e
```
> [!NOTE]
> You should now be able to see and play the game. If the performance isn't satisfactory, or you want to use this patched SDL for other purposes, keep reading.

## Other software made usable by the patched SDL:
### mupen64plus
> [!NOTE]
> I don't know why, but with the steps in this guide, and all other things being equal, Super Mario 64 performs better on BBB in `mupen64plus-video-gles2n64` than it does in `sm64ex`, making mupen64plus a good choice as long as there is no need for mods that only work on PC port, not real console - or if there is need for mods that only work on console and emulator.

#### Prerequisites:
* Same as [sm64ex](#prerequisites)

#### Installation:

1. Download, compile and install mupen64plus and its core plugins:
```bash
sudo apt install -y zlib1g-dev \
                    libpng-dev \
                    libfreetype-dev \
                    nasm \
                    libboost-dev \
                    libboost-filesystem-dev
git clone https://github.com/mupen64plus/mupen64plus-core.git
git clone https://github.com/mupen64plus/mupen64plus-ui-console.git
git clone https://github.com/mupen64plus/mupen64plus-audio-sdl.git
git clone https://github.com/mupen64plus/mupen64plus-input-sdl.git
git clone https://github.com/mupen64plus/mupen64plus-rsp-hle.git
export USE_GLES=1 NEON=1 HOST_CPU=armv7 PREFIX=/usr OPTFLAGS="-O3 -flto -march=armv7-a -marm -mfpu=neon -mfloat-abi=hard -mtune=cortex-a8" 
cd mupen64plus-core/projects/unix
make all
sudo make install
cd -
cd mupen64plus-ui-console/projects/unix
make all
sudo make install
cd -
cd mupen64plus-audio-sdl/projects/unix
make all
sudo make install
cd -
cd mupen64plus-input-sdl/projects/unix
make all
sudo make install
cd -
cd mupen64plus-rsp-hle/projects/unix
make all
sudo make install
cd -
```

2. Download, compile and install the `gles2n64` video plugin for mupen64plus, then adjust its system-wide configuration to match the system screen resolution:
```bash
git clone https://github.com/ricrpi/mupen64plus-video-gles2n64.git
cd mupen64plus-video-gles2n64/projects/unix
export USE_GLES=1 NEON=1 HOST_CPU=armv7 PREFIX=/usr OPTFLAGS="-O3 -flto -march=armv7-a -marm -mfpu=neon -mfloat-abi=hard -mtune=cortex-a8" 
make all
sudo make install
sudo sed -i -e 's/window\ width=400/window\ width=640/g' -e 's/window\ height=240/window\ height=480/g' /usr/share/mupen64plus/gles2n64.conf
cd -
```

3. Run mupen64plus, where `baserom.us.z64` is the path to your Super Mario 64 ROM. `pvrsrvctl` only needs to be run once since the last time it was manually stopped or the BBB was rebooted. If you are using a GameCube Controller Adapter with vendor and device ID `057e:0337`, run `wii-u-gc-adapter` from step 16 above:
> [!NOTE]
> For more info about and troubleshooting for the `pvrsrvctl` command, scroll [here](#initializing-the-powervr-driver).
```bash
lsusb
sudo ~/wii-u-gc-adapter/wii-u-gc-adapter &
sudo pvrsrvctl --start --no-module
mupen64plus --fullscreen --gfx mupen64plus-video-n64 baserom.us.z64
```

### [TODO: unfinished] Half-Life 2
#### Prerequisites:
* GNU/Linux PC with an internet connection and a MicroSD card slot or USB A Female port
* BeagleBone Black prepared as directed in steps 1-13 of the above guide
* 10GB+ MicroSD card or USB flash drive depending on the ports available on the PC
* USB A Male to Mini-B Male cord
* Micro-HDMI Male to HDMI Male cord
* Display with an HDMI Female port
* 2+ port powered USB hub
* USB mouse and keyboard

> [!CAUTION]
> Be careful with unpowered USB hubs! The BBB cannot supply much current. Rather than trying to calculate that and push the limit, it is safer to use a powered hub.

#### Installation:
1. Copy a `hl2` folder and optionally `episodic`, `ep2`, or `lostcoast` folders from your Half-Life 2 installation on PC into a new folder named `halflife2` on an empty USB flash drive or SD card formatted with a UNIX-like filesystem (such as `ext4`) with at least 10GB of free space, then unmount the drive, `sync`, remove it from your PC and plug it into the BBB. 
> [!NOTE]
> To see how to obtain a copy of Half-Life 2, click [here](https://store.steampowered.com/app/220/HalfLife_2/).

2. On the BBB, identify and mount the drive, where `/dev/XY` is the block device name of your drive's UNIX-like partition:
```bash
lsblk
sudo mount /dev/XY /mnt
cd /mnt
```

3. Download, patch, compile and install nillerusr's Source Engine into the Half-Life 2 assets folder:
```bash
sudo apt install -y libfreetype6-dev \
                    libfontconfig1-dev \
                    libopenal-dev \
                    libjpeg-dev \
                    libpng-dev \
                    libcurl4-gnutls-dev \
                    libbz2-dev \
                    libedit-dev \
                    python-is-python3
git clone --recursive --depth 1 https://github.com/nillerusr/source-engine.git
cd source-engine
curl https://raw.githubusercontent.com/robertkirkman/sm64ex-bbb-doc/main/source-engine-bbb.patch | git apply -v
./waf configure -T debug --prefix=/mnt/halflife2
./waf install
```

4. Prepare input devices (keyboard and mouse), navigate to the new Half-Life 2 installation, start the PowerVR userspace daemon if it's not already running, and run `hl2_launcher`:
```bash
cd /mnt/halflife2/
sudo pvrsrvctl --start --no-module
./hl2_launcher
```

### Bad Apple!!
>The famous Touhou music video

[![demo](https://i.ytimg.com/vi_webp/Bh3-1u7sITI/maxresdefault.webp)](https://www.youtube.com/watch?v=Bh3-1u7sITI "Bad Apple!! on BeagleBone Black")

#### Prerequisites:
* BeagleBone Black prepared as directed in steps 1-13 of the above guide
* High-speed 10GB+ MicroSD card or USB flash drive
* USB A Male to Mini-B Male cord
* Micro-HDMI Male to HDMI Male cord
* Display with an HDMI Female port

#### Installation:
1. On the BBB, identify and mount the drive, where `/dev/XY` is the block device name of your drive's UNIX-like partition:
> [!NOTE]
> The BBB has no hardware video decoder and a slow CPU, so to make this possible, the video is converted into a raw format that doesn't require decoding to play on a screen. This vastly increases the file size and moves the primary performance bottleneck to drive read speed. Make sure you are using a fast drive whose arbitrary read speed nears or exceeds the ~25MB/s maximum transfer speed of the SD card slot or the ~50MB/s maximum speed of the USB port, otherwise your video might stutter. You can check this by installing `hdparm` with `sudo apt install -y hdparm` and then benchmarking the drive with `sudo hdparm -t /dev/X` where `X` is the block device name of the drive.
```bash
lsblk
sudo mount /dev/XY /mnt
```

2. Download, patch and compile FFmpeg, being careful not to install any packages that conflict with the BBB's graphics driver:
```bash
sudo apt install -y libass-dev \
                    libgnutls28-dev \
                    libmp3lame-dev \
                    libvorbis-dev \
                    libopus-dev \
                    meson \
                    ninja-build \
                    texinfo \
                    yasm \
                    libvpx-dev \
                    python3-pip
cd /mnt
git clone https://github.com/FFmpeg/FFmpeg.git
cd FFmpeg
git checkout aa8905a1b13c6ac206d532afbad953dc71319247
curl https://raw.githubusercontent.com/robertkirkman/sm64ex-bbb-doc/main/ffmpeg-bbb.patch | git apply -v
PKG_CONFIG_PATH=/mnt/FFmpeg/ffmpeg_build/lib/pkgconfig ./configure \
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
make
make install
```

3. Install yt-dlp, download the official Bad Apple!! video, convert it to grayscale raw bitmap video format, and play it using OpenGL (ES 2.0) backend in an SDL window:
> [!NOTE]
> For some reason, without intervention the audio is delayed by almost 1 second, so I manually offset the video to improve the experience. Also, the video is 30 FPS, so if you have a 60 hz screen, it will play at 2x speed, yes really, so you would need to change the settings in the first `ffmpeg` command a bit in order to match the framerate with the refresh rate.
```bash
cd /mnt
python3 -m pip install -U yt-dlp
echo 'export PATH=~/.local/bin:/mnt/FFmpeg:$PATH' >> ~/.bashrc && . ~/.bashrc 
yt-dlp -o badapple_video.webm -f 248 https://www.youtube.com/watch?v=i41KoE0iMYU
yt-dlp -o badapple_audio.webm -f 251 https://www.youtube.com/watch?v=i41KoE0iMYU
ffmpeg -itsoffset 0.83 -i badapple_video.webm -i badapple_audio.webm -vf scale=640:480 -c:v rawvideo -c:a copy -pix_fmt gray -f matroska badapple_480_gray.mkv
sudo pvrsrvctl --start --no-module
ffmpeg -i badapple_480_gray.mkv -f opengl "badapple" -f alsa default
cd
```

### [TODO: unfinished] GZDoom

1.
```bash
wget -O ~/.config/gzdoom/doom.wad https://distro.ibiblio.org/slitaz/sources/packages/d/doom1.wad
sudo apt install -y libgme-dev libmpg123-dev timidity
git clone https://github.com/coelckers/ZMusic.git
cd ZMusic
mkdir build
cd build
export CFLAGS="-O3 -flto -march=armv7-a -marm -mfpu=neon -mfloat-abi=hard -mtune=cortex-a8" 
cmake .. -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=/usr
make
sudo make install
cd
```

2.
```bash
git clone https://github.com/coelckers/gzdoom.git
cd gzdoom
mkdir build
cd build
export CFLAGS="-O3 -flto -march=armv7-a -marm -mfpu=neon -mfloat-abi=hard -mtune=cortex-a8" 
cmake .. -DNO_FMOD=ON -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=/usr
make
sudo make install
```

3.
```bash
sudo pvrsrvctl --start --no-module
gzdoom -gles2_renderer
```

## Other software that can work with SGX530

### [TODO: try to update guide] Qt/Quick EGLFS and Weston Wayland compositor
> [!NOTE]
> The guide at the link below is very outdated and uses much older versions of the graphics driver and other software than I use in this repository, which are incompatible, and if you try to follow the steps there, you will have to start over again before anything here will work again; however, until I create a replacement, it is very good for historical reference and helped me out a lot when I started trying to use the BBB.

Years ago, Remi Avignon's eLinux wiki page was the best SGX530 tutorial, before I created this repository. It includes directions for using Qt/Quick EGLFS and Weston compositor (separately) on BBB, which I have verified do work if you carefully use the exact versions he mentions of all the software involved. If you are interested, you can read it [here](https://elinux.org/BeagleBoneBlack/SGX_%2B_Qt_EGLFS_%2B_Weston).

### [TODO: fill with all working demos] Small OpenGL ES 1.1/2.0 Demos

1. [TODO: rebase] kmscube
```bash
git clone https://github.com/robertkirkman/kmscube.git
cd kmscube
meson setup build
meson compile -C build
cd build
sudo pvrsrvctl --start --no-module
./kmscube
```

### Initializing The PowerVR Driver
Most graphics drivers handle their equivalents to this program robustly and automatically, and if you would like, you may automate this with systemd, but I think it's important to be familiar with this process for troubleshooting and understanding purposes. Here are some common errors that a lot of OpenGL ES 2.0 software will print if you try to execute it before the PowerVR driver is initialized:
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
```bash
sudo dmesg | grep -e PVR -e pvr
```
The first thing you should see is this:
```
pvrsrvkm: loading out-of-tree module taints kernel.
[drm] Initialized pvr 1.17.4948957 20110701 for 56000000.gpu on minor 1
```
If you don't see this or the numbers are different, unfortunately you might have the wrong boot image, device tree, kernel, or kernel module installed. In that case, review steps 1-11 again and make sure you didn't miss anything.

The next thing you should see is this:
```
PVR_K: UM DDK-(4948957) and KM DDK-(4948957) match. [ OK ]
```
If you don't, then `pvrsrvctl` has not been run yet. It needs to be run only once each time the BBB boots. Use this command to run it:
```bash
sudo pvrsrvctl --start --no-module
```
These arguments work the best for me, and if you try to use `pvrsrvctl` a different way, unfortunately it might not work correctly. If you want to experiment or improve on this method, though, feel free to. After doing this, you should now see the "`UM and KM match OK`" message in your kernel log, and can proceed to use the GPU for as long as this BBB stays on. If you don't, you might have the wrong user-space driver installed, so you should carefully review step 11.

## Other things not related to graphics

### Stream audio from BBB to PC using pulseaudio `module_native_protocol_tcp`
> [!WARNING]
> This uses a lot of CPU and is probably not a good idea.

1. BeagleBone Black:
```bash
sudo apt install -y pulseaudio
systemctl --user stop pulseaudio.service
```

2. Server (GNU/Linux system with speakers), where `192.168.6.2` is your BBB's IP address:
```bash
systemctl --user stop pulseaudio.service
ssh -R 4713:localhost:4713 debian@192.168.6.2
```

3. Server (leave the shell above running and launch a new, separate shell):
```bash
systemctl --user start pulseaudio.service
pactl load-module module-native-protocol-tcp auth-anonymous=1
```

4. BeagleBone Black:
```bash
systemctl --user start pulseaudio.service
pactl load-module module-native-protocol-tcp
export PULSE_SERVER=localhost
sm64ex/build/us_pc/sm64.us.f3dex2e
```

### [TODO: update guide] Libreboot
The BeagleBone Black is one of the most open-source hardware affordable computers created before the market availability of RISC-V. For this reason, it is popular as a bootstrap for improving the potential user Freedom of more performant, more proprietary devices using the [Libreboot](https://libreboot.org/) fully open source motherboard firmware project. Click [here](https://libreboot.org/docs/install/spi.html#beaglebone-black-bbb) for their guide to flashing compatible motherboard firmware chips using the BBB's SPI port (may be outdated).

### How to turn off the LEDs

- **on the BBB motherboard**
```bash
sudo bash -c 'for i in $(seq 0 3); do echo 0 | sudo tee "/sys/class/leds/beaglebone:green:usr${i}/brightness"; done'
```
- **on the [4DCAPE-70T display cape](https://4dsystems.com.au/products/4dcape-70t/)**
```bash
echo 0 | sudo tee /sys/class/leds/lcd\:green\:usr0/brightness
```


> [!NOTE]
> ### CREDITS:
> Huge thanks to zmatt on #beagle at irc.libera.chat, vanfanel, Rob Clark, Remi Avignon, Robert Nelson, and all the engineers at Texas Instruments and Imagination Technologies, without whom playing Super Mario 64 on BeagleBone Black would not be possible.
> 
> Thank you to benedani on The Cult Discord guild for the B3313 mod for helping with the mupen64plus setup
> 
> Thank you very much to nillerusr