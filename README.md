# Taobotics IMU drivers for Wildboar

ROS 2 drivers for the Taobotics/HandsFree serial IMU. The retained packages
publish inertial data and, when the selected sensor model supplies all three
magnetic axes, a separate magnetic-field message. ROS 2 Humble has been
verified for the Wildboar deployment.

## Packages

| Package | Purpose |
| --- | --- |
| `mrpt_sensor_imu_taobotics` | Serial IMU node and configurable launch file |
| `mrpt_sensorlib` | MRPT observation conversion, IMU and magnetometer publishers, static TF, diagnostics, retry handling, and optional rawlog recording |

This fork retains the Taobotics IMU driver and its support library from the
upstream `mrpt_sensors` repository. Other upstream sensor drivers and
maintenance tools are not included. The package and executable names are
compatible with Wildboar sensor bringup. Existing changelogs retain upstream
history and may mention packages that are no longer present.

## Verified HFI-A9 configuration

A Wildboar deployment using the HFI-A9 parser was verified with ROS 2 Humble.
In that configuration, the IMU and magnetic-field topics each published at
approximately 166.6 Hz. Magnetic-field publishing was enabled when all three
magnetic components were present in the MRPT observation. Rates can depend on
the device configuration; measure them on the target robot. Use the actual
device path, namespace, and frame IDs for each robot.

| Setting | Value or convention |
| --- | --- |
| Sensor model | `hfi-a9` |
| IMU topic | `/<namespace>/imu` (`sensor_msgs/msg/Imu`) |
| Magnetic-field topic | `/<namespace>/imu/mag` (`sensor_msgs/msg/MagneticField`) |
| Observed HFI-A9 IMU and magnetometer rate | Approximately 166.6 Hz |
| Example sensor frame | `imu_link` |
| Example parent frame | `box` |

The `hfi-b6` model is also supported by the upstream driver, but its documented
protocol has no magnetometer fields, so it does not provide a magnetic topic.

## Build

Use a ROS 2 Humble workspace with `colcon` and `rosdep` configured. Add this
repository under `src`, install dependencies, and build the two packages:

```bash
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws/src
git clone --branch main git@github.com:Innovations-in-Smart-and-Secure-Systems/mrpt_sensors.git
cd ~/ros2_ws
source /opt/ros/humble/setup.bash
rosdep install --from-paths src --ignore-src -r -y
colcon build --symlink-install --packages-up-to mrpt_sensor_imu_taobotics
source install/setup.bash
```

Dependencies include MRPT hwdrivers, `mrpt_libros_bridge`, `mrpt_msgs`,
`rclcpp`, TF2, `sensor_msgs`, `diagnostic_updater`, `launch`, and `launch_ros`.
The verified build used MRPT 3.5.0 hwdrivers and `mrpt_libros_bridge` 4.0.0.
The user running the node needs serial-device access, typically through the
`dialout` group.

## Launch

The Wildboar bringup normally owns the serial device. Do not start a second
node against the same device while bringup is running. For a standalone
maintenance session, pass the robot's namespace, serial path, model, and frame
IDs explicitly:

```bash
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash
ROBOT_NAMESPACE=robot_name
IMU_SERIAL_DEVICE=/dev/serial/by-id/replace-with-the-device-id
ros2 launch mrpt_sensor_imu_taobotics mrpt_sensor_imu_taobotics.launch.py \
  namespace:="$ROBOT_NAMESPACE" \
  serial_port:="$IMU_SERIAL_DEVICE" \
  sensor_model:=hfi-a9 \
  publish_topic:=imu \
  sensor_frame_id:=imu_link \
  robot_frame_id:=box
```

The example uses zero mounting pose. Set `sensor_pose_x/y/z` in metres and
`sensor_pose_yaw/pitch/roll` in degrees after measuring the mount. If the URDF
already publishes this transform, set `publish_sensor_pose_tf:=false` to avoid
duplicate TF authorities. Without overrides, the launch uses namespace
`taobotics`, topic `sensor`, frames `base_link` to `imu`, and parser `hfi-a9`;
`serial_port` is required.

```bash
ros2 launch mrpt_sensor_imu_taobotics mrpt_sensor_imu_taobotics.launch.py --show-args
```

## Topics and data

With `namespace:=<robot_namespace>` and `publish_topic:=imu`, the main topics
are:

| Topic | Type | Contents |
| --- | --- | --- |
| `/<namespace>/imu` | `sensor_msgs/msg/Imu` | Orientation, angular velocity (rad/s), and acceleration (m/s²), converted by MRPT's ROS bridge |
| `/<namespace>/imu/mag` | `sensor_msgs/msg/MagneticField` | Magnetic field in tesla, expressed in the sensor frame |
| `/diagnostics` | `diagnostic_msgs/msg/DiagnosticArray` | Sensor availability, observation rate, stale data, and errors |
| `/tf_static` | `tf2_msgs/msg/TFMessage` | Configured fixed transform, unless disabled |

The magnetic publisher is created only when all three magnetic components are
present. It converts MRPT gauss to ROS tesla with a factor of `1e-4`; covariance
is zero (unknown). Both messages use the observation timestamp and sensor
frame. The driver does not apply magnetic calibration or declination
correction. Check sensor orientation conventions, calibration, and mounting
before using the field for heading. Publishers use
`rclcpp::SystemDefaultsQoS()`.

```bash
ros2 topic echo /<namespace>/imu --once
ros2 topic echo /<namespace>/imu/mag --once
ros2 topic hz /<namespace>/imu
ros2 topic hz /<namespace>/imu/mag
ros2 topic info /<namespace>/imu/mag --verbose
ros2 topic echo /diagnostics --once
```

Optional rawlog recording is enabled with `out_rawlog_prefix:=/path/to/run`.
An optional serialized MRPT observation topic can be enabled with
`publish_mrpt_obs_topic:=imu_mrpt`. Both are disabled by default.

## Parameters

The node accepts `diag_startup_timeout` (30 s), `diag_expected_rate` (1 Hz),
and `retry_on_error_delay` (5 s). The expected rate is a diagnostic threshold,
not the sensor output frequency. `process_rate` (default 500 Hz) controls
polling, not the device output rate. Set `diag_expected_rate` to match the
measured rate for the selected model.

## Validation

From the workspace root, after sourcing its environment:

```bash
colcon list
colcon test --packages-select mrpt_sensorlib mrpt_sensor_imu_taobotics
colcon test-result --verbose
ros2 launch mrpt_sensor_imu_taobotics mrpt_sensor_imu_taobotics.launch.py --show-args
```

In a dedicated workspace, `colcon list` should show the two packages above.
Automated tests validate package metadata and launch arguments; they do not
validate physical sensor behavior. Confirm both IMU and magnetic-field topics
on the target hardware.

## Troubleshooting

- **Serial open error:** check the stable by-id path, `dialout` permissions,
  and whether another driver owns the device. The node reports errors through
  diagnostics and retries after `retry_on_error_delay`.
- **No magnetic topic:** check the selected parser and MRPT version. The topic
  appears only after a complete magnetic observation; not every supported
  sensor model provides magnetic data.
- **Empty ROS graph:** source the intended workspace overlay after the ROS
  installation and use the robot's ROS domain and DDS configuration.
- **Unexpected executable/version:** check `ros2 pkg prefix
  mrpt_sensor_imu_taobotics`, then ensure the intended workspace overlay is
  sourced after `/opt/ros/humble/setup.bash`.
