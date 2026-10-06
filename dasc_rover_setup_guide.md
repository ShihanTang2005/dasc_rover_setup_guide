# DASC Rover Hardware and Software Setup Guide
Note: This is the setup guide for rovers that uses Raspberry Pi. For rovers that use Jetson Orin, check this guide: https://dasc-lab.github.io/robot-framework/px4_robots/px4_rover_setup.html.

## 0. System Overview & Architecture
The system operates as a closed-loop autonomous ground vehicle consisting of three main layers:
* **High-Level Control (Host Laptop & Docker):** Runs ROS 2, Vicon pose subscriber, EKF filter, target velocity publisher, PID velocity controller and Control Barrier Function (CBF-QP) safety filter.
* **Middle-Level Communication (MicroXRCE-DDS Agent on Raspi):** Relays Offboard control topics and actuator inputs between ROS 2 and PX4 autopilot via serial interface.
* **Low-Level Execution (PX4 Autopilot & RoboClaw):** Decodes `/fmu/in/actuator_motors`, outputs PWM control signals to RoboClaw motor drivers, and drives differential motors.

## 1. Docker environment setup
### PC Docker setup:

---

## 2. PX4 Firmware Build & Flashing

### Target & Airframe Setup
1. Recursively clone the custom PX4 repository compatible with CubeOrange hardware. Use this repo: https://github.com/ShihanTang2005/PX4-Autopilot-Quad. After cloning, enter the directory and run
   ```bash
   git fetch /home/px4 'refs/tags/*:refs/tags/*'
   ```
   Then check by running
   ```bash
   git describe --exclude 'ext/*' --always --tags
   ```
   It should output
   ```bash
   v1.14.0-beta2-77-gfc326c0446
   ```
2. Build the target firmware in a Docker environment:
   ```bash
   make cubepilot_cubeorange
   ```
3. Flash `cubepilot_cubeorange_default.px4` onto the Orange Cube via QGroundControl (QGC).
4. Set vehicle autostart parameter for Aion Robotics R1 Rover:
   * `SYS_AUTOSTART = 50003` (Ensure `50003_aion_robotics_r1_rover` script is included during build).

### Custom DDS Topics
If you want to customize DDS topics, modify `src/modules/microdds_client/dds_topics.yaml` to enable more functions you like:
The yaml file is in this pattern:
```yaml
subscriptions:
  - topic: /fmu/in/offboard_control_mode
    type: px4_msgs::msg::OffboardControlMode
  - topic: /fmu/in/actuator_motors
    type: px4_msgs::msg::ActuatorMotors
```
After modification, run
```bash
make cubepilot_cubeorange
```
again to build the firmwire. Flash this firmwire onto the PX4 Orange Cube.

---

## 3. QGC setup for PX4

Configure PX4 serial parameters via QGC MAVLink Console or Parameter tab:
* **`XRCE_DDS_CFG`**: Set to `TELEM1`.
* **`SER_TEL1_BAUD`**: Set to `921600`. I remembered that with a baudrate below 115200 the communication will fail. 
* **`XRCE_DDS_DOM_ID`**: Set to `4` (or your group's domain ID).
* **Battery Configuration:** Update battery cell count in PX4 parameters (`3S` battery setup) to avoid false low-battery safety triggers. Also, calibrate the actual voltage of the battery in Parameter Tab or the command tab on the top in QGC.

## 4. Raspi Env Setup
Setup the docker environment by following this repo:https://github.com/dasc-lab/robot-jumpstart

The repo did not contain the Micro-XRCE-DDS-agent in it. After we set up the docker environment, run
```bash
docker ps -a
``` 
to check if the **robot-jumpstart-px4-1** docker env has been created.
Then run
```bash
docker start robot-jumpstart-px4-1
docker exec -it robot-jumpstart-px4-1 bash
```
to enter docker env. Before installing the missing part, we need to upgrade Cmake of python3.
```bash
apt-get update
apt-get install -y git cmake build-essential
apt-get install -y python3-pip
python3 -m pip install --upgrade 'cmake==3.28.4'
hash -r
cmake --version
```
The last line should output a version higher than 3.20.

Then run the following bash lines to install Micro-XRCE-DDS-agent:
```bash
cd /root
git clone --branch v2.4.3 --depth 1 \
  https://github.com/eProsima/Micro-XRCE-DDS-Agent.git

cd Micro-XRCE-DDS-Agent
mkdir -p build
cd build
cmake ..
make -j2
make install
ldconfig
```
Check if it has been successfully installed:
```bash
MicroXRCEAgent --help
```

Verify that the **msg** directory  in Raspi is the same as PX4 firmware, otherwise the Raspi can't communicate with Cube Orange. To be specific, the PX4 will continously report **offboard_control_signal_lost: True** in the failsafe flag.

If not, 
```bash
cd /root/px4_ros_com_ros2/src
rm -r px4_msgs/
git clone https://github.com/kalebbennaveed/px4_msgs.git
cd /root/px4_ros_com_ros2/src/px4_msgs
git checkout d839e2fede12fd99b585c4b9c517572dcd92478f
```
to replace the whole **px4_msg** directory. 

Add the corresponding **OffboardControlMode.msg** and **ActuatorMotors.msg** to the CMakelist in px4_msg directory on Raspi. 
```bash
nano /root/px4_ros_com_ros2/src/px4_msgs/CMakeLists.txt 
```
Add the two msg into the **set()**.
This is because we need them to control the motor through PX4. Those two files correspond to the dds_topic.


```bash
colcon build --symlink-install --packages-select px4_msgs
source install/setup.bash
```
After replacing the original directory, run the following lines to test if it is successful:
```python
>>> from px4_msgs.msg import TrajectorySetpoint
>>> msg = TrajectorySetpoint()
>>> print(msg)
px4_msgs.msg.TrajectorySetpoint(timestamp=0, position=array([0., 0., 0.], dtype=float32), velocity=array([0., 0., 0.], dtype=float32), acceleration=array([0., 0., 0.], dtype=float32), jerk=array([0., 0., 0.], dtype=float32), yaw=0.0, yawspeed=0.0, raw_mode=False, cmd=array([0., 0., 0., 0.], dtype=float32))
>>> hasattr(msg,"velocity")
True
>>> hasattr(msg,"vx")
False
>>> 
```

Then 
```bash
rm -rf build/px4_msgs install/px4_msgs #be careful about your path!
colcon build --symlink-install --packages-select px4_msgs
source install/setup.bash
```


Launch the Micro-XRCE-DDS-Agent on Raspberry Pi (This is covered also in https://github.com/ShihanTang2005/RaspiFile_for_DascLab_Rover_Socialnav_Demo/blob/main/README.md):
```bash
MicroXRCEAgent serial --dev /dev/ttyS0 -b 921600
```

Verify available ROS 2 topics:
```bash
ros2 topic list -t
```

---

## 5. Hardware Wiring & Motor Driver Setup

### Pin Wiring
* **PX4 Main Signals:** Connect Main Output 1 to RoboClaw S1, Main Output 2 to RoboClaw S2, and establish a common GND connection between PX4 and RoboClaw.
* **Roboclaw and Motor Wiring:** Connect the 2 pairs of yellow and green wires of the motors to EN1 & EN2. This is for the Roboclaw to read the encoder values. Connect the 2 pairs of red and black wires on the same strand to the 2 [+ -]. Note connect red to + and black to -. 


### RoboClaw Setup (PWM Mode)
Download Motion Studio software on Windows first. Open this link https://www.basicmicro.com/downloads?srsltid=AfmBOopMl_-A8dDWMYJk_x3SBqipTUnb1lpvnhUB3ywTDFqQZ7ji0QUK and download Motion Studio (legacy). From my experience, the new version does not support our hardware. 
1. Open Motion Studio on PC and change **Control Mode** to **RC**.
2. Check MCU options to set the first received PWM signal value as the neutral zero-speed point. In this case, the neutral point is 1500us.
3. You can try auto-calibrate PID parameters for wheel motors.
4. Remember to click "Save Settings" after any changes.

### QGC actuator testing
Open QGroundControl and enter **Actuator Test**. Click **MAIN OUT** and enter your desired PWM parameters. I would recommend MIN as 1000, MAX as 2000, and the Neutral zero-speed point as 1500. Set the sources to the corresponding hardware MAIN number. Lift the rover so that all four wheels are off the ground, then move the virtual joystick to test the wheel motion.

---

## 5. Raspi Control
Follow the instructions in https://github.com/ShihanTang2005/RaspiFile_for_DascLab_Rover_Socialnav_Demo/blob/main/README.md. 

---

## 6. High-Level Controller & CBF-QP Safety Filter

### Velocity Limit Calibration
Physical limits calibrated from ground experiments:
* Maximum Linear Velocity ($v_{max}$): `1.26 m/s`
* Maximum Angular Velocity ($\omega_{max}$): `2.96 rad/s`

### Control Pipeline
1. **Nominal Controller:** Calculates target headway, heading error ($\Delta \theta$), and distance error to goal.
2. **Unicycle CBF-QP Filter:** Solves real-time QP constraints around obstacles to yield safe velocity output $(v, \omega)$.
3. **Actuator Mapping (`cmd_vel_to_actuator_motors`):** Normalizes velocities and commands individual wheel motors via `/fmu/in/actuator_motors`:
   ```python
   left_motor  = normalized_v - normalized_w
   right_motor = normalized_v + normalized_w
   ```
