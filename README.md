# STM32 Dual-MCU Audio & Ultrasonic Acquisition System

## Overview
This repository contains a complete, full-stack data acquisition system. It utilizes a **Dual-STM32 architecture** to achieve high-speed (44.4 kHz), 12-bit audio sampling without overloading a single processor. 

The system features two operating modes: a timed **Manual Mode** and a smart **Auto Mode** that uses an HC-SR04 ultrasonic sensor to automatically start and stop recording based on hand proximity. The backend includes a Python pipeline for data visualization (PNG) and processing, along with a custom C-based WAV file compiler.

## System Architecture
To ensure zero data loss at 44.4 kHz, the workload is split between two STM32 microcontrollers:
1. **Master Node (Data Acquisition):** - Uses `TIM6` to trigger `ADC1` at precisely 44.4 kHz.
   - Streams raw ADC data directly into memory via DMA.
   - Pushes 512-sample blocks to the Slave node via High-Speed SPI (DMA).
2. **Slave Node (Processing & Control):**
   - Receives SPI data via DMA (Ping-Pong buffer via Half/Full callbacks).
   - **On-the-fly DSP:** Performs real-time Outlier Rejection (Median-of-3) and a Moving Average Filter (Window=3) to clean the audio signal.
   - Reads the HC-SR04 Ultrasonic sensor using `TIM6` and EXTI interrupts for non-blocking distance measurement.
   - Transmits the cleaned payload and control tokens (Start/Stop/Sample Rate) to the PC via UART at **921600 baud**.
3. **PC Host (Python & C):**
   - Python script reads the 921600 baud serial stream.
   - Re-aligns 12-bit data, removes DC offsets, and applies a Butterworth High-Pass Filter (SciPy).
   - Automatically compiles and executes a custom `convert_to_wav.c` tool to perfectly encode the binary data into a `.wav` file.

## Repository Structure
* `/Master_STM32/`: Code for the ADC sampling and SPI transmission.
* `/Slave_STM32_V1/`: Slave code optimized for Manual Mode testing.
* `/Slave_STM32_V2/`: Advanced Slave code with Ultrasonic auto-triggering, dynamic thresholding, and on-board DSP filtering.
* `/PC_Backend/`: Contains the Python listener script and `convert_to_wav.c`.

## Hardware Pinout
**Master STM32:**
* `ADC1_IN8` - Audio Input
* `SPI1_MOSI / SCK` - Connected to Slave STM32

**Slave STM32:**
* `SPI1_MISO / SCK` - Connected to Master STM32
* `USART2 (TX: PA2 / RX: PA3)` - PC Connection via USB-TTL
* `PA11 (Output)` - HC-SR04 Ultrasonic Trigger
* `PA8 (EXTI Input)` - HC-SR04 Ultrasonic Echo
* `PB3 (Output)` - Status LED (Recording Indicator)

## How to Run
1. Flash the Master STM32 and Slave STM32 (V2 recommended).
2. Wire the SPI lines between the two boards, and connect the HC-SR04.
3. Connect the Slave STM32 to the PC via USB-TTL.
4. Run the Python script. The script will auto-compile the WAV converter.
5. **Menu Options:**
   - **Mode 1:** Enter the number of seconds to manually record.
   - **Mode 2 (Auto):** Set a trigger distance. Bring your hand within the threshold to start recording automatically, and move it away to stop.
6. Check the `/recordings` folder for the generated `.csv`, `.png` waveform graph, and `.wav` audio file!

## Demo
*At above*
