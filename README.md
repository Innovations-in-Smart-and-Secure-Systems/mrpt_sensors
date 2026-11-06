# Jabalí IMU and magnetometer drivers

Minimal ROS 2 driver repository for the external Taobotics/HandsFree IMU on
Jabalí 4 (`wildboar4`). It publishes inertial data and, when provided by the
sensor, a separate magnetic-field message. ROS 2 Humble is the verified build
environment.

## Repository scope

Only two packages are built:

| Package | Purpose |
| --- | --- |
| `mrpt_sensor_imu_taobotics` | Serial IMU node and configurable launch file |
| `mrpt_sensorlib` | MRPT observation conversion, IMU/magnetometer publishers, static TF, diagnostics, retry handling and optional rawlog recording |

The `main` branch is derived from the original `ros2` branch at commit
`fd07363c958b646d4eeb6627bedda0b0a73ff80d`. GNSS drivers, Bumblebee camera,
Novatel messages, generic sensor executable, all-sensors metapackage,
Velodyne examples, maintenance scripts and GitHub workflows were removed.
GPS and image publishing code was also removed from the retained library.
The package names and executable name remain compatible with the robot's
`wildboar_sensors` bringup. Existing changelogs preserve upstream history;
they also mention components no longer included here.

Source: [Innovations-in-Smart-and-Secure-Systems/mrpt_sensors](https://github.com/Innovations-in-Smart-and-Secure-Systems/mrpt_sensors),
based on [mrpt-ros-pkg/mrpt_sensors](https://github.com/mrpt-ros-pkg/mrpt_sensors).
Original copyright notices and BSD-3-Clause licensing are retained.

## Hardware observed on Jabalí 4

Read-only inspection on 2026-11-06 verified the following running configuration:

| Setting | Observed value |
| --- | --- |
| Host | `wildboar4` |
| OS / ROS | Ubuntu 22.04 / ROS 2 Humble, ARM64 |
| ROS domain / namespace | `104` / `wildboar4` |
| MRPT sensor model parameter | `hfi-a9` |
| USB identity | `Silicon_Labs_HandsFree_IMU_USB_to_UART_Bridge_Controller_0001` |
| Serial device | `/dev/serial/by-id/usb-Silicon_Labs_HandsFree_IMU_USB_to_UART_Bridge_Controller_0001-if00-port0` |
| Node | `/wildboar4/mrpt_sensor_imu_taobotics` |
| IMU | `/wildboar4/imu`, `sensor_msgs/msg/Imu` |
| Magnetometer | `/wildboar4/imu/mag`, `sensor_msgs/msg/MagneticField` |
| Sensor / parent frame | `imu_link` / `box` |
| Mounting pose parameters | All zero in the running configuration; physical extrinsics were not measured |

Live messages were received from both topics with `frame_id: imu_link`.
The model is identified by the running parser configuration, not by a physical
label inspection. The existing robot driver was already at the source commit
listed above. No robot files or services were changed during inspection.

The ZED camera separately provides `/wildboar4/zed/imu/data` and
`/wildboar4/zed/imu/mag`; those require the ZED wrapper and are outside this
repository. The inspected `wildboar_localization/launch/imu_compass.launch.py`
currently combines the external IMU with the **ZED magnetometer**. Switching
that consumer to `imu/mag` requires checking calibration and frame alignment;
this repository does not change the localization configuration.

## Installation and build

Use an existing ROS 2 Humble installation with `colcon` and `rosdep` configured.
Clone into a workspace that does not already contain copies of these packages:

```bash
mkdir -p ~/imu_ws/src
cd ~/imu_ws/src
git clone --branch main git@github.com:Innovations-in-Smart-and-Secure-Systems/mrpt_sensors.git
cd ~/imu_ws
source /opt/ros/humble/setup.bash
rosdep install --from-paths src --ignore-src -r -y
colcon build --packages-select mrpt_sensorlib mrpt_sensor_imu_taobotics
source install/setup.bash
```

Dependencies include MRPT hwdrivers, `mrpt_libros_bridge`, `mrpt_msgs`,
`rclcpp`, TF2, `sensor_msgs`, `diagnostic_updater`, `launch` and `launch_ros`.
The local build was verified with the ROS Humble MRPT 3.5.0 hwdrivers and
`mrpt_libros_bridge` 4.0.0 packages. Older MRPT installations may lack the
Taobotics driver or its magnetic-field decoding; inspect the MRPT version if
IMU data arrives without magnetic data.

The user running the driver needs serial-device access, typically membership
of `dialout`. After adding the user to that group, log out and back in:

```bash
sudo usermod -aG dialout "$USER"
ls -l /dev/serial/by-id/
```

## Launch on Jabalí 4

The robot's `sensors.service` already owns the serial port. Use its existing
bringup for normal operation. Run the following standalone command only when
the existing IMU process is stopped during a planned maintenance session;
two processes must not read the same device.

```bash
source /opt/ros/humble/setup.bash
source ~/imu_ws/install/setup.bash
export ROS_DOMAIN_ID=104
ros2 launch mrpt_sensor_imu_taobotics mrpt_sensor_imu_taobotics.launch.py \
  namespace:=wildboar4 \
  serial_port:=/dev/serial/by-id/usb-Silicon_Labs_HandsFree_IMU_USB_to_UART_Bridge_Controller_0001-if00-port0 \
  sensor_model:=hfi-a9 \
  publish_topic:=imu \
  sensor_frame_id:=imu_link \
  robot_frame_id:=box
```

This reproduces the observed zero mounting pose; set `sensor_pose_x/y/z`
(metres) and `sensor_pose_yaw/pitch/roll` (degrees) after measuring the mount.
If URDF/robot_state_publisher already owns this transform, add
`publish_sensor_pose_tf:=false` to avoid duplicate TF authorities.

For another robot, specify its actual serial path, namespace and frames.
Without overrides, the launch uses namespace `taobotics`, topic `sensor`,
frames `base_link` → `imu`, and parser `hfi-a9`. `serial_port` is required.
`hfi-b6` is also supported by the upstream driver, but its documented protocol
has no magnetometer fields; do not expect a magnetic topic from every model.

```bash
ros2 launch mrpt_sensor_imu_taobotics mrpt_sensor_imu_taobotics.launch.py --show-args
```

The installed robot bringup currently has its own launch file in
`wildboar_sensors`. Updating that file is a separate deployment task; building
this repository does not replace it. This repository's launch now exposes
`namespace` and `publish_sensor_pose_tf` while retaining upstream defaults.

## Published data

For the Jabalí launch command above:

| Interface | Meaning |
| --- | --- |
| `/wildboar4/imu` | Orientation, angular velocity (rad/s), acceleration (m/s²), as converted by MRPT's ROS bridge |
| `/wildboar4/imu/mag` | Magnetic field in tesla, expressed in `imu_link` axes |
| `/diagnostics` | Sensor availability, observation rate, stale data and errors |
| `/tf_static` | Configured fixed transform `box` → `imu_link`, unless disabled |

The magnetic publisher is created only when all three magnetic components
exist. It converts MRPT gauss to ROS tesla with a factor of `1e-4`; covariance
is zero (unknown). Both messages use the observation timestamp and sensor
frame. Mounting TF describes sensor placement; measurements remain in sensor
axes. This driver performs no additional magnetic calibration or declination
correction. Sensor orientation conventions and mounting must be checked before
using heading for localization. Publishers use `rclcpp::SystemDefaultsQoS()`.

```bash
ros2 topic echo /wildboar4/imu --once
ros2 topic echo /wildboar4/imu/mag --once
ros2 topic hz /wildboar4/imu
ros2 topic info /wildboar4/imu/mag --verbose
ros2 topic echo /diagnostics --once
```

Optional rawlog recording is enabled with `out_rawlog_prefix:=/path/to/run`.
An optional serialized MRPT observation topic can be enabled with
`publish_mrpt_obs_topic:=imu_mrpt`. These are disabled by default in the launch.

The node also accepts ROS parameters `diag_startup_timeout` (30 s),
`diag_expected_rate` (1 Hz) and `retry_on_error_delay` (5 s). The expected rate
is a diagnostic threshold, not the configured sensor output frequency.
`process_rate` (default 500 Hz) controls polling, not the device output rate.
For parameters not exposed as launch arguments, use the executable directly:

```bash
ros2 run mrpt_sensor_imu_taobotics mrpt_sensor_imu_taobotics_node --ros-args \
  -r __ns:=/wildboar4 \
  -p serial_port:=/dev/serial/by-id/usb-Silicon_Labs_HandsFree_IMU_USB_to_UART_Bridge_Controller_0001-if00-port0 \
  -p publish_topic:=imu -p sensor_frame_id:=imu_link -p robot_frame_id:=box \
  -p diag_expected_rate:=100.0
```

Use an expected rate measured on the actual sensor; 100 Hz above is an example.

## Validation

From the workspace root, with its environment sourced:

```bash
colcon list
colcon test --packages-select mrpt_sensorlib mrpt_sensor_imu_taobotics
colcon test-result --verbose
ros2 launch mrpt_sensor_imu_taobotics mrpt_sensor_imu_taobotics.launch.py --show-args
```

`colcon list` should show exactly the two packages if this is a dedicated
workspace. The inherited automated tests validate package XML; they do not
validate physical sensor behavior. Local validation covered compilation of
both packages, the XML checks, launch-argument loading and a missing-port
smoke test. Read-only robot inspection checked messages from the existing
driver, not deployment of this cleaned branch. Hardware calibration,
extrinsics and localization performance still require physical validation.

## Troubleshooting

- **Serial open error:** check the stable by-id path, `dialout` permissions and
  whether another driver or `sensors.service` owns the device. The node reports
  errors through diagnostics and retries after `retry_on_error_delay`.
- **No magnetic topic:** check parser/model and MRPT version. The topic appears
  only after a complete magnetic observation; the ZED topic is a different sensor.
- **Empty ROS graph:** source the workspace and match the robot's domain and
  DDS configuration. The inspected service used domain 104 and the Fast DDS
  profile `/etc/wildboar/fastdds-udp-only.xml` on the robot.
- **Wrong heading:** check magnetic calibration, interference, sensor axes,
  mounting TF and consumer topic remaps. Do not reuse ZED calibration for the
  external IMU without measuring it.
- **Unexpected executable/version:** check
  `ros2 pkg prefix mrpt_sensor_imu_taobotics`; source the intended overlay after
  `/opt/ros/humble/setup.bash` and avoid duplicate source packages.
