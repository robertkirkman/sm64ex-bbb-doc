# unfinished/outdated things

> [!NOTE]
> These are unfinished directions for using several things that worked in the past or that me or someone else **started** porting, but either the ports aren't finished or the directions are so outdated that nothing in there will work on the current images for Debian 11/12 without extensive rewrites.

### [TODO: regression - bisect] mupen64plus
> [!NOTE]
> I don't know why, but with the steps in this guide, and all other things being equal, Super Mario 64 performs better on BBB in `mupen64plus-video-gles2n64` than it does in `sm64ex`, making mupen64plus a good choice as long as you don't need mods that only work on PC port, not real console - or if you need mods that only work on console and emulator.

#### Prerequisites:
* Same as [sm64ex](#prerequisites)

#### Installation:

1. Install dependencies:
```bash
sudo apt install -y zlib1g-dev \
                    libpng-dev \
                    libfreetype-dev \
                    nasm \
                    libboost-dev \
                    libboost-filesystem-dev
```

2. Set build flags:

> [!IMPORTANT]
> Make sure to set these environment variables in every shell where you compile mupen64plus components.

```bash
export VULKAN=0 \
       USE_GLES=1 \
       NEON=1 \
       HOST_CPU=armv7 \
       PREFIX=/usr \
       OPTFLAGS="-O3 -flto -march=armv7-a -marm -mfpu=neon -mfloat-abi=hard -mtune=cortex-a8"
```

3. mupen64plus-core

```
git clone https://github.com/mupen64plus/mupen64plus-core.git
cd mupen64plus-core/projects/unix/
make all
sudo -E make install
cd -
```

4. mupen64plus-ui-console

```
git clone https://github.com/mupen64plus/mupen64plus-ui-console.git
cd mupen64plus-ui-console/projects/unix/
make all
sudo -E make install
cd -
```

5. mupen64plus-audio-sdl

```bash
git clone https://github.com/mupen64plus/mupen64plus-audio-sdl.git
cd mupen64plus-audio-sdl/projects/unix/
make all
sudo -E make install
cd -
```

6. mupen64plus-input-sdl

```bash
git clone https://github.com/mupen64plus/mupen64plus-input-sdl.git
cd mupen64plus-input-sdl/projects/unix/
make all
sudo -E make install
cd -
```

7. mupen64plus-rsp-hle

```
git clone https://github.com/mupen64plus/mupen64plus-rsp-hle.git
cd mupen64plus-rsp-hle/projects/unix/
make all
sudo -E make install
cd -
```

8. mupen64plus-video-gles2n64

```bash
git clone https://github.com/ricrpi/mupen64plus-video-gles2n64.git
cd mupen64plus-video-gles2n64/projects/unix/
make all
sudo -E make install
cd -
```

> [!TIP]
> Optionally, you might want to change the default resolution for the `mupen64plus-video-gles2n64` plugin to your screen resolution or a different resolution:

```bash
sudo sed -i -e 's/window\ width=400/window\ width=640/g' -e 's/window\ height=240/window\ height=480/g' /usr/share/mupen64plus/gles2n64.conf
```


9. Run mupen64plus, where `baserom.us.z64` is the path to your Super Mario 64 ROM. `pvrsrvctl` only needs to be run once since the last time it was manually stopped or the BBB was rebooted. If you are using a GameCube Controller Adapter with vendor and device ID `057e:0337`, run `wii-u-gc-adapter` from step 16 above:
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

### [TODO: try to update guide] Qt/Quick EGLFS and Weston Wayland compositor
> [!NOTE]
> The guide at the link below is very outdated and uses much older versions of the graphics driver and other software than I use in this repository, which are incompatible, and if you try to follow the steps there, you will have to start over again before anything here will work again; however, until I create a replacement, it is very good for historical reference and helped me out a lot when I started trying to use the BBB.

Years ago, Remi Avignon's eLinux wiki page was the best SGX530 tutorial, before I created this repository. It includes directions for using Qt/Quick EGLFS and Weston compositor (separately) on BBB, which I have verified do work if you carefully use the exact versions he mentions of all the software involved. If you are interested, you can read it [here](https://elinux.org/BeagleBoneBlack/SGX_%2B_Qt_EGLFS_%2B_Weston).


### [TODO: update guide] Libreboot
The BeagleBone Black is one of the most open-source hardware affordable computers created before the market availability of RISC-V. For this reason, it is popular as a bootstrap for improving the potential user Freedom of more performant, more proprietary devices using the [Libreboot](https://libreboot.org/) fully open source motherboard firmware project. Click [here](https://libreboot.org/docs/install/spi.html#beaglebone-black-bbb) for their guide to flashing compatible motherboard firmware chips using the BBB's SPI port (may be outdated).