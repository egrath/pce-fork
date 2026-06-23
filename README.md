# PCE Emulator

This is a fork of Hampa Hug's PCE, a emulator of various systems, including the Macintosh Plus and IBM PC 5150.

## Changes in this fork

### SDL3 driver

The original version supports X11, SDL1 and SDL2. I wanted to have a SDL3 driver for a few reasons:

* SDL2 is no longer under active development
* SDL3 has a nice integrated upscaling shader called 'Pixelart' which looks absolutely stunning

As SDL3 is quite different from SDL1/2 in some aspects and because I wanted to keep the source tree as clean as possible, I did not integrate the driver in the already existing SDL1/2 driver, but created a new one called 'sdl3'.

To use the new driver, you have to modify your configuration file to use 'sdl3' as your terminal driver and 'sdl3' as your sound driver.

For the sound driver modify your sound section:
```
driver="sdl3:wav=speaker.wav:lowpass=0:wavfilter=0"
```

For the video driver modify your terminal section:
```
driver="sdl3"
```

There is also a new option in the terminal section, named 'scale_quality' which controls the algorithm used for scaling the framebuffer to the output window. By default, it's set to 'pixelart' for SDL>=3.4 and 'linear' for SDL<3.4. The other option is nearest.

## Some documentation

Some topics are not so well covered in the documentation (e.g. non existent), so I try to fill the gap as I run over them.

### Disk emulation in Macintosh

First of all, PCE does not use simple sector based images, but instead his own format, PSI (PCE sector image) and PRI (PCE raw image). Think of them as sector images (PSI) and flux images (PRI). I'm using the PSI format as it's easier to handle and I don't have the need for the other.

PCE has two approaches on emulating floppy disk drives:

- Sony drive emulation (high-level emulation)
- IWM controller emulation (very low-level emulation)

The difference between those two is the approach they're using. The Sony drive emulation maps the disk image to the macintosh memory space and uses a extension ROM to redirect access there. This works well and is fast, but I prefer the IWM approach as I simply prefer emulation accuracy. In order to use the IWM emulation, you have to disable the Sony driver in the configuration by commenting out the entire 'sony' and extension rom section.

To insert a disk into a emulated drive, enter the monitor (Escape-M, or whatever you configured your terminal escape key to) and enter:

```
di myimage.psi
g
```

This will eject whatever is in the drive right now, insert the new disk and continue the emulation. The most important question of this section is how to create a PSI from another format (or a empty one). Let's start with an empty one because the process for converting can be easily deducted:

```
dd if=/dev/zero of=empty.dsk bs=1k count=800
hformat -l "Empty" empty.dsk
psi --input empty.dsk --input-format raw --output empty.psi --output-format psi
```

Please note that the `hformat` command comes from the hfsutils package and is highly recommended as it allows for mounting HFS images (as used on floppies) for easy file transfer. There is also a easier way to create a 400/800 KB Macintosh floppy image, but you need to format it after inserting it:

```
psi --new mac 800 --output empty.psi
```

If you need to convert a PSI back to a sector image (e.g. for use in another emulator) use the following command:

```
psi --input mydisk.psi --input-format psi --output mydisk.img --output-format raw
```

### Serial communication

PCE emulates the SCC for serial communication. In order to use it, you have to 'connect' something to it. What you connect is up to you, it can be a physical serial port available on the host, or as shown below a pair of virtual serial ports to allow communication between the host and the emulated machine.

To create such a pair of virtual connected serial ports, we use the tool 'socat':

```
socat -d -d pty,raw,echo=0,link=endpoint1 pty,raw,echo=0,link=endpoint2
```

This creates two new tty's and symlinks them to the file 'endpoint1' and 'endpoint2'. Whatever data is written to one comes out on the other and vica-versa. We use the symlink approach to make our configuration more robust, as we never know which tty device is being created by 'socat'. You have to leave this command running in the background for the ports to persist.

To use these ports, modify the serial section of your configuration file to include a line like:

```
driver = "tios:file=endpoint1"
```

Connect something else to the other side of the pair, like a Terminal emulation or modem emulator software.

## How to build it

```
autoreconf -vis
./configure --disable-tun --disable-char-posix --disable-char-ppp --disable-char-pty --disable-char-slip --disable-char-tcp --disable-char-termios --disable-char-wincom --disable-readline --disable-atari-st --disable-cpm80 --disable-ibmpc --disable-ibmpc-rom --disable-rc759 --disable-sim405 --disable-sims32 --disable-simarm --disable-vic20 --disable-spectrum --disable-sound-oss --without-x --with-sdl=3
make -j $(nproc)
```

