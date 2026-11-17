# README

## 0. 安装conda环境管理器

参考：https://docs.conda.io/projects/conda/en/stable/user-guide/install/linux.html

推荐安装Miniconda installer for Linux

## 1. Download project1 code, put it under

`/home/ubuntu/arm_student_ws/project1/`

放入自己的目录下

## 2. 创建环境

```bash
conda create -n arm_env python=3.9

conda activate arm_env

cd /home/ubuntu/arm_student_ws/piper_ros/

catkin_make

source /home/ubuntu/arm_student_ws/piper_ros/devel/setup.bash

cd /home/ubuntu/arm_student_ws/project1/

pip install -r requirements
```

## 1. Test Forward Kinematics

terminal 1:
> source /home/ubuntu/arm_student_ws/piper_ros/devel/setup.bash
> 
> roslaunch piper start_single_piper_rviz.launch 
> 
注意路径要修改为你自己的路径地址

terminal 2:
> source /home/ubuntu/arm_student_ws/piper_ros/devel/setup.bash
> 
> cd /home/ubuntu/arm_student_ws/project1/src/
> 
> python fk_test.py
> 
RVIZ里面，Add->By topic->/fk_marker

可以看到，小球跟踪在机械臂末端点 

## 2. Test Inverse Kinematics

terminal 1:
> source /home/ubuntu/arm_student_ws/piper_ros/devel/setup.bash
> 
> roslaunch piper start_single_piper_rviz.launch


terminal 2:
> rosnode kill /joint_state_publisher_gui /piper_ctrl_single_node
>
> source /home/ubuntu/arm_student_ws/piper_ros/devel/setup.bash
> 
> cd /home/ubuntu/arm_student_ws/project1/src/
> 
> python ik_test.py
> 
RVIZ里面，Add->By topic-> /ik_traget_point   /planned_trajectory
可以看到机械臂规划出一条画圆的轨迹


## 5. Implement FK & IK Functions in src/piper_arm.py by yourself 

##### 1.1 Complete the DH table of piper arm
##### 1.2 Complete the forward_kinematics function
##### 1.3 Complete the inverse_kinematics function
