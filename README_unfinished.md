# unfinished/outdated things

> [!NOTE]
> These are unfinished directions for using several things that worked in the past or that me or someone else **started** porting, but either the ports aren't finished or the directions are so outdated that nothing in there will work on the current images for Debian 11/12 without extensive rewrites.

### [TODO: fix black areas on most of screen] Minetest

![image](https://github.com/user-attachments/assets/ebe41186-8e0c-4a19-b77d-3be31e9b87e1)

#### Prerequisites:

- At least 1 GB [swap partition or swap file](https://wiki.debian.org/Swap)
- USB keyboard
- Monitor screen with compatible mini-HDMI cord or display cape

#### Steps

1. Install dependencies

```bash
sudo apt install meson \
                 libjpeg-dev \
                 libvorbis-dev \
                 libopenal-dev \
                 libsqlite3-dev \
                 libcurl-dev
```

2. Clone repository

```bash
git clone --recursive https://github.com/minetest/minetest.git
cd minetest/
```

3. Enable swap partition (if not already enabled) and compile Minetest:

> [!TIP]
> Swap space is necessary **both to link some of the object files, and to generate Minetest worlds at runtime**, on systems that only have 512 MB of physical RAM like the BBB

```bash
sudo swapon /dev/mmcblk0p2
mkdir build/
cd build/
cmake -G Ninja .. -DENABLE_GLES2=ON \
                  -DENABLE_OPENGL=OFF \
                  -DENABLE_OPENGL3=OFF
ninja
sudo ninja install
```

4. Download a Minetest gamemode:

> [!CAUTION]
> This is **not** the [officially-sanctioned way to install VoxeLibre](https://git.minetest.land/VoxeLibre/VoxeLibre#installation), this is an incomplete development test on BeagleBone Black that currently has an unplayable mostly-black screen as shown above

```bash
mkdir -p ~/.minetest/{games/,worlds/world/}
git clone https://git.minetest.land/VoxeLibre/VoxeLibre.git ~/.minetest/games/VoxeLibre
```

5. Launch a server, then kill it after it has set a valid gameid into the default `world` folder:

```bash
minetest --server --gameid VoxeLibre
sleep 30
kill -9 $(pidof minetest)
```

6. Run Minetest client with the default world

```bash
sudo pvrsrvctl --start --no-module
minetest --go
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
