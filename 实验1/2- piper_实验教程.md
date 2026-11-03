# Piper 机械臂 SDK 与 ROS 环境搭建实验教程

> 适用机型：AgileX（松灵机器人）Piper 机械臂
> 适用系统：Ubuntu 20.04 + ROS Noetic + Python 3.8
> 参考仓库：
> - piper_sdk：<https://github.com/agilexrobotics/piper_sdk>
> - piper_ros：<https://github.com/agilexrobotics/piper_ros>
>
> 本教程全部步骤均在真机上实测通过。实测环境：piper_sdk 0.7.0a1、机械臂固件 S-V1.9-0（V3 协议）、ros-noetic-moveit 1.1.16（apt 版）。

---

## 实验目标

完成本实验后，你将能够：

1. 安装 Piper 机械臂的 Python SDK（piper_sdk），并通过 CAN 总线与机械臂通信；
2. 运行官方 demo，使能机械臂并控制其完成简单关节运动；
3. 搭建 piper_ros 功能包工作环境，在 RViz 中显示机械臂模型；
4. 通过 ROS 话题和服务控制真实机械臂，并理解其背后的机制。

## 实验前准备

| 项目 | 要求 |
| --- | --- |
| 电脑 | Ubuntu 20.04（已安装 ROS Noetic 桌面版） |
| 硬件 | Piper 机械臂一台、官方 USB-CAN 模块一个、机械臂电源 |
| 权限 | 具有 sudo 权限的账号 |
| 网络 | 可访问 GitHub 与 PyPI |

> **安全须知（务必阅读）**
> 本实验中机械臂会真实运动。运行任何控制程序前，请确认：
> - 机械臂已可靠固定，工作半径内无障碍物和人员；
> - 急停开关触手可及；
> - 首次运行时先用小幅度姿态测试，确认方向正确后再做更大幅度运动。

---

## 第一部分：安装 piper_sdk

### 1.1 安装 pip 与 python-can

Ubuntu 20.04 默认可能没有 pip，先安装：

```bash
# 方法A（推荐）：apt 安装
sudo apt-get update
sudo apt-get install -y python3-pip

# 方法B：如果 apt 源不可用，使用官方引导脚本安装到用户目录
cd /tmp
curl -sS https://bootstrap.pypa.io/pip/3.8/get-pip.py -o get-pip.py
python3 get-pip.py --user
```

安装 python-can（SDK 依赖，版本需高于 3.3.4）：

```bash
pip3 install python-can
```

> **检查点**：执行 `pip3 --version` 和 `python3 -c "import can; print(can.__version__)"` 均不报错（实测 python-can 4.5.0）。
> 如果提示 `pip3: command not found`，说明 pip 安装到了用户目录，执行：
> ```bash
> export PATH="$HOME/.local/bin:$PATH"
> echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
> ```

### 1.2 安装 piper_sdk

三种方法任选其一（推荐方法 B，可同时获得 demo 源码）：

```bash
# 方法A：直接从 PyPI 安装
pip3 install piper_sdk

# 方法B：克隆 GitHub 仓库安装（本实验采用）
cd ~/code          # 或你喜欢的目录
git clone https://github.com/agilexrobotics/piper_sdk.git
cd piper_sdk
pip3 install .

# 方法C：下载 GitHub Release 中的 whl 文件安装
pip3 install piper_sdk-X.X.X-py3-none-any.whl
```

验证安装：

```bash
pip3 show piper_sdk
python3 -c "from piper_sdk import C_PiperInterface_V2; print('import OK')"
piper-can -h
```

> **检查点**：`pip3 show piper_sdk` 显示版本号（本实验实测 0.7.0a1）；`piper-can -h` 打印 `find` / `activate` 子命令帮助。
> 若 `piper-can` 命令找不到，同样按 1.1 的方法把 `~/.local/bin` 加入 `PATH`。

> **关于 Anaconda / conda 环境（常见踩坑，本实验实测）**
>
> 如果电脑安装了 Anaconda 并激活了某个 conda 环境（例如 `emc`），终端里的 `python3` 和 `pip3` 会指向 conda 环境（可用 `which pip3` 确认，conda 环境的 Python 可能是 3.11 等高版本），而不是系统 Python 3.8。实测现象与解法：
>
> - **现象 1**：之前用系统 Python 装的 piper_sdk 在 conda 环境里**不可见**，`piper-can` 命令找不到。
>   解决：在你打算使用的 conda 环境里重新安装一遍即可：
>   ```bash
>   conda activate emc
>   which pip3        # 确认指向 ~/anaconda3/envs/emc/bin/pip3
>   pip3 install python-can
>   cd ~/code/piper_sdk && pip3 install .
>   piper-can -h      # 验证
>   ```
> - **现象 2**：conda 环境下运行 `roslaunch` 时节点**闪退**，报错 `ModuleNotFoundError: No module named 'yaml'`。原因：ROS 的部分 Python 脚本（如 `joint_state_publisher_gui`、`piper_ctrl_single_node.py`）shebang 是 `#!/usr/bin/env python3`，会被 conda 的 Python 接管。两种解法：
>   - **简单做法**：运行 ROS 命令前先 `conda deactivate` 退出 conda 环境（提示符前的 `(base)`/`(emc)` 消失）；
>   - **本实验机已做的修复（推荐）**：把相关脚本的 shebang 改为 `#!/usr/bin/python3` 强制使用系统 Python，这样在 conda 环境下也能直接运行 ROS：
>     ```bash
>     sudo sed -i '1s|^#!/usr/bin/env python3|#!/usr/bin/python3|' \
>       /opt/ros/noetic/lib/joint_state_publisher_gui/joint_state_publisher_gui
>     sed -i '1s|^#!/usr/bin/env python3|#!/usr/bin/python3|' \
>       ~/piper_ws/src/piper_ros/src/piper/scripts/piper_ctrl_single_node.py
>     ```
> - 如果希望新开终端不自动进入 conda base 环境：`conda config --set auto_activate_base false`

### 1.3 安装 CAN 工具并激活 CAN 模块

机械臂通过 CAN 总线通信，需先安装工具并激活接口：

```bash
sudo apt-get install -y can-utils ethtool iproute2
```

将机械臂的 USB-CAN 模块插入电脑并给机械臂上电，然后查找 CAN 模块：

```bash
piper-can find
```

正常输出示例：

```
Both ethtool and can-utils are installed.
Interface can0 is connected to USB port 3-1.4:1.0
```

激活 CAN 接口（波特率固定为 1000000，不可修改）：

```bash
# 电脑只插了一个 CAN 模块时：
piper-can activate can0 1000000

# 插了多个 CAN 模块时，用 USB 端口号区分（以上面查到的 3-1.4:1.0 为例）：
piper-can activate can_piper 1000000 --usb-port "3-1.4:1.0"
```

> **检查点**：执行 `ifconfig` 或 `ip -details link show can0` 能看到 can0 接口即激活成功。实测正常状态如下（注意 `state UP`、`ERROR-ACTIVE`、波特率 1000000）：
>
> ```
> 5: can0: <NOARP,UP,LOWER_UP,ECHO> mtu 16 ... state UP mode DEFAULT
>     link/can ... can state ERROR-ACTIVE ... bitrate 1000000
> ```
>
> 注意：`piper-can activate` 激活的是内核 CAN 接口，**重启电脑或重新拔插 USB 后需要重新激活**。

### 1.4 运行关节控制 demo（机械臂会运动！）

> **先查固件版本**：执行附录中的固件查询代码。
> - 固件 < 1.8-8：使用 V2 版 demo（下述命令）
> - 固件 ≥ 1.8-8（如本实验机的 S-V1.9-0）：**必须改用 V3 版 demo**
>   `python3 piper_sdk/demo/V3/piper_ctrl_joint.py`
>   否则模式切换指令不生效——实测 V2 demo 在新固件上发出指令后机械臂完全不动作。

```bash
cd ~/code/piper_sdk
python3 piper_sdk/demo/V2/piper_ctrl_joint.py        # 旧固件
# 或
python3 piper_sdk/demo/V3/piper_ctrl_joint.py        # 新固件（≥1.8-8）
```

**预期现象**：

- 终端持续打印机械臂状态 `GetArmStatus()` 和目标位置；
- 控制模式从待机/示教切换为 `CAN_CTRL(0x1)`，机械臂自动使能（`EnablePiper`）；
- 各关节在零位与目标姿态 `[0.2, 0.2, -0.2, 0.3, -0.2, 0.5]`（单位：rad）之间每约 1.5 秒循环切换一次；
- 夹爪同步开合（0 ↔ 70 mm）。

按 `Ctrl+C` 停止程序。

> **实测验证方法**：程序停止后立即读取关节反馈，若停在 `[0,0,0,0,0,0]` 或目标姿态（取决于停止时机），且 `Arm Status: NORMAL`、`Error Code: 0`，说明控制链路完全正常。

> **思考**：阅读 `piper_sdk/demo/V3/piper_ctrl_joint.py`，回答：
> 1. `factor = 57295.7795` 的作用是什么？（提示：弧度 → 0.001 度）
> 2. `MotionCtrl_2(0x01, 0x01, 100, 0x00)` 中各参数含义是什么？（查阅接口 README）
> 3. 为什么程序每 5ms 就重复发送一次相同的控制指令，而不是只发一次？

**实验结束让机械臂失能**（掉电前请先将机械臂放回安全姿态）：

```bash
python3 piper_sdk/demo/V3/piper_ctrl_disable.py
```

**常见问题**：

| 现象 | 原因与处理 |
| --- | --- |
| `Message NOT sent` / `SEND_MESSAGE_FAILED` | CAN 未连通。检查绿色端子是否插紧，重新拔插 USB-CAN 模块，机械臂重新上电，重新激活 CAN |
| 读不到关节反馈 | 机械臂可能处于主模式（master），用 `piper.MasterSlaveConfig(0xFC, 0, 0, 0)` 切换为从模式 |
| 使能一直失败 | 确认波特率为 1000000；确认急停未按下；断电重启机械臂后重试 |
| 状态卡在 `TEACHING_MODE(0x2)`，发指令不动作 | 机械臂处于拖动示教模式（通常是机身上的示教按钮被按下过）。**实测结论：硬件按钮进入的示教模式，CAN 指令无法退出**，必须先按一次机身上的示教按钮结束记录（Teach Status 由 `START_RECORDING(0x1)` 变为 `STOP_RECORDING(0x2)`），之后 CAN 指令即可接管；若是 CAN 指令进入的示教模式，可用 `piper.MotionCtrl_1(0x00, 0x00, 0x02)` 退出。从示教/MIT 模式切回位置速度控制后需执行一次 `demo/V3/piper_ctrl_reset.py`（恢复指令） |
| 模式正常但 demo 发了指令机械臂不动 | 检查固件版本：≥1.8-8 必须用 V3 接口/demo（`C_PiperInterface_V3`），V2 的模式切换指令在新固件上不生效（本实验实测踩中此坑） |

---

## 第二部分：安装 piper_ros 并用 RViz 测试

### 2.1 安装 ROS 依赖

```bash
source /opt/ros/noetic/setup.bash
sudo apt-get install -y python3-wstool python3-catkin-tools python3-rosdep \
  ros-noetic-ruckig ros-noetic-eigen-stl-containers ros-noetic-geometric-shapes \
  ros-noetic-pybind11-catkin ros-noetic-moveit-resources-panda-moveit-config \
  ros-noetic-ompl ros-noetic-warehouse-ros ros-noetic-eigenpy \
  ros-noetic-rosparam-shortcuts ros-noetic-moveit-msgs ros-noetic-srdfdom
```

piper_ros 通过 pip 版的 piper_sdk 与机械臂通信，确保第一部分已完成。

> 提示：本实验机的 `~/.bashrc` 中已加 `source /opt/ros/noetic/setup.bash`，新终端可省略第一条 source。可用 `grep noetic ~/.bashrc` 确认。

### 2.2 克隆并编译

```bash
mkdir -p ~/piper_ws/src
cd ~/piper_ws/src
git clone https://github.com/agilexrobotics/piper_ros.git
cd piper_ros
git checkout noetic        # 必须切换到 noetic 分支
catkin_make                # 在 piper_ros 目录内编译
```

> **重要（本实验实测）**：`src/piper_moveit/` 下捆绑了 **MoveIt 1.1.11 完整源码**（约 19MB），直接编译需要 30~60 分钟以上且容易报错。**强烈建议直接用 apt 版 MoveIt 跳过源码编译**：
>
> ```bash
> sudo apt-get install -y ros-noetic-moveit
> touch ~/piper_ws/src/piper_ros/src/piper_moveit/moveit-1.1.11/CATKIN_IGNORE
> cd ~/piper_ws/src/piper_ros && rm -rf build devel && catkin_make
> ```
>
> 实测：源码编译超过 30 分钟未完成；改用 apt 版后干净重编译仅约 35 秒。

> **检查点**：编译结束无 `error`，且存在文件 `~/piper_ws/src/piper_ros/devel/setup.bash`。

### 2.3 RViz 测试

```bash
source ~/piper_ws/src/piper_ros/devel/setup.bash
roslaunch piper start_single_piper_rviz.launch
```

**预期现象**：

- RViz 打开并显示带夹爪的 Piper 机械臂模型；
- 同时弹出一个 `joint_state_publisher_gui` 滑条窗口；
- 若机械臂已连接，RViz 中的模型会**跟随真实机械臂同步运动**（拖动真机，模型跟着动）。

> **说明**：该 launch 文件同时做了两件事：
> 1. 启动 RViz 显示节点（`piper_description/launch/piper_with_gripper/display_xacro.launch`，内含 joint_state_publisher_gui）；
> 2. 启动 CAN 控制节点 `piper_ctrl_single_node.py`（默认 `can_port=can0`，`auto_enable=true` 会自动尝试使能机械臂）。
>
> 若机械臂未连接，控制节点会报错退出，RViz 模型显示不受影响。
>
> **检查点**：`rosnode list` 应看到 5 个节点：`/joint_state_publisher_gui`、`/piper_ctrl_single_node`、`/robot_state_publisher`、`/rosout`、`/rviz`；`rostopic echo -n 1 /joint_states_single` 能看到真实关节角。

### 2.4 通过 ROS 控制机械臂（重点理解）

#### 服务是怎么来的？

`/enable_srv`、`/go_zero_srv` 等服务**不需要、也不能单独启动**——它们由 `piper_ctrl_single_node` 节点在启动时创建。

> **踩坑实录**：`rosservice call /go_zero_srv` 报 `Service [/go_zero_srv] is not available`，原因就是控制节点没在运行。先 `rosnode list` 确认 `/piper_ctrl_single_node` 存在再调用服务。
>
> 另外注意：**不要重复启动多个 roslaunch**。后启动的同名节点会把先启动的顶替掉（ROS master 会通知旧节点下线），造成"刚才还好好的，服务突然没了"的现象。

服务列表及用法（实测输出 `enable_response: True` / `status: True`）：

```bash
rosservice call /enable_srv "enable_request: true"     # 使能
rosservice call /enable_srv "enable_request: false"    # 失能
rosservice call /go_zero_srv "is_mit_mode: false"      # 回零位（非MIT模式）
rosservice call /stop_srv                              # 停止（恒定阻尼下落）
rosservice call /reset_srv                             # 重置（立刻掉电下落，慎用！）
```

话题列表：

| 话题 | 作用 |
| --- | --- |
| `/joint_states_single` | 机械臂关节状态反馈（节点发布） |
| `/joint_states` | 关节控制指令（节点订阅，经 remap） |
| `/end_pose`、`/end_pose_euler` | 末端位姿反馈（四元数 / 欧拉角） |
| `/arm_status` | 机械臂状态（控制模式、错误码等） |
| `/enable_flag` | 发布 `std_msgs/Bool` 使能/失能 |

#### 用话题控制关节运动：一个关键陷阱

> **踩坑实录**：在 `start_single_piper_rviz.launch` 下直接 `rostopic pub /joint_states ...` **不会生效**。
> 原因：rviz launch 里的 `joint_state_publisher_gui`（滑条窗口）**持续不断地**把滑条值发布到 `/joint_states`，而控制节点订阅的正是这个话题（launch 中做了 `joint_ctrl_single → /joint_states` 的 remap）。你单发一帧，约 20ms 内就被 GUI 的全 0 值覆盖。可用 `rostopic info /joint_states` 查看谁在发布这个话题来验证。
>
> 另一个前提：控制节点的 `joint_callback` 会先检查 `GetEnableFlag()`，**机械臂未使能时直接丢弃指令**。

正确的控制方式（三选一）：

- **方式 A（rviz launch 的设计用法）**：直接拖动 `joint_state_publisher_gui` 窗口里的滑条，机械臂实时跟随。**注意滑条一动真机就动**，务必小幅度先试。（rviz launch 中 `gripper_val_mutiple=2`，滑条的 joint7 值会 ×2 后发给夹爪，因为 GUI 滑条量程 0~0.04 而真实夹爪行程 0.08m。）
- **方式 B（用 rostopic 命令控制）**：换不带 GUI 的 launch：
  ```bash
  roslaunch piper start_single_piper.launch
  ```
  再发布关节目标（机械臂会抬起，运动前确认周围无障碍）：
  ```bash
  rostopic pub /joint_states sensor_msgs/JointState "header:
    seq: 0
    stamp: {secs: 0, nsecs: 0}
    frame_id: ''
  name: ['']
  position: [0.2,0.2,-0.2,0.3,-0.2,0.5,0.01]
  velocity: [0,0,0,0,0,0,10]
  effort: [0,0,0,0,0,0,0.5]"
  ```
- **方式 C**：`rostopic pub -r 10 ...` 持续发布压过 GUI（不推荐，两路指令打架会导致抖动）。

> **思考**：
> 4. 用 `rostopic info /joint_states` 观察发布者和订阅者，解释为什么方式 B 中换成 `start_single_piper.launch` 后 rostopic 控制就能生效。
> 5. 结合 `piper_ctrl_single_node.py` 源码（`~/piper_ws/src/piper_ros/src/piper/scripts/`），找出服务（Service）和话题（Topic）这两种通信方式在本实验中分别适合什么场景。

---

## 实验报告要求

1. 截图并粘贴以下命令的输出：`pip3 show piper_sdk`、`piper-can find`、`ip -details link show can0`；
2. 记录运行 `piper_ctrl_joint.py` 时机械臂的实际运动过程，并解释程序中 `position` 数组每一位的含义；
3. 截图 RViz 中机械臂模型显示界面（要求模型跟随真机）；
4. 截图 `rosservice call /enable_srv "enable_request: true"` 的返回结果；
5. 回答 1.4 节的思考题 1~3 和 2.4 节的思考题 4~5。

## 附录 A：URDF 版本说明

- 固件版本 < S-V1.6-3：使用 `piper_description_old.urdf`
- 固件版本 ≥ S-V1.6-3：使用 `piper_description.urdf`（默认）

固件版本查询方法：

```python
from piper_sdk import *
import time
piper = C_PiperInterface("can0")
piper.ConnectPort()
time.sleep(0.025)
print(piper.GetPiperFirmwareVersion())    # 本实验机实测：S-V1.9-0
```

## 附录 B：本实验机实测环境备忘

| 项目 | 实测值 |
| --- | --- |
| 系统 | Ubuntu 20.04.6，ROS Noetic，系统 Python 3.8.10 |
| conda | Anaconda 装有 `emc` 环境（Python 3.11），SDK 已在系统 Python 和 emc 双环境安装 |
| piper_sdk | 0.7.0a1（GitHub 源码安装于 `~/code/piper_sdk`） |
| python-can | 4.5.0 |
| piper_ros | noetic 分支，`~/piper_ws/src/piper_ros`，MoveIt 用 apt 版 + CATKIN_IGNORE |
| 机械臂固件 | S-V1.9-0（V3 协议，须用 V3 demo） |
| CAN | can0，1000000bps，gs_usb 官方模块 |
| 已做的系统修改 | `joint_state_publisher_gui` 与 `piper_ctrl_single_node.py` 的 shebang 改为 `#!/usr/bin/python3`（解决 conda 下闪退） |
