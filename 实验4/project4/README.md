#  第4次实验（数据采集）和第5次实验（训练和推理）
本项目可支持Piper机械臂的数据采集、训练和推理。

> # ⚠️ **Warning**
>
> ## 采集数据阶段请连接主从臂 !
>
> ## 推理阶段请断开主臂, 只保留从臂 !
>
> ## 在项目的整个过程中，如机械臂出现非正常现象，可尝试拔掉机械臂电源线，切记拔电时应手扶机械臂
---
## ✅ 预备操作
### 参考第一次实验，分别激活两个臂的端口，设置主从臂。首先将带有夹爪的臂连接到电脑，将其设置为从臂模式

```bash
$ piper-can activate can0 1000000
$ cd ~/code/piper_sdk
$ python piper_sdk/demo/V3/piper_set_slave.py
```
~/code/piper_sdk改成第一次实验中，自己电脑上的路径

### 然后将带有示教器的臂连接到电脑，将其设置为主臂模式
```bash
$ piper-can activate can0 1000000
$ cd ~/code/piper_sdk
$ python piper_sdk/demo/V3/piper_set_master.py
```

然后将两个机械臂下电

将两个机械臂共同连接在一个can模块上（两个臂的黄色线拧在一起，蓝色线拧在一起，然后接入绿色的模块中），令主从臂的can连接

先将从动臂上电，然后将主动臂上电，稍等几秒，即可进行遥操作

如未成功，请查看can线是否连接正常，尝试重新断电上电

## ✅ 环境安装

### 创建虚拟环境并安装依赖
```bash
$ conda create -n env python=3.10.9
$ conda activate env
$ python -m pip install -r requirements.txt
```
---

## ✅ 接口配置

#### 1. 启动单臂通信
##### (1) 连接机械臂 usb
> ⚠️ **Note: 第一次连接机械臂前请务必检查 CAN 口配置**

在第一次连接机械臂时，请执行以下步骤查找并配置对应的 **CAN 口**：

*在终端中执行以下命令：

```bash
sudo ethtool -i can0 | grep bus
```
并记录下 bus-info 的数值，例如 3-5.1:1.0; 如果未识别出 can0 , 执行以下脚本查看所有连接的 CAN 模块及其对应的 USB 地址
```bash
$ cd ${项目路径}
$ conda activate env
$ bash piper_sdk/find_all_can_port.sh
```
将 bus-info 的数值填写到{项目路径}/activate_single_arm_can.sh 中, 如下图所示
![camera idx](figure/configure_single_piper_can.png)
配置完成后，只要不更换连接的 USB 端口，后续重复连接机械臂时无需再次更改该脚本
##### (2) 激活机械臂 can 口通信
```bash
$ cd ${项目路径}
$ conda activate env
$ bash activate_single_arm_can.sh # 会将机械臂 can 口映射为 can1
```
> ⚠️ **Note**
> 
> 执行完 ./activate_single_arm_can.sh 后会出现如下 can 口激活成功打印，若失败, 重新插拔 usb 或机械臂重新上电, 再尝试连接
![camera idx](figure/single_arm_can_activate_success.png)
#### 2. 查找相机编号
##### (1) 将相机通过 usb 口与主机相连
##### (2) 查找相机编号
```bash
$ cd ${项目路径}
$ PYTHONPATH=. python lerobot/find_cameras.py opencv
```

终端会出现所有识别到的 camera 编号, 如下图所示

![camera idx](figure/camera_idx.png)

> 同时 record_train_poicy 路径下会出现一个 outputs 文件夹，该文件夹下会产生一个名为 captured_images 的文件夹，存放所有识别到的相机拍摄的单张图，如下图所示，也可以通过看图确定编号，只需要看腕部相机和第三视角相机的RGB图。

![camera idx](figure/capture_imgs.png)

记住所需要使用相机的编号，编号为 video 后的数字 (如 video4, 则该相机对应的编号是4)

腕部相机(wrist)与第三视角相机(top)编号要分别对应相应的index_or_path

## ✅ 数据采集验收清单

正式采集数据前，建议先试录 3～5 条完整的任务轨迹，并排检查腕部相机（wrist）与第三视角相机（top）的图像。逐项完成以下验收要求后，再开展批量数据采集。

### 1. 第三视角相机（top）。建议将第三视角相机放在从臂的正前方斜上方，或者侧前方斜上方，朝向从臂的任务操作区域。

- [ ] 完整覆盖所有任务相关操作区域。
- [ ] 同时看得到目标物体和目标放置区域。
- [ ] 关键操作阶段末端执行器清晰可见。
- [ ] 目标物体在图像中具有足够的像素尺寸。
- [ ] 不存在持续性的严重遮挡。
- [ ] 尽量排除主臂及遥操作人员。

### 2. 腕部相机（wrist）

- [ ] 能够看到夹爪的关键部位。
- [ ] 抓取中心及夹爪前方区域清晰可见。
- [ ] 近距离接近物体时不会严重失焦。
- [ ] 夹爪张开及闭合过程可辨识。
- [ ] 机械臂运动时无明显运动模糊。

### 3. 共同要求

- [ ] 相机安装牢固，不发生意外位姿变化。
- [ ] 两路图像与机械臂状态和动作时间对齐。
- [ ] 确认 wrist 和 top 对应正确的 index_or_path）。
- [ ] 光照稳定，无严重过曝或欠曝。
- [ ] 完整试录并检查接近、抓取、搬运、放置全过程。
- [ ] 训练数据采集阶段与自主推理阶段的相机配置保持一致，包括安装位置、拍摄角度、图像方向、分辨率及帧率。

> ⚠️ **Note**
>
> 评价相机摆放是否合理的根本标准，不是图像是否美观或是否完整拍摄机械臂，而是模型能否从图像中获得决定下一步动作所需要的信息。
>
> 若发现任一验收项不满足要求，应先调整相机配置或采集环境，再重新试录并检查。试录时建议使用独立的 dataset.root，避免与正式采集的数据混合。

## ✅ 采集数据
自行设计抓取任务，例如，抓苹果放入盘子中，叠毛巾，放置水瓶等
```bash
$ cd ${项目路径}
# 通过步骤 3 查找到的相机编号，依次修改所需要使用的相机 index_or_path
# dataset.repo_id     : 可根据自己 huggingface 账号命名 (随意命名也可)
# dataset.num_episodes： 记录多少条数据,可根据自己要求进行修改。可以先设置成2条，查看dataset.root中采集的图像等是否正确，然后将dataset.num_episodes改成30条
# dataset.root        : 数据存放地址
# dataset.single_task : 任务名称
# display_data        : 是否在录数据过程中可视化记录数据
$ python -m lerobot.record \
--robot.type=single_piper \
--robot.port=" " \
--robot.id=single_piper_robot \
--robot.cameras="{wrist: {type: opencv, index_or_path: 8, width: 640, height: 480, fps: 30},top: {type: opencv, index_or_path: 14, width: 640, height: 480, fps: 30}}" \
--dataset.repo_id=emb/single_piper \
--dataset.root=./data \
--dataset.num_episodes=30 \
--dataset.single_task="Grab the cube" \
--display_data=True \
--resume=False
```
> ⚠️ **Note**
> 
> robot.cameras -> index_or_path 使用上一步找到的相机编号
> 
> dataset.root 使用当前电脑的路径  请保持改路径为空（新的） 若该路径不为空 会报错（folder already exist...）
> 
> 如果中途因各种问题导致数据录制提前终止，若想保留原有数据继续录制，需在运行代码中的"--resume=False改为"--resume=True"
> 
> 通过键盘上的方向键（← →）、 esc 键控制记录数据
>
> →   : 当前数据记录结束, 开启下一段数据记录
>
> ←   : 不想保存当前数据, 重新录制该episode（如当前录制episode1，按下 ← 后会重新录制episode1）
>
> esc : 提前终止数据记录, 并保存当前数据


---
## ✅ 第5次实验-训练
```bash
$ cd ${项目路径}
$ conda activate env

### 参数说明
# policy.type          : 策略 (如 act、 diffusion\ smolvla)
# dataset.repo_id      : 可根据自己 huggingface 账号命名 (随意命名也可)
# batch_size           : batch_size 大小
# steps                : 轮次
# output_dir           : 训练权重保存地址
# dataset.root         : 数据集存放地址


### act
$ PYTHONPATH=. python lerobot/scripts/train.py \
--policy.type=act \
--dataset.repo_id=my_name/single_piper \
--batch_size=4 \
--steps=200000 \
--output_dir=/home/ubuntu/VLA/result \
--dataset.root=/home/ubuntu/VLA/data

```
> ⚠️ **Note**
>  
> dataset.root 是执行record脚本时设置的路径
> 
> output_dir 使用当前电脑的路径 (保存模型训练权重的位置)
> 
---


## ✅ 推理部署


### 测试训练权重
```bash
### act
# robot.type           : 机器人类型 (single_piper/dual_piper/moving_dual_piper)
# policy.type          : 策略 (如 act/diffusion/smolvla)
# ckpt_path            : 权重保存地址

$ python -m lerobot.inference \
--robot.type=single_piper \
--robot.id=single_piper_robot \
--robot.port=can1 \
--robot.cameras="{wrist: {type: opencv, index_or_path: 4, width: 640, height: 480, fps: 30},top: {type: opencv, index_or_path: 10, width: 640, height: 480, fps: 30}}" \
--policy.type=act \
--ckpt_path=/home/ubuntu/VLA/result/checkpoints/200000/pretrained_model \
--task="Grab the cube"

```

> ⚠️ **Warning**
>
> ckpt_path 与上一步output_dir保持一致
>
> 推理时，优先使用最后一个ckpt文件，如200000
>
> 若推理效果不理想，可尝试使用其他轮次权重，如 180000,160000 等，找到推理效果最佳的轮次