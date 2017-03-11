# EC500C1 -- Positional Tracking Project


## Objectives


The objective of the project is to provide a mechanism for tracking physical objects in a room and relaying the spacial coordinates to the Microsoft HoloLens for tracking within a mixed reality application.


## Setup


We aim to achieve our goal by basing this project on the [https://github.com/ashtuchkin/vive-diy-position-sensor](Vive DIY Position Sensor Project) repository on GitHub. In this open sourced project, the HTC Vive base stations are used for positional tracking combined with a DIY tracker circuit. This circuit interprets the IR signals emitted by the base stations and converts it into a (X,Y,Z,Distance) coordinate relative to the tracker’s location between the base stations.


### Hardware


We first needed to build our own tracker based on the specifications setup by the GitHub project. This involved the following parts:


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
| R5 | 1.9k | 1  | -- |
| Teensy | Model 1.2 | 1 | -- |


These parts will create one tracker. In addition to the DIY components, two HTC Vive Base Stations (Lighthouses) are required for the project.


As we built the circuit, we keep track of the different signals produced:


![Bad Signal](https://github.com/VentanaIoT/vive-diy-position-sensor/blob/master/files/Bad_Signal.jpg?raw=true "Bad Signal")


![Noisy Signal Signal](https://github.com/VentanaIoT/vive-diy-position-sensor/blob/master/files/Noisy_Signal.jpg?raw=true "Noisy Signal")


![Good Signal Signal](https://github.com/VentanaIoT/vive-diy-position-sensor/blob/master/files/Good_Signal.jpg?raw=true "Good Signal")


We have made minor modifications to the circuit to improve reliability in the signal readings and compatibility with the HoloLens. Further details will be described below.


### Software


The Teensy is an Arduino compatible microprocessor that uses C++ as its programming language. In order to remain compatible with the specifications of the Vive Tracker project, we choose to use the same model of the Teensy as well. Compilation of the code requires the ARM Embedded Toolchain, CMake >v3.5, TY serial monitor (equivalent software to Arduino IDE).


## Progress


### Theory
Within the GitHub project that we are basing our project on, they use an actual HTC Vive to calibrate their setup. We are looking to build a completely self sufficient system, so we need a new process for calibration.


### Hardware


Our testing determined that the HoloLens interfered with tracker’s ability to ‘see’ the base station signals due to the IR blasts emitted by the HoloLens for its own version of positional tracking. In order to achieve HoloLens Compatibility, we utilized the Oscilloscope to determine the modulation and carrier frequency of the IR blasts emitted by the HoloLens. It turned out that we could implement a low-pass filter on the output to remove the HoloLens IR blasts and get a clean, Base station and HoloLens friendly circuit. 


![HoloLens Interference Signal](https://github.com/VentanaIoT/vive-diy-position-sensor/blob/master/files/HoloLens%20Noise.jpg?raw=true "HoloLens Interference Signal")


### Measurements
While we don’t have usable rotation matrices yet, we decided to take some values of our setup to make sure they were consistent. They can all be found [https://github.com/VentanaIoT/vive-diy-position-sensor/tree/master/files/Measurements%20and%20plots](here)
## Next Steps


### Developing calibration process
There are two main parts of the calibration process. The first is the translation values for each of the lighthouses. Essentially just setting up our coordinate system and placing the lighthouses in it. We plan to do this in one of two ways. 


The first is to expand our current hardware module to have multiple sensors instead of just one and then use the difference in timing data to detect how far away and in what direction the individual lighthouses are.


The second possibility is to layer virtual lighthouses over the physical ones and then use the HoloLens spatial map to do the calculation of where they are in relation to each other. This is the harder, but also most favorable possibility as this technique will also allow us to do other pieces of the calibration process in the same step and layer the coordinate systems over each other.


This brings us to the rotational matrix, the second part of the calibration process. This is the place where we will be doing a lot of work because other projects assume the user has vive to do this calibration. Essentially the rotational matrix indicates how the lighthouses are rotated in space. More details can be found [https://github.com/VentanaIoT/vive-diy-position-sensor/blob/master/files/Rotational%20Matrix.pdf](Here)


## Challenges
### Layering coordinate systems
Along with calibration, we need a method for overlapping the Hololens coordinate system with the tracker’s coordinate system. This is integral to our project because without it, we cannot match location in one coordinate system with another.


We found a project that does a similar type of overlay [https://github.com/dag10/HoloViveObserver](HoloViveObserver) and uses the vive controller to set a common point for both the Vive and the Hololens. We are looking to do the same thing but instead of using the controller to set our common point we are going to use place virtual models of the HTC Vive Lighthouses on top of the real ones to set our common points that we can then translate into the HoloLens coordinate system


### Adding wireless to teensy
The position calculation that our teensy does needs to be transferred wirelessly so that we can read the data on the HoloLens and place holograms accordingly. We have a raspberry pi based server that acts as our middleman, so we will be transferring all of the position data there.


We can add different wireless technologies to our Teensy to achieve this including Bluetooth, Wi-Fi, or one of the IoT specific standards like ZigBee or Z-Wave