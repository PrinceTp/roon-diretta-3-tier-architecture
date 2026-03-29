# Three-tier Roon + Diretta audio chain with network isolation for low-noise, high-fidelity playback.

## Overview

This document explores a three-tier architecture built around the Roon Core, Diretta Host, and Diretta Target. The approach is centered on achieving an audiophile-grade system by separating processing, transmission, and playback into distinct functional layers. By isolating these roles, the system minimizes electrical noise, stabilizes data flow, and creates a cleaner, more controlled signal path—ultimately resulting in improved clarity, detail, and overall sound quality.

---

## The Diretta Protocol

Diretta is a network audio protocol designed around precise synchronization between the Host and Target. Audio data is transmitted in constant, short intervals to ensure stable and predictable delivery.

In this model, the Target defines the timing behavior and is intentionally kept minimal in processing, while the Host handles data transmission in strict accordance with this timing. By avoiding burst-based transfer, Diretta reduces CPU load fluctuations and power variability on the playback device.

The result is a quieter, more stable electrical environment, where timing inconsistencies and system-induced interference are significantly reduced—an essential requirement for high-end audio reproduction.

---

## The Roon Paradox

The Roon Labs Core is a powerful and feature-rich system, but it performs multiple tasks simultaneously—decoding audio, applying DSP (upsampling, EQ), managing the music library, and streaming data over the network.

This constant workload forces the CPU and system components to operate continuously, leading to fluctuations in power consumption. These fluctuations generate electrical noise (EMI/RFI), which can interfere with nearby audio equipment, especially when the DAC is connected to or located near the same system.

A second challenge lies in Roon’s streaming protocol—RAAT.

RAAT transmits audio in small bursts rather than a continuous flow. Each burst requires the endpoint device (such as a Raspberry Pi) to rapidly wake its CPU, process incoming data, and buffer it. This repeating cycle of idle → spike → idle introduces micro-level electrical disturbances near the DAC.

While the digital data itself remains bit-perfect, these rapid fluctuations can subtly affect the surrounding electrical environment—an effect that becomes more noticeable in resolving, high-end audio systems.

---

## Three Tier Architecture: The Solution To The Paradox

The three-tier architecture separates the audio chain into three clearly defined layers: processing, transmission, and playback. Instead of consolidating all tasks into a single device, each layer is optimized for a specific role, reducing interference and improving overall system stability.

### 1. Processing Layer (Roon Core)

The Roon Labs Core handles all computationally intensive operations, including decoding, DSP, and streaming. As the primary source of electrical noise, it is positioned away from the audio chain to prevent interference with the DAC.

### 2. Transmission Layer (Diretta Host)

The Diretta Host acts as the intermediary, receiving audio from the Core and transmitting it in a controlled, time-consistent manner. Its purpose is to regulate data flow, eliminating burst behavior and minimizing sudden CPU activity, thereby improving timing precision.

### 3. Playback Layer (Diretta Target)

The Diretta Target resides closest to the DAC and performs only the essential task of receiving and outputting audio via USB. With minimal processing requirements, it operates in a low-noise state, providing the DAC with a cleaner electrical environment for optimal performance.

---

## Design Philosophy

This three-tier approach enhances audio performance by isolating processing, transmission, and playback into separate devices, ensuring that the DAC operates under optimal conditions.

- The Roon Core performs all intensive processing remotely, reducing electrical noise near the playback chain.
- The transmission layer ensures a smooth, stable flow of data rather than burst-based delivery.
- The playback layer remains minimal and low-noise, preserving signal integrity at the final stage.

Together, these elements create a more stable signal path with reduced interference, resulting in improved detail, spatial accuracy, and overall sonic refinement.

---

### Audiolinux: The Control Layer

### What Is Audiolinux

AudioLinux is a specialized Linux-based operating system (built on Arch Linux) designed specifically for high-quality audio playback. It is optimized to minimize background processes, reduce CPU interruptions, and lower overall system noise—allowing the hardware to focus almost entirely on audio delivery.

It includes built-in tools tailored for audiophile environments, with support for Roon Bridge, Diretta, HQPlayer NAA, and USB DAC configurations, all accessible through a streamlined menu-driven interface.

### How It Is Used In This Setup

Within this three-tier architecture, AudioLinux runs on the Raspberry Pi devices serving as the Diretta Host and Target.

- On the Diretta Host, it manages incoming audio from the Roon Core and handles controlled data transmission.
- On the Diretta Target, it functions as a minimal, low-noise playback endpoint, delivering audio directly to the DAC via USB.

Its lightweight and optimized design reduces CPU activity and electrical noise, helping preserve a clean and stable signal path at the most critical stage of playback.

---

## Hardware Used For This Setup

LHY OCK-2S ↓ (BNC clock signal)  
LHY AS8 Pro (Network Switch)  
LHY FMC (Fiber Media Converter)  
LHY RPI (Diretta Host)  
LHY RPI Pro (Diretta Target)  

---

## Architecture

<p align="center">
  <img src="/arch.png" alt="System Architecture" width="700"/>
</p>

Roon Core → Network (clocked + isolated) → Host → Target → DAC

---

## Setup Your Own

[Setup Guide](./guide.md) provides a comprehensive guide for setup, covering all steps from initial hardware preparation to final system configuration.

Follow the guide step-by-step to ensure proper installation, network configuration, and Diretta integration. Each section is structured to help you build a stable, isolated audio chain with minimal errors.

It is recommended to complete the setup in sequence without skipping steps, as later configurations depend on earlier stages being correctly implemented.

---
