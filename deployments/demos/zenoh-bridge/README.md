# OpenADKit Remote Visualization with Zenoh Bridge: Implementation Report and User Manual

## 1. Introduction

This document provides a comprehensive implementation guide and technical overview for the distributed architecture of the OpenADKit project. The core objective is to separate compute-intensive and lightweight components of an autonomous driving system, enabling deployment across different hardware. For example, the core Autoware software stack runs on the local side (e.g., vehicle, or a powerful simulation server). Users can remotely visualize and manage Autoware from their laptops or a cloud-based management system.

To achieve this, we utilize [Zenoh](https://zenoh.io/) as a high-performance, low-latency communication protocol, paired with the [`zenoh-bridge-ros2dds`](https://github.com/eclipse-zenoh/zenoh-plugin-ros2dds) tool to seamlessly connect two ROS 2 (Robot Operating System 2) environments isolated by Docker virtual networks. This manual covers architecture design, setup steps, system startup, and troubleshooting, providing complete operational guidance.

## 2. Detailed Architecture Design

### 2.1. Core Concept: Client-Server Model

The system decouples the previously monolithic architecture into a client-server model:

- **Vehicle/Server Side**: Includes components with high computational demands (especially CPU and GPU).
  - `autoware`: Core perception, decision-making, and planning modules.
  - `scenario_simulator`: Generates virtual traffic environments, obstacles, and vehicle dynamics, providing simulated sensor data for Autoware.
  - **Deployment**: Typically on vehicles or powerful servers.

- **Monitor Side**: Includes lightweight components critical for user interaction and visualization.
  - `visualizer`: Based on RViz2, encapsulated with noVNC, allowing users to access the visualization interface via any modern web browser without installing ROS 2 or RViz locally.
  - **Deployment**: Typically on a user's laptop or a cloud-based management system, where low computational power is sufficient.

### 2.2. Architecture Diagram

The following Mermaid diagram illustrates the components, networks, and data flow paths.

```mermaid
graph TD
    %% --- Style Definitions ---
    classDef machine fill:#f9f9f9,stroke:#333,stroke-width:2px,stroke-dasharray:5 5
    classDef network fill:#e6f3ff,stroke:#0066cc,stroke-width:1.5px
    classDef service fill:#ffffff,stroke:#333
    classDef bridge fill:#fffbe6,stroke:#f0ad4e
    classDef invisible stroke:none,fill:none

    %% === Monitor ===
    subgraph Monitor["Monitor (noVNC)"]
        direction LR
        class Monitor machine

        subgraph VisualizerNet[visualizer_net Network]
            direction TB
            class VisualizerNet network
            
            visualizer["**Visualizer**<br><br>Based on RViz2<br>Provides noVNC Remote Desktop"]
            visualizer_bridge["**Visualizer Zenoh Bridge**<br><br><u>Role</u>: Client<br>Converts Zenoh ↔️ DDS"]
            class visualizer service
            class visualizer_bridge bridge
            
            visualizer -->|"ROS2 DDS Data"| visualizer_bridge
        end
    end

    %% === Vehicle Server ===
    subgraph VehicleServer[Vehicle Server]
        direction LR
        class VehicleServer machine

        subgraph AutowareNet[autoware_net Network]
            direction TB
            class AutowareNet network

            autoware["**Autoware**<br><br>Core Autonomous Driving Algorithms<br>Perception, Planning, Control"]
            scenario_simulator["**Scenario Simulator**<br><br>Provides Simulation Scenarios<br>With Sensor Data"]
            autoware_bridge["**Autoware Zenoh Bridge**<br><br><u>Role</u>: Router<br>Listens on TCP/7447<br>Converts DDS ↔️ Zenoh"]
            class autoware,scenario_simulator service
            class autoware_bridge bridge

            autoware <-->|"ROS2 DDS Data"| scenario_simulator
            autoware -->|"ROS2 DDS Data"| autoware_bridge
            scenario_simulator -->|"ROS2 DDS Data"| autoware_bridge
        end
    end

    %% === External & Cross-Network Connections ===
    user[fa:fa-user User] -->|"HTTP (Port 6080)"| visualizer
    visualizer_bridge <-->|"<b>Zenoh Protocol over zenoh_net</b><br>Connects to tcp/autoware_zenoh_bridge:7447"| autoware_bridge
```

### 2.3. Network Isolation and Communication Bridge

- **Network Design**:
  - `autoware_net`: An isolated virtual network for `autoware` and `scenario_simulator`, using ROS 2 DDS multicast for low-latency communication.
  - `visualizer_net`: An isolated network for `visualizer`, simulating physical or logical separation from the server.
  - TCP/IP (use `zenoh_net` docker network in this demo for simplicity): The network that is possible to connect `autoware_zenoh_bridge` and `visualizer_zenoh_bridge`, ensuring a clean cross-domain data transmission path.

- **Communication Core: Zenoh Bridge**:
  - `autoware_zenoh_bridge` (`zenoh-bridge-ros2dds` container): Acts as a **Router**, scans ROS 2 topics in `autoware_net`, converts them to Zenoh format, and listens for client connections on TCP/7447 in `zenoh_net`.
  - `visualizer_zenoh_bridge` (`zenoh-bridge-ros2dds` container): Acts as a **Client**, connecting to `autoware_zenoh_bridge` via `zenoh_net`, receiving Zenoh data, and converting it back to ROS 2 DDS for `visualizer` to subscribe.
  - `config/zenoh-bridge-ros2dds.json5`: A configuration file defining the bridge's mode, listening endpoints, and topic filtering rules, allowing precise control over transmitted data to optimize bandwidth.

## 3. User Manual

### 3.1. Prerequisites

Ensure the following software is installed:
1. **Docker Engine**: See [Docker Installation Guide](https://docs.docker.com/engine/install/).
2. **Docker Compose**: Usually included with Docker Desktop; otherwise, see [Docker Compose Installation Guide](https://docs.docker.com/compose/install/).
3. **Git**: For cloning the project.
4. A stable internet connection for pulling Docker images.

### 3.2. Installation and Setup

1. **Clone the Project**:
   Execute the following commands in a terminal:
   ```bash
   git clone https://github.com/autowarefoundation/openadkit
   cd openadkit
   ```

2. **Verify Directory Structure**:
   Ensure the project includes:
   ```
   .
   ├── README.md
   ├── docker-compose.yaml
   └── config/
       └── zenoh-bridge-ros2dds.json5
   ```
   Modify `zenoh-bridge-ros2dds.json5` as needed to filter topics.

### 3.3. Starting the System

1. **Launch All Services**:
   In the project root directory, run:
   ```bash
   docker-compose up -d
   ```
   - `-d` runs containers in the background.
   - First-time execution may take a few minutes to pull images from `ghcr.io` and `eclipse`.

2. **Monitor Startup Logs (Optional)**:
   View service logs to check startup status:
   ```bash
   docker-compose logs -f
   ```

### 3.4. Verification and Usage

1. **Check Container Status**:
   Run `docker ps` or `docker-compose ps` to ensure all containers are running:
   - `autoware`
   - `scenario_simulator`
   - `visualizer`
   - `autoware_zenoh_bridge`
   - `visualizer_zenoh_bridge`

2. **Access noVNC Visualization Interface**:
   Open a browser and navigate to:
   ```
   http://localhost:6080
   ```
   Use the default password `openadkit`.

3. **Verify Operation**:
   - The noVNC interface should display RViz2.
   - If `Global Status` shows `OK` (green), the system is running correctly, showing maps, autonomous vehicle models, and simulated objects.
   - If `Warning` appears, refer to the "Troubleshooting" section.

4. **Stop the System**:
   Stop and remove containers and networks:
   ```bash
   docker-compose down -v
   ```
   - `-v` removes the `autoware_map` volume; omit to retain map data.

## 4. Troubleshooting

### Issue 1: Visualizer Shows "Global Status: Warning" or Blank Screen

- **Cause**: A race condition where ROS 2 nodes start before the Zenoh bridge is ready.
- **Solutions**:
  1. **Restart Services**:
     ```bash
     docker-compose restart
     ```
  2. **Increase Wait Time**:
     Edit `docker-compose.yaml`, e.g., change `autoware`’s `sleep 16` to `sleep 30`:
     ```yaml
     services:
       autoware:
         entrypoint: >
           bash -c "echo 'Waiting longer for bridges...' && sleep 30 && echo 'Starting...' && /entrypoint.sh"
     ```
     Then run:
     ```bash
     docker-compose up -d --force-recreate
     ```
  3. **Staged Startup**:
     ```bash
     docker-compose up -d autoware_zenoh_bridge visualizer_zenoh_bridge visualizer
     sleep 15
     docker-compose up -d autoware scenario_simulator
     ```

### Issue 2: Port Conflict (Port is already allocated)

- **Cause**: Ports `6080` or `7447` are in use by another program.
- **Solution**:
  - Stop the program using the port.
  - Or modify `docker-compose.yaml`, e.g., change `6080:6080` to `8080:6080`, and access via `http://localhost:8080`.

### Issue 3: Container Fails to Start with `file not found` or Permission Issues

- **Cause**: The `config/zenoh-bridge-ros2dds.json5` file is missing or inaccessible.
- **Solution**:
  - Verify the `config` directory and file exist.
  - On Linux/macOS, check file permissions to ensure Docker can read them.