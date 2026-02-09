Bulk Energy Flux Processing (ROS vs. nROS)
Overview

This repository contains a Python workflow for calculating the mean energy flux components during:

Rain-on-Snow (ROS) melt events

Non-Rain-on-Snow (nROS) melt events

The script integrates two distinct datasets to quantify the physical drivers of snowmelt:

Event Data: Custom NetCDF files defining when and where melt events occurred
(snowmelt_events_YYYY.nc)

Forcing Data: Raw ERA5-Land accumulated surface energy flux files (J/m²)

The output consists of yearly NetCDF files containing spatial maps of the average energy flux intensity (W/m²) during ROS and nROS events.

Dependencies

Python 3.x

xarray

numpy

netCDF4 (xarray backend)

dask (optional, recommended for large datasets)

Install via:

pip install xarray numpy netCDF4 dask

Configuration

Before running the script, update the configuration section:

# Years to process
YEARS = range(2017, 2020)

# Input Directories
EVENT_DIR = '/path/to/snowmelt_events/'   # Contains snowmelt_events_YYYY.nc
FLUX_DIR  = '/path/to/raw_era5_fluxes/'   # Contains ERA5_sshf_YYYY.nc, etc.

# Output Directory
OUTPUT_DIR = '/path/to/output/'

Methodology & Algorithm

The workflow proceeds in the following stages:

1. Data Loading

For each year:

Loads event file:

ros_melt_amount

nros_melt_amount

Loads four ERA5-Land accumulated energy flux variables:

Variable	Description
sshf	Sensible Heat Flux
slhf	Latent Heat Flux
ssr	Net Shortwave (Solar) Radiation
str	Net Longwave (Thermal) Radiation
2. De-accumulation (Physics Correction)

ERA5-Land energy flux variables are stored as accumulated energy (J/m²) that resets daily at 00:00 UTC.

To convert them into physical flux intensity (W/m²), temporal differencing is applied:

Step A: Temporal Differencing
Δ
𝐸
=
𝐸
(
𝑡
)
−
𝐸
(
𝑡
−
1
)
ΔE=E(t)−E(t−1)

This converts cumulative energy into interval energy.

Step B: Reset Handling (03:00 UTC)

Because accumulation resets at 00:00:

00:00 contains previous day's total (large value)

03:00 contains first accumulation of the new day

Subtracting 00:00 from 03:00 would produce a large negative value.

Logic applied:

if hour == 3:
    use raw value
else:
    use difference

Step C: Solar Radiation Safety Clamp

Small negative values in ssr (numerical noise) are set to 0.

3. Alignment & Unit Conversion
Temporal Alignment

The .diff() operation removes the first time step.

To ensure exact matching between event and flux datasets:

common_times = np.intersect1d(flux.valid_time, event.valid_time)


Both datasets are subset to this intersection.

Unit Conversion

Input:

Energy increment in J/m² per 3-hour interval

Conversion to power:

Power
(
𝑊
/
𝑚
2
)
=
Energy
(
𝐽
/
𝑚
2
)
3
×
3600
Power(W/m
2
)=
3×3600
Energy(J/m
2
)
	​


This yields mean flux intensity over each 3-hour interval.

4. Event Masking & Conditional Averaging

The script computes flux averages only during event hours, not across the full year.

Mask Definitions
ROS mask  = ros_melt_amount  > -9999
nROS mask = nros_melt_amount > -9999

Conditional Averaging

For each flux component:

𝐹
‾
𝑅
𝑂
𝑆
(
𝑥
,
𝑦
)
=
1
𝑁
𝑅
𝑂
𝑆
∑
𝑡
∈
𝑅
𝑂
𝑆
𝐹
(
𝑥
,
𝑦
,
𝑡
)
F
ROS
	​

(x,y)=
N
ROS
	​

1
	​

t∈ROS
∑
	​

F(x,y,t)

Result:

One spatial map per flux variable

Separate maps for ROS and nROS

Key Technical Decisions
Why reset logic at 03:00?

Accumulation resets at 00:00 UTC.

00:00 contains previous day’s full accumulation.

03:00 is first valid increment of the new day.

Therefore:

At 03:00 → use raw value

At all other hours → use differenced value

Why do we lose one timestep?

Differencing (.diff()) reduces array size by one.

The first timestamp (Jan 1, 00:00) has no previous value to subtract.

This is expected and physically correct.

Output Files

Each year produces:

mean_flux_components_YYYY_direct.nc

Output Variables

Each NetCDF contains eight variables:

Variable Name	Description	Units
mean_ros_sensible	Avg. Sensible Heat during ROS	W/m²
mean_ros_latent	Avg. Latent Heat during ROS	W/m²
mean_ros_solar	Avg. Net Solar Radiation during ROS	W/m²
mean_ros_thermal	Avg. Net Thermal Radiation during ROS	W/m²
mean_nros_sensible	Avg. Sensible Heat during nROS	W/m²
mean_nros_latent	Avg. Latent Heat during nROS	W/m²
mean_nros_solar	Avg. Net Solar Radiation during nROS	W/m²
mean_nros_thermal	Avg. Net Thermal Radiation during nROS	W/m²
Scientific Interpretation

The resulting maps represent:

The average physical energy contribution (W/m²) from each flux component during melt events.

These maps can be used to:

Diagnose dominant melt drivers

Compare ROS vs. nROS energy structures

Develop climatologies of melt energetics

Support hydroclimatic attribution studies

Repository Structure (Suggested)
├── bulk_flux_processing.py
├── README.md
├── data/
│   ├── snowmelt_events_YYYY.nc
│   └── ERA5_flux_files/
└── outputs/
    └── mean_flux_components_YYYY_direct.nc

Notes

Ensure ERA5-Land fluxes and snowmelt follow identical accumulation conventions.

Temporal alignment is critical.

All accumulated variables must be de-accumulated before physical interpretation.
