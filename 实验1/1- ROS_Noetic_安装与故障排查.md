# Ubuntu 20.04 安装 ROS Noetic：学生操作指南

本文根据一台实际安装并验证成功的计算机整理，包含 ROS Noetic Desktop-Full、开发工具、环境变量、rosdep，以及安装过程中遇到的故障和修复方法。

参考教程：

- [ROS Noetic Ubuntu 官方安装教程](https://wiki.ros.org/noetic/Installation/Ubuntu)
- [ROS 中文教程](https://wiki.ros.org/cn/ROS/Tutorials)
- [ROS 官方快照仓库说明](https://docs.ros.org/en/humble/Installation/Snapshot-Repository.html)

> **重要说明**
>
> - 本指南适用于 **Ubuntu 20.04（Focal）**，本机版本为 **20.04.6 LTS，x86_64**。
> - ROS Noetic 已停止维护。原主仓库签名密钥已经过期，因此本文使用官方 **Noetic final 快照仓库**。
> - 快照中的软件不再接收功能或安全更新。
> - 不要使用 `curl -k`、`[trusted=yes]` 或 `--allow-unauthenticated` 来绕过安全检查。

## 1. 检查系统版本

```bash
lsb_release -a
uname -m
```

应看到：

```text
Ubuntu 20.04.x LTS
x86_64
```

如果不是 Ubuntu 20.04，不要直接套用本指南。Noetic 的官方二进制包只支持特定 Ubuntu 版本。

安装过程需要使用具有 sudo 权限的账号。出现密码提示时，在自己终端中输入账号密码；输入时屏幕不会显示字符，这是正常现象。

## 2. 准备 Ubuntu 软件源和工具

确保 Ubuntu 已启用 `restricted`、`universe` 和 `multiverse`，然后安装基础工具：

```bash
sudo apt update
sudo apt install -y software-properties-common curl gnupg2 lsb-release
sudo add-apt-repository restricted
sudo add-apt-repository universe
sudo add-apt-repository multiverse
```

## 3. 配置 Noetic final 快照仓库

### 3.1 下载并检查快照仓库密钥

```bash
curl -fL \
  'https://keyserver.ubuntu.com/pks/lookup?op=get&search=0x4B63CF8FDE49746E98FA01DDAD19BAB3CBF125EA' \
  -o /tmp/ros-snapshots.asc

gpg --show-keys --with-fingerprint /tmp/ros-snapshots.asc
```

必须确认显示的完整指纹为：

```text
4B63 CF8F DE49 746E 98FA 01DD AD19 BAB3 CBF1 25EA
```

如果指纹不同，立即停止，不要导入该密钥。

### 3.2 安装密钥并添加软件源

```bash
gpg --yes --dearmor \
  -o /tmp/ros-snapshots-archive-keyring.gpg \
  /tmp/ros-snapshots.asc

sudo mkdir -p /etc/apt/keyrings

sudo install -m 644 \
  /tmp/ros-snapshots-archive-keyring.gpg \
  /etc/apt/keyrings/ros-snapshots-archive-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/ros-snapshots-archive-keyring.gpg] http://snapshots.ros.org/noetic/final/ubuntu focal main" \
  | sudo tee /etc/apt/sources.list.d/ros-latest.list

sudo apt update
```

`apt update` 不应再出现 ROS 仓库的 `EXPKEYSIG` 错误。

## 4. 安装 ROS Noetic Desktop-Full

```bash
sudo apt install -y ros-noetic-desktop-full \
  python3-rosdep python3-rosinstall python3-rosinstall-generator \
  python3-wstool build-essential
```

各组件用途：

- `ros-noetic-desktop-full`：ROS 核心、RViz、rqt、仿真和常用机器人库。
- `python3-rosdep`：解析和安装 ROS 软件包的系统依赖。
- `python3-rosinstall`、`python3-rosinstall-generator`、`python3-wstool`：获取和管理源代码工作区。
- `build-essential`：C/C++ 编译工具。

安装包较多，下载和配置需要一些时间。不要在安装或配置软件包时强制关闭终端。

## 5. 配置 Bash 环境

让每个新的 Bash 终端自动加载 Noetic：

```bash
echo 'source /opt/ros/noetic/setup.bash' >> ~/.bashrc
source ~/.bashrc
```

检查：

```bash
printenv ROS_DISTRO
printenv ROS_VERSION
```

预期输出分别为：

```text
noetic
1
```

> 如果计算机安装了多个 ROS 版本，`~/.bashrc` 中只应自动加载当前要使用的一个版本。

## 6. 初始化 rosdep

### 6.1 先尝试标准方法

```bash
sudo rosdep init
rosdep update --rosdistro noetic
```

必须保留 `--rosdistro noetic`。当前 rosdep 默认跳过停止维护的发行版，不加该选项时可能不会生成 Noetic 的软件包规则。

如果两条命令都成功，跳到第 7 节。

如果出现以下错误：

```text
SSL: CERTIFICATE_VERIFY_FAILED
self signed certificate in certificate chain
```

并且本机使用与本次安装相同的 Scholar Verify 网络，则按第 6.2 节处理。

### 6.2 修复本机 Scholar Verify 网络的 TLS 证书问题

本机访问 GitHub 时，网络会提供由以下 CA 签发的证书：

```text
Scholar Root CA v1
```

以下方法只给 rosdep 命令临时指定 CA bundle，不把该 CA 安装到系统信任库，也不关闭证书验证。

> 仅当证书主题和指纹与本文完全一致时继续。如果你不在 Scholar Verify 网络，或指纹不同，应停止并联系网络管理员，不要盲目信任证书。

创建配置目录并提取、核验 CA：

```bash
mkdir -p ~/.config/rosdep

python3 - <<'PY'
from pathlib import Path
import re
import subprocess

expected = '4DA1578B889DA6F04CBA86F90A347E8FD1569C6469FE2D64C0EF0496310B060A'

try:
    result = subprocess.run(
        [
            'openssl', 's_client', '-showcerts',
            '-connect', 'raw.githubusercontent.com:443',
            '-servername', 'raw.githubusercontent.com',
        ],
        input='', text=True, capture_output=True, timeout=20,
    )
    output = result.stdout
except subprocess.TimeoutExpired as error:
    output = error.stdout or ''
    if isinstance(output, bytes):
        output = output.decode()

certs = re.findall(
    r'-----BEGIN CERTIFICATE-----.*?-----END CERTIFICATE-----',
    output,
    re.S,
)
if not certs:
    raise RuntimeError('没有获取到证书，请检查网络后重试')

root_cert = certs[-1]
check = subprocess.run(
    ['openssl', 'x509', '-noout', '-fingerprint', '-sha256'],
    input=root_cert, text=True, capture_output=True, check=True,
)
actual = check.stdout.strip().split('=')[-1].replace(':', '').upper()
if actual != expected:
    raise RuntimeError(f'CA 指纹不匹配：{actual}；已停止，未写入配置')

config_dir = Path.home() / '.config' / 'rosdep'
(config_dir / 'scholar-root-ca.pem').write_text(root_cert + '\n')
bundle = config_dir / 'rosdep-ca-bundle.pem'
bundle.write_text(
    Path('/etc/ssl/certs/ca-certificates.crt').read_text()
    + '\n' + root_cert + '\n'
)
print(f'CA 指纹核验通过，已创建：{bundle}')
PY
```

如果第 6.1 节的 `rosdep init` 失败了，重新执行：

```bash
sudo env SSL_CERT_FILE="$HOME/.config/rosdep/rosdep-ca-bundle.pem" rosdep init
```

然后更新 Noetic 缓存：

```bash
SSL_CERT_FILE="$HOME/.config/rosdep/rosdep-ca-bundle.pem" \
  rosdep update --rosdistro noetic
```

如果第 6.1 节只有 `rosdep update` 失败，而 `rosdep init` 已成功，不要重复初始化，只执行上面的更新命令。

以后在这台计算机上更新 rosdep 时，也使用最后这条带 `SSL_CERT_FILE` 和 `--rosdistro noetic` 的命令。

## 7. 验证安装

### 7.1 检查 ROS 命令和 rosdep

```bash
command -v roscore
command -v rosrun
rosdep resolve roscpp
rosdep resolve python3-numpy
```

前两条应显示 `/opt/ros/noetic/bin/` 下的路径；后两条应包含：

```text
#apt
ros-noetic-roscpp
```

以及：

```text
#apt
python3-numpy
```

### 7.2 启动 ROS Master

```bash
roscore
```

看到 `started core service [/rosout]` 等内容说明 ROS Master 已启动。测试后按 **Ctrl+C** 停止。

### 7.3 使用 turtlesim 进行端到端测试

打开三个终端，分别执行：

终端 1：

```bash
roscore
```

终端 2：

```bash
rosrun turtlesim turtlesim_node
```

终端 3：

```bash
rosrun turtlesim turtle_teleop_key
```

让第三个终端保持焦点，按方向键。如果海龟移动，说明 ROS Master、节点、话题通信和图形界面工具均可正常工作。

## 8. 实际故障与修复汇总

### 8.1 `gpg: no valid OpenPGP data found`

**示例：**

```text
curl -s https://raw.githubusercontent.com/.../ros.asc | sudo apt-key add -
gpg: no valid OpenPGP data found
```

**原因：** `curl -s` 隐藏了下载错误，导致空内容或错误内容被传给 `apt-key`。本机的实际原因是 GitHub TLS 连接被 Scholar Verify 网络拦截。

**修复：**

- 不要使用静默管道下载密钥。
- ROS 软件源直接使用第 3 节的 final 快照和快照专用密钥。
- rosdep 的 GitHub 访问问题按第 6.2 节处理。
- 下载其他文件时使用 `curl -fL`，让 HTTP 错误正确返回失败。

### 8.2 `EXPKEYSIG F42ED6FBAB17C654`

**示例：**

```text
The following signatures were invalid:
EXPKEYSIG F42ED6FBAB17C654 Open Robotics <info@osrfoundation.org>
E: The repository 'http://packages.ros.org/ros/ubuntu focal InRelease' is not signed.
```

**原因：** Noetic 主仓库使用的旧签名密钥已于 2025 年 6 月过期。

**修复：** 按第 3 节改用 `http://snapshots.ros.org/noetic/final/ubuntu`，并使用指纹为 `4B63 CF8F ... 25EA` 的快照专用密钥。不要通过 `[trusted=yes]` 禁用验证。

修改软件源后重新执行：

```bash
sudo apt update
```

### 8.3 `Waiting for cache lock` / `Could not get lock /var/lib/dpkg/lock-frontend`

**原因：** 另一个 `apt`、`apt-get`、`dpkg` 或系统更新进程正在使用包管理器。本机当时正在完成内核更新。

**修复：** 先查看提示中的进程号，例如进程号为 `76683`：

```bash
ps -o pid,ppid,stat,etime,args -p 76683
```

等待该进程正常结束，再重新执行安装命令。若进程已经结束，直接重试即可。

在仅显示 `Waiting for cache lock` 时按 **Ctrl+C**，只会取消当前等待，不会中断另一个正在安装软件的进程。

> 不要删除 `/var/lib/dpkg/lock-frontend`，也不要强制杀死正在配置软件包的进程，否则可能损坏包管理状态。

### 8.4 `SSL: CERTIFICATE_VERIFY_FAILED`

**原因：** 本机 Scholar Verify 网络的 CA 未被 Python/curl 默认信任，不是 ROS 或 rosdep 本身损坏。

**修复：** 按第 6.2 节核验 CA 指纹，并仅对 rosdep 命令设置 `SSL_CERT_FILE`。不要使用 `curl -k` 或 Python 的未验证 TLS 上下文。

### 8.5 `Skip end-of-life distro "noetic"` 或 `ERROR: no rosdep rule for 'roscpp'`

**原因：** 普通 `rosdep update` 默认跳过停止维护的 Noetic，因而没有生成 Noetic 的 ROS 软件包规则。

**修复：**

正常网络：

```bash
rosdep update --rosdistro noetic
```

本机 Scholar Verify 网络：

```bash
SSL_CERT_FILE="$HOME/.config/rosdep/rosdep-ca-bundle.pem" \
  rosdep update --rosdistro noetic
```

然后验证：

```bash
rosdep resolve roscpp
```

### 8.6 `sudo rosdep init` 提示默认 sources list 已存在

**原因：** rosdep 已经初始化过。

**修复：** 不需要删除已有文件或重复初始化，直接执行：

```bash
rosdep update --rosdistro noetic
```

如果当前网络存在 TLS 问题，则使用第 6.2 节带 `SSL_CERT_FILE` 的更新命令。

### 8.7 `sudo: a password is required`

**原因：** 当前终端没有可用的 sudo 凭据，或命令由无法交互输入密码的脚本启动。

**修复：** 在自己的终端中运行该命令并输入账号密码。不要把密码写进文档、脚本或聊天记录。

## 9. 安装完成后学习顺序

建议按照 ROS 中文教程继续学习：

1. ROS 文件系统：`roscd`、`rosls`、`rospack`。
2. 创建 ROS 软件包。
3. 创建并构建 catkin 工作区。
4. 理解 ROS 节点和话题。
5. 理解 ROS 服务和参数。
6. 使用 `roslaunch`。
7. 编写简单的发布者、订阅者、服务端和客户端。
