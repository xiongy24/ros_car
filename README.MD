### 1. 硬件准备

**硬件清单：**

- 树莓派 4B（4GB 或 8GB RAM）
- MicroSD 卡（推荐 32GB 以上）
- 树莓派电源适配器
- 电机驱动板（如 L298N）
- 两个或四个编码电机（取决于小车设计）
- 深度摄像头或激光雷达（可选）
- 车轮系统（如麦克纳姆轮）
- 12V 可充电电池（为电机供电）
- 必要的连线、连接器和机械底盘

确保您将所有硬件正确连接，电机驱动与树莓派的GPIO引脚连接，电机驱动板与电池相连提供动力，摄像头、激光雷达等连接到树莓派USB端口。

### 2. 树莓派设置

#### 2.1 下载并安装Ubuntu 22.04

1. **获取Ubuntu 22.04**：

   - 前往 [Ubuntu Raspberry Pi 下载页面](https://ubuntu.com/download/raspberry-pi)，选择 `Ubuntu 22.04 LTS Server`。
2. **烧录镜像到SD卡**：

   - 使用 `Raspberry Pi Imager` 或 `balenaEtcher` 将 Ubuntu 22.04 的镜像烧录到MicroSD卡中。
3. **启动树莓派并进行初步配置**：

   - 将烧录好的SD卡插入树莓派，连接显示器和键盘。
   - 启动后完成初步配置，选择网络、设置用户名和密码。

#### 2.2 设置SSH远程连接

完成树莓派的初步配置后，建议通过SSH进行远程控制：

1. **启用SSH**：

   - 在树莓派终端中，启用SSH服务：
     ```bash
     sudo systemctl enable ssh
     sudo systemctl start ssh
     ```
2. **通过SSH连接树莓派**：

   - 获取树莓派的IP地址：
     ```bash
     hostname -I
     ```
   - 在PC上，打开终端并使用SSH连接树莓派：
     ```bash
     ssh 用户名@树莓派IP地址
     ```

### 3. 在树莓派上安装ROS 2 Humble

#### 3.1 更新并安装必要依赖

首先更新系统并安装所需的依赖包：

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl gnupg2 lsb-release
```

#### 3.2 添加ROS 2 Humble源

1. **添加ROS 2的GPG密钥**：

   ```bash
   sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.asc | sudo apt-key add -
   ```
2. **添加ROS 2软件源**：

   ```bash
   sudo sh -c 'echo "deb http://packages.ros.org/ros2/ubuntu $(lsb_release -cs) main" > /etc/apt/sources.list.d/ros2-latest.list'
   sudo apt update
   ```

#### 3.3 安装ROS 2 Humble

由于树莓派资源有限，建议安装基础版`ros-base`，该版本不包含图形界面。

```bash
sudo apt install ros-humble-ros-base
```

#### 3.4 配置环境变量

每次使用ROS 2时都需要加载环境。为简化操作，修改`.bashrc`文件以自动加载ROS环境：

```bash
echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

### 4. 创建ROS 2工作空间

#### 4.1 创建一个新的工作空间

1. **创建工作空间目录**：

   ```bash
   mkdir -p ~/ros2_ws/src
   cd ~/ros2_ws
   ```
2. **编译工作空间**：

   ```bash
   colcon build
   ```
3. **加载工作空间**：
   编译成功后，加载工作空间的环境：

   ```bash
   source install/setup.bash
   ```

将这条命令添加到`.bashrc`文件中，以便每次打开终端时自动加载工作空间：

```bash
echo "source ~/ros2_ws/install/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

### 5. 电机控制设置

#### 5.1 电机驱动板连接树莓派

使用 L298N 电机驱动板与树莓派 GPIO 引脚连接。典型的连接方式如下：

- `ENA` 和 `IN1`、`IN2` 引脚连接到树莓派的 GPIO 引脚，用于控制电机转速和方向。
- `VCC` 和 `GND` 连接到电池，提供电力。
- `OUT1` 和 `OUT2` 引脚连接到电机。

#### 5.2 编写ROS 2电机控制节点

1. **创建一个新的Python包**：

   - 进入 `src` 目录，并使用`ros2 pkg create`创建一个新的包：
     ```bash
     cd ~/ros2_ws/src
     ros2 pkg create --build-type ament_python motor_control
     ```
2. **编辑`setup.py`文件**：
   在 `motor_control` 目录下找到`setup.py`，将其中的`entry_points`部分更新为：

   ```python
   entry_points={
       'console_scripts': [
           'motor_controller = motor_control.motor_controller:main',
       ],
   },
   ```
3. **编写电机控制代码**：
   在 `motor_control/motor_control/` 目录中创建 `motor_controller.py` 文件：

   ```python
   import rclpy
   from rclpy.node import Node
   from geometry_msgs.msg import Twist
   import RPi.GPIO as GPIO

   class MotorController(Node):
       def __init__(self):
           super().__init__('motor_controller')
           self.subscription = self.create_subscription(Twist, 'cmd_vel', self.listener_callback, 10)
           self.subscription  # prevent unused variable warning

           # 设置GPIO引脚
           GPIO.setmode(GPIO.BCM)
           self.motor_pin1 = 18  # 连接到电机驱动IN1的GPIO引脚
           self.motor_pin2 = 23  # 连接到电机驱动IN2的GPIO引脚
           GPIO.setup(self.motor_pin1, GPIO.OUT)
           GPIO.setup(self.motor_pin2, GPIO.OUT)

       def listener_callback(self, msg):
           linear_speed = msg.linear.x
           if linear_speed > 0:
               GPIO.output(self.motor_pin1, GPIO.HIGH)
               GPIO.output(self.motor_pin2, GPIO.LOW)  # 前进
           elif linear_speed < 0:
               GPIO.output(self.motor_pin1, GPIO.LOW)
               GPIO.output(self.motor_pin2, GPIO.HIGH)  # 后退
           else:
               GPIO.output(self.motor_pin1, GPIO.LOW)
               GPIO.output(self.motor_pin2, GPIO.LOW)  # 停止

   def main(args=None):
       rclpy.init(args=args)
       motor_controller = MotorController()
       rclpy.spin(motor_controller)
       motor_controller.destroy_node()
       rclpy.shutdown()

   if __name__ == '__main__':
       main()
   ```

#### 5.3 编译代码

返回工作空间目录，编译ROS 2包：

```bash
cd ~/ros2_ws
colcon build
source install/setup.bash
```

### 6. 在PC上发布运动指令

#### 6.1 安装ROS 2 Humble并配置网络

在PC上安装ROS 2 Humble，并确保PC和树莓派处于同一局域网中。为确保PC和树莓派能够通信，您可以设置相同的`ROS_DOMAIN_ID`：

```bash
export ROS_DOMAIN_ID=0
echo "export ROS_DOMAIN_ID=0" >> ~/.bashrc
source ~/.bashrc
```

#### 6.2 发布控制命令

在PC上，使用键盘发布控制命令：

```bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

使用箭头键控制小车的前进、后退和转向。这些命令将发布到 `/cmd_vel` 话题，树莓派将接收并根据消息控制电机。

### 7. 测试与调试

- **电机测试**：通过PC发布`/cmd_vel`消息，检查电机是否按预期转动。
- **调试GPIO控制**：确保GPIO的引脚配置正确，调整电机的转速和方向。
- **监控消息流**：使用`ros2 topic echo /cmd_vel`监控树莓派上收到的消息，确保通信正常。

### 8. 扩展：传感器集成与RViz监控

如果您想集成传感器如激光雷达或摄像头，可以通过编写额外的ROS 2节点来处理传

感器数据。通过PC上的`RViz`，可以可视化传感器数据并进行导航等更高级别的控制。
