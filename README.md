# Fullstack IMU Pipeline (ESP32 + micro-ROS + ROS 2)

This project implements an **end-to-end IMU processing pipeline**, starting from raw sensor measurements on an embedded microcontroller and ending with orientation estimation and visualization in ROS 2.

The system reads motion data from an **ICM-20948 9-axis IMU**, streams the measurements from an **ESP32 using micro-ROS**, and estimates orientation using the **Madgwick filter** on a host computer running **ROS 2**.

The goal of the project is to demonstrate how **embedded systems and robotics middleware integrate to form a complete sensing pipeline**.

---

## System Architecture

<p align="center">
  <img src="docs/architecture.png" width="900"/>
</p>

Pipeline stages:

1. **Sensor Acquisition (Embedded)**  
   The ESP32 reads accelerometer, gyroscope, and magnetometer data from the ICM-20948 via I²C.

2. **micro-ROS Communication**  
   The ESP32 runs a micro-ROS client that publishes IMU data to a host computer over a serial connection.

3. **ROS 2 Integration**  
   The micro-ROS Agent bridges the embedded client into the ROS 2 graph.

4. **Orientation Estimation**  
   The `imu_filter_madgwick` package estimates IMU orientation from the sensor data.

5. **Visualization**  
   RViz visualizes the orientation estimate and coordinate frames.

## Demo



https://github.com/user-attachments/assets/7fa7b564-36bb-4191-b4d8-bdf679163d4d



---

## Repository Structure

This repository ties together two independent components:

```
Fullstack_imu_filter
│
├── esp32_firmware
│   Embedded firmware running on the ESP32
│   - IMU driver (ICM-20948)
│   - FreeRTOS acquisition task
│   - micro-ROS client
│
├── madgwick_sensor_fusion
│   ROS 2 integration package
│   - micro-ROS agent launch
│   - Madgwick filter configuration
│   - RViz visualization
│
└── docs
    └── architecture.png
```

### Component Repositories

**Embedded firmware**

```
https://github.com/spereira02/esp32_firmware
```

Implements the embedded layer:

- IMU driver
- FreeRTOS sensor acquisition
- micro-ROS client publishing IMU data

---

**ROS 2 processing layer**

```
https://github.com/spereira02/madgwick_sensor_fusion
```

Handles the host-side computation:

- micro-ROS agent
- Madgwick filter
- RViz visualization

---

## Cloning the Project

Clone the full system including submodules:

```bash
git clone --recurse-submodules https://github.com/spereira02/Fullstack_imu_filter.git
```

Repository layout after cloning:

```
Fullstack_imu_filter/
├── esp32_firmware
├── madgwick_sensor_fusion
└── docs
```

---

## Running the Full Pipeline

### 1. Flash the ESP32 Firmware

Follow the instructions in:

```
esp32_firmware/README.md
```

This builds and flashes the firmware that reads the IMU and publishes data through micro-ROS.

**Note:** Firmware building and flashing were tested on **Windows (PlatformIO)**.  
ROS 2 integration and runtime deployment were performed on **Ubuntu**.

---

### 2. Build the ROS 2 Workspace

Clone both repositories into a ROS workspace:

```
ros2_ws/
└── src/
    ├── madgwick_sensor_fusion
    └── micro-ROS-Agent
```

Then build:

```bash
cd ros2_ws
rosdep install --from-paths src --ignore-src -r -y
colcon build
source install/setup.bash
```

---

### 3. Launch the ROS 2 System

Start the ROS pipeline:

```bash
ros2 launch imu_madgwick madgwick_launch.py
```

This launches:

- micro-ROS Agent  
- Madgwick filter node  
- RViz visualization

---

### 4. Start the Embedded Client

Once the agent is running, reset the ESP32 to establish the micro-ROS session.

The IMU data should now appear on ROS topics such as:

```
/imu/data_raw
/imu/mag
```

---

## What This Project Demonstrates

From an **embedded systems perspective**:

- sensor integration over I²C
- FreeRTOS task-based firmware architecture
- micro-ROS communication from a microcontroller

From a **robotics systems perspective**:

- bridging embedded hardware into a ROS 2 graph
- running orientation estimation with the Madgwick filter
- visualizing sensor fusion results in RViz

---

## Technologies Used

- ESP32
- FreeRTOS
- micro-ROS
- ROS 2 (Jazzy)
- ICM-20948 IMU
- Madgwick sensor fusion algorithm
- RViz

## Troubleshooting & Engineering Challenges

During development, several integration issues had to be diagnosed across both the embedded and ROS2 sides of the pipeline:

- **Serial throughput bottleneck:** Although the IMU was sampled at **50 Hz**, the effective ROS2 publish rate initially dropped to around **12 Hz**. By increasing the UART baudrate, the pipeline reached a stable **~50 Hz**, showing that the main bottleneck was communication bandwidth rather than filter computation.

- **Timestamp mismatch / `TF_OLD_DATA`:** RViz2 intermittently reported `TF_OLD_DATA` errors. The root cause was an unsynchronized timestamp source on the MCU, which produced message header timestamps that did not match ROS2 host time. After switching to synchronized epoch-based timestamps, TF timing became consistent.

- **System-level debugging workflow:** These issues were diagnosed using tools such as `ros2 topic hz`, topic echoing, TF inspection, and targeted testing across both firmware and ROS2 nodes. This project was a strong exercise in debugging timing, transport, and middleware interactions in a real robotics pipeline.
