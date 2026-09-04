# ESP32-S3 + DWM3000 UWB Drone Localization System

Real-time indoor and GPS-denied drone localization using **ESP32-S3**, **DWM3000 Ultra-Wideband (UWB)** ranging, and **ESP-NOW 2.4 GHz peer-to-peer communication**. The system uses two fixed UWB anchors and one mobile tag mounted on a drone. The DWM3000 devices calculate the distances between the drone and each anchor, while ESP-NOW provides a low-latency wireless network for exchanging anchor IDs, tag IDs, distance measurements, synchronization data, system status, and estimated position.

![](https://github.com/1Px-Vision/UWB-Drone-Nav/blob/main/DWM3000%20Multi-Anchor.jpg)
---

## Overview

This project implements a distributed localization architecture composed of:

* **Anchor 1**

  * ESP32-S3
  * DWM3000 UWB module
  * Known position `(x1, y1, z1)`

* **Anchor 2**

  * ESP32-S3
  * DWM3000 UWB module
  * Known position `(x2, y2, z2)`

* **Drone Tag**

  * ESP32-S3
  * DWM3000 UWB module
  * Optional IMU
  * Moving target

* **Python Dashboard**

  * Receives ranging and localization data
  * Displays anchor positions
  * Displays drone trajectory
  * Displays real-time distances
  * Displays localization error and system status

The system is intended for:

* GPS-denied drone navigation
* Indoor UAV localization
* Search-and-rescue applications
* Robotics
* Autonomous navigation
* UWB ranging experiments
* Embedded localization research

---

# System Architecture

```text
                       MOVING DRONE
              ┌────────────────────────┐
              │      ESP32-S3 TAG      │
              │        TAG_ID=01       │
              │                        │
              │ DWM3000 UWB            │
              │ IMU                    │
              │ UKF Position Filter    │
              └───────────┬────────────┘
                         / \
                        /   \
                UWB d1 /     \ UWB d2
                      /       \
                     ▼         ▼

          ┌────────────────┐  ┌────────────────┐
          │    ANCHOR 1    │  │    ANCHOR 2    │
          │    ESP32-S3    │  │    ESP32-S3    │
          │    DWM3000     │  │    DWM3000     │
          │    ID = A1     │  │    ID = A2     │
          │    (x1,y1)     │  │    (x2,y2)     │
          └────────────────┘  └────────────────┘

             <------ ESP-NOW 2.4 GHz ------>

               IDs / Range / Status
              Timestamp / Sync / Position

                         │
                         ▼
                Python Dashboard
```

---

![](https://github.com/1Px-Vision/UWB-Drone-Nav/blob/main/Anchor_Target.jpg)

# Communication Architecture

The project uses two different wireless technologies.

## UWB

The **DWM3000** modules are responsible for precise distance measurement.

Typical ranging sequence:

```text
TAG <------ UWB TWR ------> ANCHOR 1

Distance:

d1 = distance(TAG, Anchor 1)
```

and

```text
TAG <------ UWB TWR ------> ANCHOR 2

Distance:

d2 = distance(TAG, Anchor 2)
```

The system may use:

* Single-Sided Two-Way Ranging
* Double-Sided Two-Way Ranging
* TWR
* Future TDoA implementation

---

## ESP-NOW

ESP-NOW provides the peer-to-peer 2.4 GHz communication layer.

```text
                 TAG
               /     \
              /       \
             ▼         ▼
         ANCHOR 1 <-> ANCHOR 2
```

ESP-NOW is used for:

* Anchor identification
* Tag identification
* Distance transfer
* Measurement synchronization
* Sequence numbers
* Timestamps
* Link status
* Quality information
* Position results
* System control messages

No external Wi-Fi router is required.

> Note: ESP-NOW is used instead of traditional IEEE 802.11 IBSS ad-hoc mode. ESP32-S3 does not provide standard IBSS mode through the normal Espressif Wi-Fi driver.

---

# Hardware

## Required Hardware

| Quantity | Device             | Function                   |
| -------: | ------------------ | -------------------------- |
|        3 | ESP32-S3           | Main embedded controller   |
|        3 | DWM3000            | UWB ranging                |
|        1 | IMU                | Drone motion estimation    |
|        1 | Drone/UAV platform | Moving target              |
|        2 | USB cables         | Anchor/debug interface     |
|        1 | PC                 | Dashboard and data logging |

Optional hardware:

* External 2.4 GHz antenna
* UWB external antenna
* SD card
* GPS for outdoor validation
* Barometer
* Magnetometer
* Optical-flow sensor
* Flight controller

---

# Recommended ESP32

The recommended controller is:

```text
ESP32-S3
```

Preferred module:

```text
ESP32-S3-WROOM-1
```

or

```text
ESP32-S3-WROOM-1U
```

The `WROOM-1U` version can be useful when an external 2.4 GHz antenna is required.

Advantages of the ESP32-S3 include:

* Dual-core processor
* 2.4 GHz IEEE 802.11 b/g/n
* ESP-NOW support
* SPI
* I2C
* UART
* USB
* FreeRTOS
* Sufficient processing capability for UWB ranging and filtering
* PSRAM options

---

# DWM3000 Connection

The ESP32-S3 communicates with the DWM3000 through SPI.

Example connection:

```text
ESP32-S3              DWM3000
--------------------------------
3.3 V       --------> VCC
GND         --------> GND
SCK         --------> CLK
MOSI        --------> MOSI
MISO        <-------- MISO
CS          --------> CS
IRQ         <-------- IRQ
RST         --------> RESET
```

The exact GPIO configuration should be defined in the firmware.

Example:

```cpp
#define PIN_SPI_SCK    12
#define PIN_SPI_MISO   13
#define PIN_SPI_MOSI   11

#define PIN_DWM_CS     10
#define PIN_DWM_IRQ     9
#define PIN_DWM_RST     8
```

Modify these values according to the selected ESP32-S3 board.

---

# Node Configuration

Each device has a unique role and ID.

## Anchor 1

```cpp
NODE_TYPE = ANCHOR
NODE_ID   = 1

ANCHOR_X = 0.0
ANCHOR_Y = 0.0
ANCHOR_Z = 1.5
```

## Anchor 2

```cpp
NODE_TYPE = ANCHOR
NODE_ID   = 2

ANCHOR_X = 8.0
ANCHOR_Y = 0.0
ANCHOR_Z = 1.5
```

## Drone Tag

```cpp
NODE_TYPE = TAG
NODE_ID   = 1
```

---

# Measurement Cycle

A synchronized measurement cycle is recommended for a moving drone.

```text
FRAME N

  |
  +-- TAG sends FRAME_START
  |
  +-- TAG <-> Anchor 1
  |       UWB ranging
  |
  |       d1 = 3.42 m
  |
  +-- TAG <-> Anchor 2
  |       UWB ranging
  |
  |       d2 = 5.18 m
  |
  +-- Read IMU
  |
  +-- Collect ESP-NOW measurements
  |
  +-- Localization algorithm
  |
  +-- UKF update
  |
  +-- Position:
          x
          y
          vx
          vy
  |
  +-- Send position to dashboard
```

The process repeats continuously:

```text
FRAME 100
FRAME 101
FRAME 102
FRAME 103
...
```

---

# Example Distance Measurements

As the drone moves:

```text
Time     Anchor 1     Anchor 2
--------------------------------
t0       3.42 m       5.18 m
t1       3.51 m       5.07 m
t2       3.64 m       4.92 m
t3       3.79 m       4.74 m
t4       3.95 m       4.56 m
```

---

# ESP-NOW Packet

Example packet structure:

```cpp
typedef struct
{
    uint8_t message_type;

    uint8_t tag_id;
    uint8_t anchor_id;

    uint32_t sequence;

    float distance_m;

    float anchor_x;
    float anchor_y;
    float anchor_z;

    uint32_t timestamp_ms;

    float rx_power_dbm;

    uint8_t quality;

} uwb_packet_t;
```

Possible message types:

```cpp
#define MSG_FRAME_START      0x01
#define MSG_RANGE_RESULT     0x02
#define MSG_POSITION         0x03
#define MSG_STATUS           0x04
#define MSG_SYNC             0x05
```

---

# Localization

With two anchors:

```text
Anchor 1                               Anchor 2
   ●--------------------------------------●
    \                                    /
     \                                  /
      \ d1                           d2 /
       \                              /
        \                            /
                 ● Drone
                  (x,y)
```

The range equations are:

```text
d1² = (x-x1)² + (y-y1)²

d2² = (x-x2)² + (y-y2)²
```

Two anchors can produce two possible geometrical intersections.

```text
                    P1
                     ●
                    / \
                   /   \
                  /     \
       Anchor 1 ●---------● Anchor 2
                  \     /
                   \   /
                    \ /
                     ●
                    P2
```

For this reason, the recommended drone configuration is:

```text
2 UWB ranges
     +
IMU
     +
previous position
     +
UKF
```

This allows the estimator to select the physically valid drone position and smooth noisy measurements.

For more robust localization, the system can later be extended to:

```text
3 Anchors -> 2D trilateration

4+ Anchors -> 3D localization and redundancy
```

---

# UKF Sensor Fusion

The Unscented Kalman Filter can combine:

```text
UWB Anchor 1 distance
          +
UWB Anchor 2 distance
          +
IMU acceleration
          +
IMU angular velocity
          +
previous drone position
```

to estimate:

```text
x
y
z

vx
vy
vz

yaw
```

A simplified state vector can be:

```text
X = [x, y, vx, vy]
```

or, for 3D navigation:

```text
X = [x, y, z, vx, vy, vz]
```

---

# Software Architecture

## Drone Tag Firmware

```text
ESP32-S3 TAG
│
├── DWM3000 Task
│   ├── Range Anchor 1
│   ├── Range Anchor 2
│   └── Range quality
│
├── ESP-NOW Task
│   ├── Discover peers
│   ├── Send measurements
│   ├── Receive anchor data
│   └── Synchronization
│
├── IMU Task
│
├── Localization Task
│   ├── Two-anchor geometry
│   └── UKF
│
└── Telemetry Task
    └── Position/status
```

---

## Anchor Firmware

```text
ESP32-S3 ANCHOR
│
├── DWM3000
│   └── UWB ranging
│
├── ESP-NOW
│   ├── Receive Tag request
│   └── Send range result
│
├── Node manager
│
└── USB Serial
    └── Python dashboard
```

---

# Python Dashboard

The PC application can receive data through USB serial from one anchor.

Example:

```text
Anchor 1
   |
   | USB
   |
   v
Python Dashboard
```

The anchor acts as a gateway for the localization network.

Example serial packet:

```json
{
    "tag": 1,
    "sequence": 125,
    "d1": 3.42,
    "d2": 5.18,
    "x": 2.84,
    "y": 1.91,
    "quality": 92
}
```

---

# Dashboard Features

The dashboard can provide:

* Serial-port automatic detection
* ESP32 connection status
* Anchor configuration
* Anchor coordinates
* Tag identification
* Distance Anchor 1 → Tag
* Distance Anchor 2 → Tag
* Real-time `(x,y)` location
* UWB measurement quality
* Packet loss
* ESP-NOW status
* Drone trajectory
* Raw position
* UKF-filtered position
* Velocity
* Data recording
* CSV export

Example mapping view:

```text
Y
^

8 m  |
     |
     |                    Drone
     |                      ●
     |                   .'
     |                .'
     |             .'
     |
0 m  ●--------------------------------●
     A1                              A2

     0 m                              8 m ---> X
```

---

# Suggested Repository Structure

```text
esp32-s3-uwb-drone-localization/
│
├── README.md
│
├── LICENSE
│
├── firmware/
│   │
│   ├── tag/
│   │   └── main_tag.ino
│   │
│   ├── anchor/
│   │   └── main_anchor.ino
│   │
│   └── common/
│       ├── uwb_protocol.h
│       ├── espnow_protocol.h
│       └── node_config.h
│
├── dashboard/
│   ├── uwb_dashboard.py
│   ├── serial_manager.py
│   ├── localization.py
│   └── ukf.py
│
├── config/
│   └── anchors.json
│
├── data/
│   └── example_measurements.csv
│
├── docs/
│   ├── architecture.md
│   ├── protocol.md
│   └── hardware.md
│
└── images/
    └── system_architecture.png
```

---

# Example Anchor Configuration

`config/anchors.json`

```json
{
    "anchors": [
        {
            "id": 1,
            "x": 0.0,
            "y": 0.0,
            "z": 1.5
        },
        {
            "id": 2,
            "x": 8.0,
            "y": 0.0,
            "z": 1.5
        }
    ]
}
```

---

# Operating Sequence

## 1. Start Anchor 1

```text
ANCHOR_ID = 1
```

The device initializes:

* ESP-NOW
* DWM3000
* SPI
* USB serial

---

## 2. Start Anchor 2

```text
ANCHOR_ID = 2
```

Anchor 2 joins the ESP-NOW peer network.

---

## 3. Start Drone Tag

The tag discovers the anchor nodes and starts UWB ranging.

```text
TAG -> A1
TAG -> A2
```

---

## 4. Start Dashboard

Example:

```bash
python uwb_dashboard.py
```

The dashboard detects the USB serial interface.

Example Linux ports:

```text
/dev/ttyUSB0
/dev/ttyACM0
```

Example Windows ports:

```text
COM3
COM4
COM5
```

---

# Real-Time Data Flow

```text
DWM3000 Anchor 1
        |
        | d1
        v
      ESP32
        |
        | ESP-NOW
        |
        v
      TAG
        ^
        |
        | ESP-NOW
        |
      ESP32
        ^
        | d2
        |
DWM3000 Anchor 2
```

Then:

```text
d1
 +
d2
 +
IMU
 |
 v
UKF
 |
 v
(x,y)
 |
 v
ESP-NOW / USB
 |
 v
Dashboard
```

---

# Timing

A target update rate may initially be configured around:

```text
10 Hz
```

which corresponds to:

```text
100 ms / localization cycle
```

An example cycle:

```text
0 ms      FRAME_START

5 ms      Anchor 1 ranging

25 ms     Anchor 1 result

35 ms     Anchor 2 ranging

55 ms     Anchor 2 result

65 ms     IMU update

70 ms     UKF

80 ms     position broadcast

100 ms    next cycle
```

Actual timing depends on:

* UWB ranging configuration
* SPI speed
* ESP-NOW traffic
* retry configuration
* filtering
* number of anchors

---

# RF Architecture

The system uses two independent radio technologies:

```text
2.4 GHz
ESP-NOW
-------------------------------
Control
Telemetry
Anchor IDs
Tag IDs
Synchronization
Distance packets
Position packets
```

and:

```text
UWB
DWM3000
-------------------------------
Time-of-flight ranging
Distance estimation
```

The ESP-NOW RSSI should not be used as the primary precision ranging mechanism.

---

# Antenna Considerations

For drone installation, keep the:

```text
2.4 GHz ESP32 antenna
```

and

```text
DWM3000 UWB antenna
```

physically separated when possible.

Avoid positioning antennas close to:

* ESCs
* motors
* switching converters
* high-current battery cables
* carbon-fiber structures
* large metallic objects

Antenna orientation should also be considered during localization experiments.

---

# Safety and Reliability

For autonomous drone operation, the UWB localization system should not initially be used as the only safety-critical navigation source.

Recommended additional sensors include:

* IMU
* optical flow
* barometer
* LiDAR
* visual odometry
* SLAM
* obstacle sensing

The system should provide fail-safe modes for:

* packet loss
* invalid UWB distance
* lost anchor
* filter divergence
* communication failure
* position uncertainty

---

# Future Development

Planned extensions can include:

* [ ] Three-anchor localization
* [ ] Four-anchor 3D localization
* [ ] DS-TWR
* [ ] TDoA
* [ ] Automatic ESP-NOW peer discovery
* [ ] Anchor auto-detection
* [ ] USB serial auto-detection
* [ ] UWB quality estimation
* [ ] LOS/NLOS detection
* [ ] UKF sensor fusion
* [ ] EKF comparison
* [ ] Particle filter comparison
* [ ] IMU integration
* [ ] Optical-flow integration
* [ ] Drone flight-controller integration
* [ ] PX4 integration
* [ ] MAVLink integration
* [ ] ROS 2 integration
* [ ] Real-time Plotly dashboard
* [ ] Position heatmap
* [ ] RSSI visualization
* [ ] Packet-loss monitoring
* [ ] CSV recording
* [ ] Flight-path recording
* [ ] Multiple tags
* [ ] Swarm drone localization

---

# Potential Multi-Anchor Architecture

Future configuration:

```text
                    Anchor 3
                       ●
                      / \
                     /   \
                    /     \
                   / Drone \
                  /    ●    \
                 /           \
                /             \
               ●---------------●
           Anchor 1         Anchor 2
```

For 3D:

```text
Anchor 1
Anchor 2
Anchor 3
Anchor 4
    |
    v
UWB ranges
    |
    v
3D localization
    |
    v
UKF
    |
    v
(x,y,z)
```

---

# Applications

The proposed system can be used for:

### Search and Rescue

Localization of UAVs inside buildings or disaster areas where GNSS is unavailable.

### Indoor Navigation

Autonomous drone navigation inside warehouses, laboratories, tunnels, and industrial facilities.

### GPS-Denied Environments

Localization in environments affected by:

* GNSS blockage
* multipath
* jamming
* indoor operation
* underground operation

### Robotics

UWB-based positioning for:

* mobile robots
* AGVs
* autonomous vehicles
* cooperative robots

### Research

Experimental evaluation of:

* UWB ranging
* ESP-NOW networking
* sensor fusion
* localization filters
* drone navigation

---

# Technology Stack

### Embedded

```text
ESP32-S3
Arduino / ESP-IDF
FreeRTOS
ESP-NOW
SPI
I2C
USB Serial
```

### Ranging

```text
Qorvo DWM3000
IEEE 802.15.4z UWB
TWR
SS-TWR
DS-TWR
```

### Localization

```text
Trilateration
UKF
EKF
Sensor Fusion
```

### Dashboard

```text
Python
NumPy
SciPy
Plotly
Dash
PySerial
Pandas
```

---

# Project Goal

The main goal of this project is to develop a real-time localization and communication system for an autonomous drone operating in GPS-denied environments using:

```text
ESP32-S3
      +
DWM3000 UWB
      +
ESP-NOW
      +
IMU
      +
UKF
```

The initial implementation focuses on:

```text
2 Anchors
+
1 Moving Drone Tag
```

with continuous UWB distance measurements and peer-to-peer ESP-NOW communication.

The architecture is designed to be extended toward multi-anchor and multi-drone localization systems.

---

# License

Add the license appropriate for your project, for example:

```text
MIT License
```

or an academic/research license according to the intended use.

---

