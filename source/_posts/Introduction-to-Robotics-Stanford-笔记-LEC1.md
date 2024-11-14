---
title: Introduction to Robotics-Stanford 笔记 LEC1
date: 2024-11-07 20:28:02
tags: 机器人学
banner_img: /images/kisaki3.jpg
---
<!-- <script src="https://fastly.jsdelivr.net/gh/misaka0502/live2d-widget@latest/autoload.js"></script> -->
<script src="https://fastly.jsdelivr.net/gh/misaka0502/live2d-widget@V0.2/autoload.js"></script>
<!-- <script src="/live2d-widget/autoload.js"></script> -->


最近开始学习机器人学，目前跟着斯坦福大学的机器人学导论公开课学，记录一下第一课的笔记，都是些简单介绍的概念。
斯坦福大学机器人学导论公开课地址： [Lecture 1 | Introduction to Robotics](https://www.youtube.com/watch?v=0yD3uBshJB0&list=PL64324A3B147B5578)

# Lecture 1 | Introduction to Robotics
> 要控制机器人，首先需要找到机构本身的所有位置和方向，这需要我们找到物体在空间中的位置和方向的描述。然后我们需要处理附加到这些不同物体的框架之间的变换

## Spasial Descriptions 空间描述
- Position and Orien tation 位置和方向
- Transformations between Frames 两个框架之间的变换
- Forward Kinematics 正向运动学：给出关节角度与末端执行器位置之间的关系，即将关节空间映射到笛卡尔空间
  - 例如D-H参数法

## Manipulator Kinematics 机械臂运动学
- Link Description 链描述
- Denavit-Hartenberg Notion D-H参数法
- Forward Kinematics 正向运动学

## Jacobian（雅可比矩阵）：Velocities and Forces 力和速度
- Velocities: end-effector linear and angular velocities 末端执行器的线速度和角速度
- Forces: end-effoctor foeces and movements 末端执行器的力和运动
- Jacobian: relations
  - joint velocities and end-effector velocities 关节速度和末端执行器速度之间的关系
  - joint torques and end-effector forces 关节力矩和末端执行器力之间的关系
    ![Jacobian](/images/jacobian.png)

## Inverse Kinematics 逆运动学
Finding joint positions given end-effector positions and orientations 给定末端执行器的位置和方向，求关节位置
- Solvability, Existence, Multiplicity 可解性，存在性，多重性
- Closed Form/Numerical Solutions 闭式解(解析解)/数值解

## Trajectory Generation 轨迹生成
通过以上解决方案，我们可以在机器人的给定点的位置之间进行**插值**，然后通过**速度和加速度均平滑的轨迹**以及我们可能施加的其他约束将机器移动到最终的配置（在关节空间和笛卡尔空间中）
- Path Description 路径描述
- Joint Space Trajectory 关节空间轨迹
- Cartesian Space Trajectory 笛卡尔空间轨迹

## Manipulator Control 机械臂控制
用动态结构增强控制器将PID等控制方式运用到关节空间和任务空间中，以便我们在控制机器人时考虑动力学：with the dynamic structure structure so that we account for dynamics when we are controlling the robot.
- PID Control
- Joint Space Dynamic Control 关节空间动态控制
- Catesian Space Dynamic Control 笛卡尔空间动态控制

## Manipulator Force Control 机械臂力控制
在移动中产生接触(contact)时，作用力会将整个结构置于一个约束下，必须考虑这些约束并计算法线以找到反作用力，以便控制施加到环境的力。所以我们需要处理力控制，需要稳定地从自由空间(free space)到接触空间(contact space)的过渡，即需要能够移动时控制这些接触力
- Constrained Motion 受约束的运动
- Position/Force Control 位置/力控制
- Contact Stability 接触稳定性

## Unified Motion/Force Control 统一运动/力控制
在笛卡尔空间或者任务空间中，可以将两种力(motion force 和 contact force)合并在一起，直接控制机器人产生运动和接触
