#[Original Readme](https://github.com/VentanaIoT/vive-diy-position-sensor/blob/master/docs/readme.md)

# DIY Position Tracking using HTC Vive's Lighthouse


## How it works
Lighthouse position tracking system consists of:  
&nbsp;&nbsp;– two stationary infrared-emitting base stations (we'll use existing HTC Vive setup),  
&nbsp;&nbsp;– IR receiving sensor and processing module (this is what we'll create).  

The base stations are usually placed high in the room corners and "overlook" the room.
Each station has an IR LED array and two rotating laser planes, horizontal and vertical.
Each cycle, after LED array flash (sync pulse), laser planes sweep the room horizontally/vertically with constant rotation speed.
This means that the time between the sync pulse and the laser plane "touching" sensor is proportional to horizontal/vertical angle
from base station's center direction.
Using this timing information, we can calculate 3d lines from each base station to sensor, the crossing of which yields
3d coordinates of our sensor (see [calculation details](../../wiki/Position-calculation-in-detail)).
Great thing about this approach is that it doesn't depend on light intensity and can be made very precise with cheap hardware.

Visualization of one base station (by rvdm88, click for full video):  
[![How it works](http://i.giphy.com/ijMzXRF3OYBZ6.gif)](https://www.youtube.com/watch?v=oqPaaMR4kY4)

See also:  
[This Is How Valve’s Amazing Lighthouse Tracking Technology Works – Gizmodo](http://gizmodo.com/this-is-how-valve-s-amazing-lighthouse-tracking-technol-1705356768)  
[Lighthouse tracking examined – Oliver Kreylos' blog](http://doc-ok.org/?p=1478)  
[Reddit thread on Lighthouse](https://www.reddit.com/r/Vive/comments/40877n/vive_lighthouse_explained/)  

The sensor we're building is the receiving side of the Lighthouse. It will receive, recognize the IR pulses, calculate
the angles and produce 3d coordinates.

## How it works – details
Base stations are synchronized and work in tandem (they see each other's pulses). Each cycle only one laser plane sweeps the room,
so we fully update 3d position every 4 cycles (2 stations * horizontal/vertical sweep). Cycles are 8.333ms long, which is
exactly 120Hz. Laser plane rotation speed is exactly 180deg per cycle.

Each cycle, as received by sensor, has the following pulse structure:

| Pulse start, µs | Pulse length, µs | Source station | Meaning |
| --------: | ---------: | -------------: | :------ |
|         0 |     65–135 |              A | Sync pulse (LED array, omnidirectional) |
|       400 |     65-135 |              B | Sync pulse (LED array, omnidirectional) |
| 1222–6777 |        ~10 |         A or B | Laser plane sweep pulse (center=4000µs) |
|      8333 |            |                | End of cycle |

You can see all three pulses in the IR photodiode output (click for video):
[![Lighthouse pulse structure](https://cloud.githubusercontent.com/assets/627997/20243190/0e54fc44-a902-11e6-90cd-a4edf2464e7e.png)](https://youtu.be/7OFeN3gl3SQ)

The sync pulse lengths encode which of the 4 cycles we're receiving and station id/calibration data
(see [description](https://github.com/nairol/LighthouseRedox/blob/master/docs/Light%20Emissions.md)).


### Sensor board

IR photodiodes produce very small current, so we need to amplify it before feeding to a processing module. I use 
[TLV2462IP](https://store.ti.com/TLV2462IP.aspx) opamp – a modern, general purpose
rail-to-rail opamp with good bandwidth, plus there are 2 of them in a chip, which is convenient.

One more thing we need to add is a simple high-pass filter to filter out background illumination level changes.

Full schematics:  
![schematics](https://ashtuchkin.github.io/vive-diy-position-sensor/sensor-schematics.svg)

| Top view | Bottom view |
| --- | --- |

Part list (add to cart [from here](1-click-bom.tsv) using [1-click BOM](https://1clickbom.com)):

| Part | Model | Count | Cost (digikey) |
| --- | --- | --- | ---: |
| D1, D2, D3 | BPV22NF | 3 | 3x[$1.11](https://www.digikey.com/product-detail/en/vishay-semiconductor-opto-division/BPV22NF/751-1007-ND/1681141) |
| U1, U2 | TLV2462IP | 1 | [$2.80](https://www.digikey.com/product-detail/en/texas-instruments/TLV2462IP/296-1893-5-ND/277538) |
| Board | Perma-proto | 1 | [$2.95](https://www.digikey.com/product-detail/en/adafruit-industries-llc/1608/1528-1101-ND/5154676) |
| C1 | 5pF | 1 | [$0.28](https://www.digikey.com/product-detail/en/tdk-corporation/FG28C0G1H050CNT06/445-173467-1-ND/5812072) |
| C2 | 10nF | 1 | [$0.40](https://www.digikey.com/product-detail/en/tdk-corporation/FK24C0G1H103J/445-4750-ND/2050099) |
| R1 | 100k | 1 | [$0.10](https://www.digikey.com/product-detail/en/stackpole-electronics-inc/CF14JT100K/CF14JT100KCT-ND/1830399) |
| R2, R4 | 47k | 2 | 2x[$0.10](https://www.digikey.com/product-detail/en/stackpole-electronics-inc/CF14JT47K0/CF14JT47K0CT-ND/1830391) |
| R3 | 3k | 1 | [$0.10](https://www.digikey.com/product-detail/en/stackpole-electronics-inc/CF12JT3K00/CF12JT3K00CT-ND/1830498) |
| Total |  |  | $10.16 | 

Sample oscilloscope videos:

| Point | Video |
| --- | --- |

### Teensy connection

| Teensy connections |  Full position tracker |
| --- | --- |
| ![image](https://cloud.githubusercontent.com/assets/627997/20243742/33e43b52-a919-11e6-9069-4cedc70f1c77.png) | ![image](https://cloud.githubusercontent.com/assets/627997/20243775/bca50ccc-a91a-11e6-8b45-33e086c21b3d.png) |



## Software (Teensy)

We use hardware comparator interrupt with ISR being called on both rise and fall edges of the signal. ISR (`cmp0_isr`) gets the timing 
in microseconds and processes the pulses depending on their lengths. We track the sync pulses lengths to determine which 
cycle corresponds to which base station and sweep. After the tracking is established, we convert time delays to angles and
calculate the 3d lines and 3d position (see geometry.cpp). After position is determined, we report it as text to USB console and
as Mavlink ATT_POS_MOCAP message to UART port (see mavlink.cpp).
 
NOTE: Currently, base station positions and direction matrices are hardcoded in geometry.cpp (`lightsources`). You'll need to 
adjust it for your setup. See [#2](//github.com/ashtuchkin/vive-diy-position-sensor/issues/2).

### Installation on macOS, Linux

Prerequisites:
 * [GNU ARM Embedded toolchain](https://launchpad.net/gcc-arm-embedded). Can be installed on Mac with `brew cask install gcc-arm-embedded`.
   I'm developing with version 5_4-2016q3, but other versions should work too.
 * CMake 3.5+ `brew install cmake`
 * Command line uploader/monitor: [ty](https://github.com/Koromix/ty). See build instructions in the repo.
 * I recommend CLion as the IDE - it made my life a lot easier and can compile/upload right there.

Getting the code:
```bash
$ git clone https://github.com/ashtuchkin/vive-diy-position-sensor.git
$ cd vive-diy-position-sensor
$ git submodule update --init
```

Compilation/upload command line (example, using CMake out-of-source build in build/ directory):
```bash
$ cd build
$ cmake ..
$ make  # Build firmware
$ make vive-diy-position-sensor_Upload  # Upload to Teensy
$ tyc monitor  # Serial console to Teensy
```

### Installation on Windows

I haven't been able to make it work in Visual Studio, so providing command line build solution.

Prerequisites:
 * [GNU ARM Embedded toolchain](https://developer.arm.com/open-source/gnu-toolchain/gnu-rm). Windows 32 bit installer is fine.
 * [CMake 3.5+](https://cmake.org/download/). Make sure it's in your %PATH%.
 * [Ninja build tool](https://ninja-build.org/). Copy binary in any location in your %PATH%.

Getting the code is the same as above. GitHub client for Windows will make it even easier.

Building firmware:
```
cd build
cmake -G Ninja ..
ninja  # Build firmware. Will generate "vive-diy-position-sensor.hex" in current directory.
```
