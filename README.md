# Airbus A321 Flight Telemetry & Performance Analytics Pipeline

An end-to-end data engineering and flight physics optimization pipeline that ingests high-frequency (10Hz) mobile sensor logs to accurately reconstruct flight trajectories, calculate dynamic aircraft fuel burn using OpenAP, and analyze flight safety events (such as low-altitude Go-Arounds).

---

## 📌 Project Overview & Data Origin
The dataset used in this project was recorded individually by me over a live commercial flight using the **Phyphox app**. The raw sensor outputs were originally captured as individual, disjointed CSV files tracking the aircraft's kinematic and spatial positioning. 

### Data Stream Breakdown
* **GPS Telemetry:** Logs real-world global coordinates, absolute ellipsoidal height, trajectory direction, and true ground velocity.
* **Attitude & Orientation Log:** Captures exact vehicle physics matrices including pitch angles, roll metrics, and quaternions ($w, x, y, z$).
* **IMU Accelerometer & Gyroscope Streams:** Logs raw high-frequency multi-axis structural accelerations, linear displacement vectors, and raw angular rotation rates (rad/s).

Aviation telemetry pipelines often suffer from asynchronous sensor capture, where GPS positions, IMU structural parameters, and magnetic headings are logged independently at staggered millisecond thresholds. This project unifies those disjointed data streams into a cohesive, chronological timeline to drive a flight performance physics engine powered by the **OpenAP** library.

### Key Pipeline Features
* **Asynchronous Data Alignment:** Collapses overlapping timestamps and resolves massive `dt=0` calculation errors caused by independent multi-sensor frequencies.
* **Dynamic Mass Integration:** Loops real-world operational constraints (such as mid-flight starting weights) dynamically through an aerodynamic lift/drag matrix.
* **OpenAP Performance Modeling:** Utilizes the Open Aircraft Performance (OpenAP) library to calculate high-fidelity Thrust and Fuel Flow coefficients specific to the Airbus A321 airframe.
* **Go-Around Phase Tracking:** Automatically isolates critical, low-altitude abort procedures to analyze engine spool-up delays and peak thrust parameters ($TOGA$).
* **Sensor-Fusion Mapping:** Aligns external magnetometer variations with internal gyroscope angular rotation rates.

---

## 📊 Analytics & Insights

### 1. Master Flight Profile (Main Overview)
By shifting the evaluation framework from basic time boundaries to an aggregated sensor timeline, the pipeline generates accurate cruise performance calculations matching real-world Airbus Flight Crew Operating Manuals (FCOM). 

Adjusting the starting mass parameter from a near-takeoff standard down to an accurate mid-flight weight of **64,500 kg** (to reflect fuel burnoff on the short Indore-to-Mumbai sector) scaled down the calculated induced drag exponentially—eliminating a 15% model inflation and delivering textbook validation.

*Overall Flight Telemetry Performance Profile:*
![Flight Performance Dashboard Thumbnail](photo1.jpeg)

* **Analyzed Segment Duration:** 20.38 Minutes 
* **Total Fuel Expended:** 1,024.47 kg
* **Average Cruise Burn Rate:** 49.67 kg/min (~2,980 kg/hr via OpenAP)

### 2. High-Thrust Safety Incident Analysis (Go-Around Metrics)
Near the end of the telemetry log, the pipeline automatically captures a **Go-Around** event as the aircraft descends to its minimum threshold near Mumbai ($\approx 5,373$ meters) and applies maximum power to climb away safely. 

Using OpenAP's kinematic engine, the model captures instantaneous fuel flow rates spiking sharply away from the clean cruise baseline of **49.67 kg/min** up to heavy peak values, directly capturing engine spool-up lag parameters as the turbofans compress air and transition to full $TOGA$ power.

*Targeted Fuel Flow & Go-Around Event Analysis:*
![Fuel Metrics and Go-Around Analysis](photo2.jpeg)

---

## 🛠️ Tech Stack & Libraries
* **Language:** Python 3.x
* **Development Environment:** Jupyter Notebook
* **Data Processing & Analytics:** NumPy, Pandas
* **Aviation Physics Engine:** OpenAP (Open Aircraft Performance Modeling Library)
* **Data Visualization:** Matplotlib, Seaborn

---

## ⚙️ Data Pipeline Architecture

### Data Cleaning & Synchronization Pattern
To prevent calculus loops from breaking over `NaN` values or zero-time intervals, individual logs (`GPS`, `IMU`, `Gyroscope`, `Magnetometer`) are integrated using an outer-merge tracking pattern:

```python
# Unifying independent sensor frequencies chronologically
df_grouped = df.groupby('Time (s)', as_index=False).first()
df_grouped = df_grouped.sort_values(by='Time (s)').reset_index(drop=True)

# Filling gaps horizontally across rows
df_grouped[state_columns] = df_grouped[state_columns].ffill()
