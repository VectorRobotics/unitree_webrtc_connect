# unitree_webrtc_ros

ROS 2 Jazzy package for controlling Unitree Go2 robot via WebRTC connection. Provides a simple interface for movement control using `SPORT_CMD["Move"]` and sport mode commands.

## Features

- **TwistStamped subscriber**: Control robot movement using `geometry_msgs/TwistStamped` messages
- **Sport mode services**: Execute predefined movements (standup, liedown, hello, stretch, recovery_stand)
- **Direct command forwarding**: Each cmd_vel message sends one SPORT_CMD["Move"] command
- **Clean implementation**: No camera/lidar/tf overhead, just movement control

## Prerequisites

- ROS 2 Jazzy
- Python 3.12
- Unitree Go2 robot

## Installation

### Step 1: Install unitree_webrtc_connect in a Python virtual environment

The `unitree_webrtc_connect` package is NOT available on PyPI. You need to install it from source in a virtual environment:

```bash
# Install system dependencies first
sudo apt update
sudo apt install portaudio19-dev

# Create a virtual environment in your home directory
python3 -m venv ~/unitree_venv

# Activate the virtual environment
source ~/unitree_venv/bin/activate

# Clone the unitree_webrtc_connect repository
cd ~
git clone https://github.com/legion1581/go2_webrtc_connect.git
cd go2_webrtc_connect

# Install it in editable mode (this installs all dependencies too)
pip install -e .

# Verify installation
python -c "import unitree_webrtc_connect; print('Installation successful!')"

# Deactivate when done
deactivate
```

**Important Notes:**
- The package is called `go2_webrtc_connect` in the repository but imports as `unitree_webrtc_connect`
- Installing with `-e .` makes it editable, so updates from git pull will be reflected immediately
- The venv will be at `~/unitree_venv` and the source code at `~/go2_webrtc_connect`

### Step 2: Set up your ROS 2 workspace

```bash
# Create workspace if you don't have one
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws/src

# Copy this package to your workspace
cp -r /path/to/unitree_webrtc_ros .
```

### Step 3: Build the package

```bash
cd ~/ros2_ws

# Source ROS 2
source /opt/ros/jazzy/setup.bash

# Build the package
colcon build --packages-select unitree_webrtc_ros

# Source the workspace
source install/setup.bash
```

## Usage

### Activating the environment

Every time you want to use this package, you need to:

1. **Activate the Python virtual environment** (in one terminal or add to your bashrc):
```bash
source ~/unitree_venv/bin/activate
```

2. **Source ROS 2 and your workspace**:
```bash
source /opt/ros/jazzy/setup.bash
source ~/ros2_ws/install/setup.bash
```

**Tip**: Add this to your `~/.bashrc` for convenience:
```bash
# Add to ~/.bashrc
alias ros_unitree='source ~/unitree_venv/bin/activate && source /opt/ros/jazzy/setup.bash && source ~/ros2_ws/install/setup.bash'
```

Then you can just run `ros_unitree` to set everything up.

### Launch the node

Basic launch with default IP (192.168.8.181):
```bash
ros2 launch unitree_webrtc_ros unitree_control.launch.py
```

Launch with custom IP:
```bash
ros2 launch unitree_webrtc_ros unitree_control.launch.py robot_ip:=192.168.8.100
```

Launch with LocalAP connection (robot's own WiFi at 192.168.12.1):
```bash
ros2 launch unitree_webrtc_ros unitree_control.launch.py connection_method:=LocalAP
```

### Control the robot

#### Using cmd_vel topic

Send velocity commands (note: this package uses TwistStamped):
```bash
# Move forward
ros2 topic pub /cmd_vel geometry_msgs/msg/TwistStamped "{twist: {linear: {x: 0.3, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 0.0}}}"

```

#### Using services

Execute sport mode commands:
```bash
# Stand up
ros2 service call /standup std_srvs/srv/Trigger

# Lie down
ros2 service call /liedown std_srvs/srv/Trigger

# Wave hello
ros2 service call /hello std_srvs/srv/Trigger

# Stretch
ros2 service call /stretch std_srvs/srv/Trigger

# Recovery stand position
ros2 service call /recovery_stand std_srvs/srv/Trigger
```

## Configuration

Edit [config/unitree_params.yaml](config/unitree_params.yaml) to change default parameters:

```yaml
unitree_control:
  ros__parameters:
    robot_ip: "192.168.8.181"
    connection_method: "LocalSTA"  # Options: LocalAP, LocalSTA, Remote
```

## Topics

### Subscribed Topics

- `/cmd_vel` (geometry_msgs/TwistStamped): Velocity commands for robot movement
  - `twist.linear.x`: Forward/backward velocity (m/s)
  - `twist.linear.y`: Left/right velocity (m/s)
  - `twist.angular.z`: Rotation velocity (rad/s)

## Services

- `/standup` (std_srvs/Trigger): Make robot stand up
- `/liedown` (std_srvs/Trigger): Make robot lie down
- `/hello` (std_srvs/Trigger): Make robot wave hello
- `/stretch` (std_srvs/Trigger): Make robot stretch
- `/recovery_stand` (std_srvs/Trigger): Reset to recovery stand position

## Parameters

- `robot_ip` (string, default: "192.168.8.181"): IP address of the robot
- `connection_method` (string, default: "LocalSTA"): Connection method (LocalAP/LocalSTA/Remote)

## Connection Methods

- **LocalSTA**: Connect to robot on your local network (default, requires robot IP)
- **LocalAP**: Connect directly to robot's WiFi access point (IP: 192.168.12.1)
- **Remote**: Connect via Unitree cloud service (requires authentication - not implemented in launch file)

## Implementation Details

- **Movement**: Uses `SPORT_CMD["Move"]` from unitree_webrtc_connect
- **Command forwarding**: Each TwistStamped message triggers one move command
- **Thread-safe async**: Background thread runs asyncio event loop for WebRTC connection
- **No auto-stop**: Robot continues last command until new one received (send zeros to stop)

## Troubleshooting

### ModuleNotFoundError: No module named 'unitree_webrtc_connect'

Make sure you activated the virtual environment:
```bash
source ~/unitree_venv/bin/activate
```

### Connection fails

1. **Check robot IP**: Verify with `ping <robot_ip>`
2. **Check robot is on**: LED should be lit
3. **Check WiFi connection**:
   - For LocalSTA: Robot and computer on same network
   - For LocalAP: Computer connected to robot's WiFi (Unitree_XXXXXX)
4. **Check firewall**: WebRTC needs UDP ports open

### Robot doesn't move

1. **Check robot mode**: Must be in sport/normal mode (not AI mode which is deprecated)
2. **Check messages**: `ros2 topic echo /cmd_vel`
3. **Check logs**: Look for errors in node output
4. **Try manual command**: Use `ros2 topic pub` to test

### Virtual environment issues

If ROS 2 can't find packages after activating venv:
```bash
# Make sure to activate venv BEFORE sourcing ROS 2
source ~/unitree_venv/bin/activate
source /opt/ros/jazzy/setup.bash
source ~/ros2_ws/install/setup.bash
```

## Example: Using with teleop

If you have a teleop node that publishes `TwistStamped`:
```bash
# Terminal 1: Launch unitree control
source ~/unitree_venv/bin/activate
source /opt/ros/jazzy/setup.bash
source ~/ros2_ws/install/setup.bash
ros2 launch unitree_webrtc_ros unitree_control.launch.py

# Terminal 2: Run your teleop
ros2 run <your_package> <teleop_node>
```

## License

Apache-2.0

## Notes

- This package does NOT auto-stop the robot. You must send zero velocities to stop.
- The robot uses its own coordinate frame: x=forward, y=left, z=yaw
- WebRTC connection requires good WiFi signal strength
- First connection may take up to 30 seconds
