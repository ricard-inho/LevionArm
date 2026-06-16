# LevionArm

This repository contains required packages to control "Levion Arm", which is equipped on the Floating Platform for Zero-G lab at University of Luxembourg.

![arm_image](docs/image/levion.png)

## Installation

Clone the repo including submodules under the workspace.

```bash
mkdir -p levion_ws/src
cd levion/src
```

```bash
git clone --recurse-submodules -j8 git@github.com:aky-u/LevionArm.git
```

> [!NOTE]
> If you have cloned without submodules, use the following command to clone submodules.
>
> `git submodule update --init --recursive`

Install required packages.

```bash
rosdep install --from-paths src --ignore-src -r -y
```

> [!NOTE]
> If you have'nt setup rosdep, initialize it with the command below.
>
> ```bash
> sudo rosdep init
> rosdep update
> ```

## Build

```bash
cd .. # move to workspace
colcon build --symlink-install
```

## Hardware drivers

Two ros2_control hardware interface plugins are available and can be selected at launch time via the `hw_plugin` argument.

### Servo mode (default): `cubemars_hardware/CubeMarsSystemHardware`

Uses the **servo/VESC CAN protocol** (extended 29-bit frames). Supports independent control modes per joint: current, velocity, position, and position+velocity. This is the original driver.

### MIT mode: `cubemars_mit_hardware/MITHardwareInterface`

Uses the **MIT Cheetah CAN protocol** (standard 11-bit frames). Sends `[p_des, v_des, kp, kd, t_ff]` directly to the motor's on-board PD loop every control cycle. The motor runs its own high-rate PD controller, which gives better impedance tracking and reduced shaking compared to software-side PD.

Drivers vendored from [leggedrobotics/cubemars-ak-c-drivers](https://github.com/leggedrobotics/cubemars-ak-c-drivers). The repo is cloned at `src/cubemars-ak-c-drivers/` and includes a standalone `cubemars_controller` node and `cubemars_msgs` package for direct CAN testing outside of ros2_control.

| Parameter | Servo mode | MIT mode |
|-----------|-----------|---------|
| CAN frame type | Extended (EFF, 29-bit) | Standard (SFF, 11-bit) |
| Control law | On PC | On motor (kp/kd sent each cycle) |
| Command interfaces | position / velocity / effort | position (p_des) / velocity (v_des) / effort (t_ff) |
| Extra joint params | `pole_pairs`, `gear_ratio`, `kt`, `enc_off` | `kp`, `kd`, `motor_model` |

#### MIT mode joint defaults (in `ak80_8_mit.macros.xacro`)

- `kp = 0.5` Nm/rad
- `kd = 0.03` Nm·s/rad
- `motor_model = AK80-8` (v_max = 37.5 rad/s, t_max = 32 Nm)

To change gains without recompiling, override at launch:

```bash
ros2 launch levion_arm_ros2_control levion_arm.launch.py hw_plugin:=mit
```

Or edit the defaults directly in `levion_arm_ros2_control/description/ros2_control/ak80_8_mit.macros.xacro`.

#### Switching between drivers

Pass `hw_plugin` to any launch file that exposes the `levion_system` xacro macro:

```bash
hw_plugin:=real    # servo mode (default)
hw_plugin:=mit     # MIT mode
hw_plugin:=mujoco  # MuJoCo simulation
hw_plugin:=gazebo  # Gazebo simulation
```

## Setup CAN with Holybro

```bash
sudo modprobe mttcan
sudo ip link set can0 type can bitrate 1000000
sudo ip link set can0 up
```

## Set zero position

### Servo mode

```bash
cansend can0 00000568#01 # 104
cansend can0 00000569#01 # 105
cansend can0 000005CC#01 # 204
cansend can0 000005CD#01 # 205
```

### MIT mode

In MIT mode the zero must be set while the motor is in MIT mode. Use the `set_zero` service after launching the MIT controller:

```bash
ros2 service call /cubemars_controller_node/set_zero cubemars_msgs/srv/SetZero "{motor_id: 104}"
```

Or use the `generate_mit_set_zero_message` frame directly:

```bash
# Enter MIT first (standard frame, ID = motor_id)
cansend can0 068#FFFFFFFFFFFFFFFF FC  # motor 104: enter MIT
cansend can0 068#FFFFFFFFFFFFFFFF FE  # motor 104: set zero
cansend can0 068#FFFFFFFFFFFFFFFF FD  # motor 104: exit MIT
```

## Run single motor test

```bash
source install/setup.bash
ros2 launch levion_arm_ros2_control ak80_8..launch.py # launch single motor controller with default type = position
```

```bash
source install/setup.bash
ros2 launch levion_arm_ros2_control ak80_8..launch.py controller_type:=forward_velocity_controller # launch velocity controller
```

```bash
source install/setup.bash
ros2 launch levion_arm_ros2_control ak80_8..launch.py controller_type:=forward_effort_controller # launch effort controller
```

### Launch the arm

```bash
source install/setup.bash
ros2 launch levion_arm_ros2_control levion_arm.launch.py # launch single motor controller with default type = position
```

```bash
source install/setup.bash
ros2 launch levion_arm_ros2_control levion_arm.launch.py controller_type:=forward_velocity_controller # launch velocity controller
```

```bash
source install/setup.bash
ros2 launch levion_arm_ros2_control levion_arm.launch.py controller_type:=forward_effort_controller # launch effort controller
```

## ID map

BN:1088221109 = 104 {0x68} : Right shoulder

BN:1088230418 = 105 {0x69} : Left elbow

BN:1088230509 = 204 {0xCC} : Left shoulder

BN:1088230509 = 205 {0xCD} : Right elbow

## AK Series

[Cubemars support](https://www.cubemars.com/article.php?id=261)

- [AK80-8](https://www.cubemars.com/goods-1151-AK80-8.html)

## MIT mode — impedance control workflow

The MIT interface is designed for use with software impedance controllers (e.g. `single_motor_test_impedance` in `pingu_cubo_low_level_controller`). The motor's on-board PD loop handles the fast inner loop; the ROS node handles the outer impedance computation.

```bash
# 1. Launch stack with MIT hardware interface
ros2 launch pingu_cubo_low_level_controller \
    pingu_low_level_controller_individual_thruster_control_launch.py \
    arm_controllers:="left_arm_shoulder_effort_controller" \
    hw_plugin:=mit

# 2. Activate the effort controller
ros2 control switch_controllers --activate left_arm_shoulder_effort_controller

# 3. Run the impedance node (callback-driven, runs at joint_states rate)
ros2 run pingu_cubo_low_level_controller single_motor_test_impedance \
    --ros-args -p joint:=left_shoulder_joint -p kp:=0.5 -p kd:=0.03
```

With MIT mode the `kp`/`kd` in the xacro macros define the **hardware gains** sent to the motor each cycle. The `kp`/`kd` in the impedance node compute a feedforward torque on top. For pure hardware impedance set the node's `kp`/`kd` to 0 and rely solely on the xacro values.

## Related works

- <https://github.com/leggedrobotics/cubemars-ak-c-drivers> — RSL MIT mode C drivers (vendored in `cubemars_mit_hardware`)

- <https://github.com/neurobionics/TMotorCANControl>

- <https://github.com/dfki-ric-underactuated-lab/mini-cheetah-tmotor-python-can>

- <https://github.com/SherbyRobotics/tmotor_ros>

- <https://github.com/OpenFieldAutomation-OFA/cubemars_hardware/tree/main>

## Trouble shooting

- [If Motor Failed Entering Both Modes](https://www.cubemars.com/article-330-If+Motor+Failed+Entering+Both+Modes.html)

<!-- > [!WARNING]
> -->
