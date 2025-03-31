# QUAR-HOVER
基于Ardupilot的陆空两栖无人机飞控固件
ArduPilot 作为一款开源的无人系统控制固件，支持多种平台，包括多旋翼（Copter）、固定翼（Plane）、无人车（Rover）和垂直起降固定翼（VTOL）。但目前官方版本并未直接支持车模式（Rover）与飞行模式（Copter）的无缝切换。因此，实现“一键切换”功能需要对 ArduPilot 进行二次开发，主要涉及飞控固件修改、模式管理优化和硬件兼容适配等方面。
1. 关键技术点
1.1 模式切换逻辑
在 ArduPilot 现有框架中，不同的固件（如 Copter 和 Rover）采用不同的控制逻辑和姿态解算方式。我们需要设计一个切换机制，使其能在地面行驶模式和飞行模式间快速切换，而不需要重启飞控或上传不同的固件。
核心思路：在 ArduPilot 代码中新增一个“混合模式”（Hybrid Mode），它能在 Rover 和 Copter 之间动态调整控制方式。
使用辅助通道（RC Channel）或 MAVLink 命令 触发模式切换。
在模式切换时，确保：电机控制方式调整（履带电机 vs. 多旋翼电机）。
传感器数据正确解析（地面 IMU vs. 飞行 IMU）。
PID 参数适配不同模式（地面控制 vs. 空中姿态控制）。

2. ArduPilot 代码修改
ArduPilot 代码结构主要包括：
AP_Vehicle（通用车辆控制层）
AP_Motors（电机控制）
AP_NavEKF（状态估计）
AP_Mode（模式管理）

2.1 新增混合模式
在 libraries/AP_Modes 目录下，创建 Mode_Hybrid.cpp 并在 mode.h 头文件中注册新模式
2.2 适配不同模式的控制
电机控制：
多旋翼模式：使用 PWM 信号控制无刷电机（通过 AP_MotorsMatrix）。
车模式：使用 差速驱动履带电机（通过 AP_MotorsUGV）。

3. 传感器与硬件适配
3.1 传感器切换
气压计：地面模式可能受到气压波动影响，建议融合 轮速计 或 光流传感器 来提高精度。
GPS：如果地面模式需要高精度路径规划，可搭配 RTK GPS 或 视觉 SLAM。

3.2 动力系统兼容
无刷电机（飞行模式）+ 直流电机（履带模式） 需要独立 ESC 控制：
可使用 Pixhawk AUX 通道 控制履带电机，主通道仍用于飞行控制。

4. 一键切换方式
遥控器通道触发
使用 RC Channel 7 作为模式切换：
PWM > 1800 → Copter 模式
PWM < 1200 → Rover 模式

5. 测试与优化
仿真测试：在 SITL（Software In The Loop）环境中，使用 Gazebo/JSBSim 模拟切换效果。

实机调试：先验证 单模式稳定性，确保 Rover 和 Copter 独立运行正常。
逐步测试 模式切换的过渡状态，优化 IMU 数据融合，减少切换时的抖动。
监测电机功率，防止履带模式突然加速导致翻车。
