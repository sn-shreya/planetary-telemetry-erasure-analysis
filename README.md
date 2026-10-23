# planetary-telemetry-erasure-analysis

A Python framework for parsing and analysing asynchronous MAVLink UAV telemetry. The project uses five-second median filtering and an Extended Kalman Filter (EKF) to study vehicle-state estimation when telemetry to a ground station is interrupted and restored. The work is based on ArduPilot SITL and explores problems relevant to autonomous planetary exploration, particularly missing measurements and state estimation during communication interruptions.

# Planetary Telemetry Erasure Analysis

## Overview

This project builds on my autonomous UAV capstone project and extends it towards **Planetary Science and Exploration**.

The original project involved an autonomous quadcopter payload-tracking system using ArduPilot, a Pixhawk flight controller, GPS, and MAVLink telemetry. I am using a software-in-the-loop (SITL) setup to investigate what happens to the vehicle-state data available at the ground station when its telemetry connection is interrupted and later restored.

The analysis starts with a recorded MAVLink `.tlog` file. I extract the available vehicle-state variables, organise them into five-second bins, calculate the median of each variable in each bin, and apply an Extended Kalman Filter to the resulting `median_data`.

The main focus is to understand how missing telemetry affects state estimation and how an estimator behaves when measurements become available again. This is a preliminary simulation study motivated by communication challenges in planetary exploration. It is not intended to represent a complete planetary rover navigation system.

## Objectives

- Parse a recorded MAVLink telemetry log and extract the available vehicle-state variables.
- Examine the difference between GPS receiver measurements and the autopilot's estimated global position.
- Study telemetry availability before, during, and after a ground-station communication interruption.
- Aggregate the extracted measurements into five-second median bins.
- Apply an EKF to `median_data` and compare its estimates with the original median measurements.
- Examine how missing measurements affect the estimator and its behaviour after reconnection.
- Improve the estimator and evaluate its accuracy as the project develops.

## Experimental Setup

The experiment uses the following tools:

## Tool & Purpose

ArduPilot SITL: Runs the simulated vehicle and flight-controller software
Mission Planner: Provides the ground-station interface and records telemetry logs
MAVProxy: Routes MAVLink messages and allows the ground-station telemetry path to be interrupted and restored
`pymavlink`: Reads and decodes MAVLink messages from the `.tlog` file
NumPy: Handles numerical operations used in the EKF
pandas: Organises telemetry data and calculates five-second medians
Matplotlib: Plots the measurements and EKF estimates

### Communication experiment

The recorded experiment covers three phases:

1. **Initial connection:** The ground station receives telemetry from the simulated vehicle.
2. **Telemetry interruption:** The ground-station telemetry path is interrupted while SITL continues running.
3. **Reconnection:** The ground station begins receiving telemetry again.

The purpose is to study the availability of measurements at the ground station during these phases. An interruption in the ground-station link does not necessarily mean that the simulated vehicle has stopped running or that its onboard estimator has stopped operating.

## MAVProxy and Mission Planner connection
 Using the WSL terminal (for ground station control)
1. Prerequisites & Environment Setup: To install the ArduPilot dependencies
   bash
   cd ~/ardupilot
   Tools/environment_install/install-prereqs-ubuntu.sh -y
   . ~/.profile
   <img width="1915" height="1017" alt="Screenshot 2026-10-21 121212" src="https://github.com/user-attachments/assets/c8904479-7b28-4684-9954-85a8e817f206" />

2. Navigate into the ArduCopter directory and launch the simulator:
   cd ~/ardupilot/ArduCopter
   ../Tools/autotest/sim_vehicle.py \
  --console \
  --map \
  --out=127.0.0.1:14550 \
  --mavproxy-args="--logfile=/home/shreya_narayanan/sitl_logs/run_02/telemetry.tlog"
   <img width="1917" height="1017" alt="Screenshot 2026-10-22 111233" src="https://github.com/user-attachments/assets/66bcacc1-fa5c-4796-81c1-5e816bae1d42" />

3. To check connection and ground station control:
   mode guided
   arm throttle
   takeoff 10

4. To interrupt telemetry connection:
   output remove 0
   To reconnect telemetry:
   output add 172.18.64.1:14550
   `All values are according to my computer system`
   <img width="1917" height="1015" alt="Screenshot 2026-10-22 111145" src="https://github.com/user-attachments/assets/689b613e-dca8-4a60-8cdc-86040e3b81d0" />

## Creating mission using Mission Planner
<img width="1920" height="991" alt="Mission Planner 1 3 83 build 1 3 9384 38258 ArduCopter V4 8 0-dev (afeb9c49) on  21-10-2026 15_53_03" src="https://github.com/user-attachments/assets/62960b88-dfc4-4fa9-b183-a398e5e3e8db" />
switching to auto mode from guided mode after mission creation was done through Mission Planner software. It can be done through ground station control as well (WSL terminal)


## Repository Structure

planetary-telemetry-erasure-analysis/
│
├── planetary_rover.ipynb
├── README.md
└── requirements.txt


- planetary_rover.ipynb: contains the telemetry parsing, feature extraction, median aggregation, EKF implementation, and plots.
- requirements.txt: lists the Python dependencies.
- README.md: documents the project and its methodology.

The telemetry `.tlog` file is provided separately when running the notebook.

## Methodology

### 1. Reading the MAVLink log

The notebook uses `pymavlink` to read the recorded `.tlog` file and extract the information contained in its MAVLink messages.

Messages identified as `BAD_DATA` are skipped. If a required variable is not available in the log, it is retained as missing (`NaN`) rather than being filled with an assumed value.

### 2. GPS raw measurements and global position

An important decision in this analysis is the use of `GPS_RAW_INT` instead of `GLOBAL_POSITION_INT` for GPS coordinates.

**`GPS_RAW_INT` — GPS receiver measurements**

`GPS_RAW_INT` reports the position and status information provided by the GPS receiver, including latitude, longitude, altitude, and GPS quality indicators where available. The position refers to the GPS antenna and is subject to GPS measurement errors, satellite geometry, and reception conditions.

These measurements are raw relative to the autopilot's fused position estimate. This does not mean that they are necessarily free from receiver-level processing or other corrections.

**`GLOBAL_POSITION_INT` — Estimated global position**

`GLOBAL_POSITION_INT` reports the autopilot's estimated global position and associated motion information, including altitude and velocity components.

Depending on the flight-controller configuration, the onboard estimator may combine GPS measurements with IMU and other sensor information using an EKF or another estimation method. The resulting position is therefore an estimated vehicle state, rather than simply the GPS receiver's reported coordinates.

**Why I chose `GPS_RAW_INT`**

I chose `GPS_RAW_INT` because I wanted to work with the GPS receiver's reported coordinates rather than directly use the autopilot's fused global-position estimate as the GPS input to my analysis.

I then aggregate the available measurements into five-second median bins and apply a separate EKF to `median_data`. This allows me to investigate a separate state-estimation stage using the measurements available in the recorded log.

The distinction is important, but it has a limitation: not every variable in the dataset is a raw sensor measurement. Some variables, such as attitude, velocity, and waypoint distance, may already be processed or estimated by the autopilot. Their source must be considered when interpreting the EKF results.

### 3. Extracting vehicle-state variables

The notebook tracks ten variables:

## Variable & Description 

`altitude_m`: Altitude in metres
`distance_to_waypoint_m`: Recorded distance to the current waypoint, where available
`distance_travelled_m`: Cumulative distance travelled, if available in the original log
`roll_deg`: Roll angle in degrees
`pitch_deg`: Pitch angle in degrees
`yaw_deg`: Yaw angle in degrees
`vertical_speed_m_s`: Vertical speed in metres per second
`ground_speed_m_s`: Ground speed in metres per second
`latitude_deg`: Latitude in degrees
`longitude_deg`: Longitude in degrees

The values are extracted from their respective MAVLink message fields. Necessary unit conversions are applied, such as converting attitude angles from radians to degrees and scaled GPS coordinates to degrees.

For latitude and longitude, `GPS_RAW_INT` is preferred when extracting GPS receiver measurements.

I do not calculate missing variables during feature extraction. For example, cumulative distance travelled is not calculated from successive GPS coordinates. If a variable cannot be obtained from the original log, it remains unavailable until its source and availability can be investigated.

### 4. Five-second median filtering

The extracted measurements are grouped into five-second time bins. For each variable, the median of the available measurements within a bin is calculated.

The resulting dataset is stored as `median_data` and is used as the input to the EKF.

The median provides a representative value for each time bin and can reduce the influence of isolated extreme measurements. However, it also suppresses some short-duration changes. Rapid movements or brief anomalies within a five-second interval may not be visible in the aggregated data.

Missing measurements are not automatically treated as evidence of a telemetry interruption. A bin may contain missing values because the relevant message was not recorded, the field was unavailable, or no valid measurement was present.

### 5. Extended Kalman Filter

The EKF is applied to **`median_data` only**, not to the raw `telemetry_data`.

The filter uses a state-transition model to predict the vehicle state and a measurement-update step to incorporate available median measurements. When measurements are missing, the filter can continue predicting previously initialised states using its model.

The EKF estimates are stored separately from the original measurements. For example, `altitude_m_ekf` contains the EKF estimate corresponding to `altitude_m`.

Variables without valid initial measurements are not assigned arbitrary starting values. If a variable is unavailable in the original data, the filter cannot produce a meaningful estimate for it without an appropriate source of information or an explicitly justified model assumption.

The current EKF is an initial implementation. Its noise parameters, motion model, handling of missing measurements, and estimation accuracy still need to be evaluated and improved.

## Analysis and Visualisation

The notebook compares the five-second median measurements with their corresponding EKF estimates.

It also examines:

- The availability of each variable across time.
- Bins in which all ten variables are missing.
- Bins in which only some variables are missing.
- Timestamp gaps, where present.
- The estimator's behaviour during the interruption and after telemetry returns.

These checks serve different purposes. A bin with no measurements indicates a lack of usable data in that interval, but does not independently prove that the telemetry connection failed. Similarly, if the timeline has been filled with consecutive five-second bins, a communication interruption may appear as missing measurements rather than as a timestamp gap.

The observed patterns must therefore be compared with the known interruption and reconnection periods.

## Limitations

The current project has several limitations:

- **Simulation:** The experiment uses SITL and does not validate physical flight performance or the actual behaviour of the Pixhawk, GPS receiver, or 3DR telemetry radio.
- **Telemetry availability:** A ground-station log cannot independently establish every vehicle state during a period when the ground station is not receiving data.
- **GPS noise:** Raw GPS measurements can contain errors and may become unavailable. Using `GPS_RAW_INT` does not remove these errors.
- **Mixed data sources:** Some extracted variables may already be estimated or filtered by the autopilot, so the dataset is not entirely composed of raw sensor measurements.
- **Missing variables:** The availability of distance-related variables, particularly cumulative distance travelled, must be verified against the original log.
- **Median filtering:** Five-second aggregation reduces the visibility of short-lived changes and anomalies.
- **EKF accuracy:** Smooth estimates do not necessarily mean accurate estimates. The current filter requires further testing against an appropriate reference.
- **Planetary application:** The experiment investigates a communication-related problem relevant to planetary exploration, but does not demonstrate autonomous navigation on a planetary surface.

## Future Work

The next steps are to:

1. Investigate the availability and validity of distance-related fields in the original MAVLink log.
2. Verify the timing of the interruption and reconnection against the recorded measurements.
3. Compare GPS receiver measurements with the autopilot's estimated global position.
4. Improve the EKF state-transition model and noise parameters.
5. Study how the estimator behaves when measurements are missing for different durations.
6. Evaluate the estimates against a suitable reference trajectory or ground truth, where available.
7. Use quantitative error metrics to assess estimation performance.
8. Document the results and limitations of each improvement.

The estimator will be improved incrementally. Any claims about its accuracy will depend on the results of these evaluations rather than on the smoothness of the plotted estimates alone.

## Reproducing the Analysis

1. Clone or download the repository.
2. Install the dependencies listed in `requirements.txt`.
3. Open `planetary_rover.ipynb` in Jupyter or Google Colab.
4. Provide the recorded `.tlog` file when prompted.
5. Run the notebook cells in sequence to parse the log, extract the variables, create `median_data`, apply the EKF, and generate the plots.
6. Review the measurements, estimates, and missing-data indicators before drawing conclusions.

The results depend on the input log, the fields available in its MAVLink messages, and the estimator parameters.

## Motivation

Communication interruptions are one of the challenges that must be considered when developing autonomous systems for planetary exploration. A vehicle may continue operating while information available to a remote ground station becomes temporarily unavailable.

This project uses an autonomous UAV simulation to study that problem at a preliminary level. By analysing recorded telemetry, aggregating measurements, and applying a separate EKF, I aim to develop a better understanding of state estimation when measurements are missing and when communication is restored.

The longer-term goal is to use this work as a starting point for more rigorous studies of autonomous navigation and state estimation for planetary exploration.
