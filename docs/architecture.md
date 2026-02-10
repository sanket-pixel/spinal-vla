# 🧠 Spinal-VLA System Architecture

This document outlines the **Split-Brain Architecture** used to deploy Vision-Language-Action (VLA) models on the NVIDIA Jetson Orin. 

The core design philosophy is **Decoupling**: The high-frequency motor control loop ("The Spine") must never be blocked by the low-frequency inference loop ("The Brain").

---

## 🏗️ High-Level Design

The system is divided into three distinct biological layers:

```mermaid
graph TD
    subgraph Laptop ["🦿 The Body (Simulation)"]
        Isaac[Isaac Sim / Mock Physics]
        Sensors[Sensor Pub (5555)]
        Motors[Motor Sub (5556)]
        Isaac --> Sensors
        Motors --> Isaac
    end

    subgraph Orin ["🤖 The Robot (Jetson Orin)"]
        subgraph Spine ["The Spine (C++) - 100Hz"]
            Reflex[Reflex Controller]
            Safety[Safety Limits / Dead Man's Switch]
            Reflex --> Safety
        end
        
        subgraph Cortex ["The Cortex (Python) - 5Hz"]
            VLM[VLM / Planner Agent]
        end
        
        %% Communication Lines
        Sensors -.->|ZMQ TCP| Reflex
        Safety -.->|ZMQ TCP| Motors
        
        Reflex <-->|ZMQ IPC / Shared Mem| VLM
    end

```

---

## 🧩 Component Breakdown

### 1. The Body (Simulation Layer)

* **Hardware:** Laptop (RTX 4090)
* **Software:** NVIDIA Isaac Sim (or Python Mock)
* **Role:** Simulates physical reality. It is the "Ground Truth."
* **Interface:**
* **Output:** Publishes `JointState` (position, velocity) @ 100Hz.
* **Input:** Subscribes to `MotorCommand` (velocity targets).



### 2. The Spine (Reflex Layer)

* **Hardware:** Jetson Orin (ARM Cortex-A78AE)
* **Software:** C++17, ZeroMQ
* **Frequency:** **100Hz (Hard Real-Time)**
* **Responsibilities:**
* **Smooth Pursuit:** Interpolates rough goals from the AI into smooth motor curves.
* **Safety Enforcement:** Clamps velocities, checks joint limits.
* **Dead Man's Switch:** If the AI (Cortex) stops sending heartbeats for >500ms, the Spine automatically brakes the robot to a stop.



### 3. The Cortex (Cognitive Layer)

* **Hardware:** Jetson Orin (Ampere GPU / DLA)
* **Software:** Python, PyTorch, Transformers
* **Frequency:** **1-5Hz (Best Effort)**
* **Responsibilities:**
* **Vision:** Processes RGB-D camera frames.
* **Reasoning:** Uses VLM (e.g., Cosmos, CLIP) to understand semantic commands ("Pick up the red apple").
* **Planning:** Generates high-level waypoints (x, y, z) for the Spine to execute.



---

## 📡 Communication Protocol

We use **ZeroMQ (ZMQ)** for the "Nervous System" due to its brokerless low latency and language agnosticism.

| Link | Type | Transport | Data Format | Description |
| --- | --- | --- | --- | --- |
| **Sensation** | `PUB/SUB` | `TCP` | JSON | Joint angles from Sim to Orin. |
| **Action** | `PUB/SUB` | `TCP` | JSON | Velocity commands from Orin to Sim. |
| **Intention** | `REQ/REP` | `IPC` | Binary | High-level goals from Cortex to Spine. |

---

## 🛡️ Safety Systems

The critical innovation in this architecture is that **Safety is handled in C++, not Python.**

1. **Latency Independence:** Even if the Python Garbage Collector pauses the AI for 200ms, the C++ loop continues to send smooth `0.0` velocity commands (or maintains the last valid trajectory), preventing physical jitter.
2. **Boundary Checks:** The Spine enforces hard joint limits before sending commands to the hardware, protecting the motors from "hallucinated" AI commands.

