# RM65-B + BrainCo Revo2 + Orbbec Femto Mega 视觉跟随控制 Demo

一个基于 Orbbec Femto Mega RGB-D 相机、睿尔曼 RM65-B 机械臂、强脑 BrainCo Revo2 基础版灵巧手的视觉跟随控制项目。

项目核心目标是：**用视觉识别人手运动，让机械臂跟随手掌位移，让灵巧手跟随手指弯曲**。在通信架构上，推荐机械臂走 RealMan CANFD 控制，灵巧手走强脑官方 RS485 SDK 独立控制，避免机械臂和灵巧手挤在同一条通信链路上“堵车”。

## Pose Lite + 双手优化版

本仓库是原始 Demo 的独立优化版本，原项目代码未被修改。新的默认入口为：

```text
scripts/camera_arm_rm65b_control_pose_hands.py
```

优化版识别链路为：

```text
Orbbec 对齐 RGB-D
  -> 深度连通域生成最近人员候选
  -> MediaPipe Pose Lite 验证躯干并计算稳健躯干深度
  -> 连续多帧锁定最近操作者
  -> 识别左右肩、肘、腕和完整人体骨架
  -> 按左右腕关联两只手的 21 点关键点
  -> 计算左右肩/肘角度、双手掌 XYZ 和双手手指弯曲
  -> 由 control_hand_side 选定控制手
  -> 控制 RM65-B XYZ 跟随和 BrainCo Revo2 手指弯曲
```

关键变化：

- 默认加载仓库内 `pose_landmarker_lite.task`，无需启动时下载 Pose 模型。
- 最多同时识别两只手，并用 Pose 左右腕位置修正 MediaPipe handedness。
- `control_hand_side:=left|right|auto` 决定哪只手控制机械臂和灵巧手。
- 操作者必须具有有效肩/髋躯干关键点和 Orbbec 深度，且稳定达到确认帧数后才能按 `s` 启动。
- 关键点深度使用邻域有效值中位数，不再依赖单个深度像素。
- 保留原版主程序作为对照，不作为优化版自启动入口。

新增 ROS2 话题：

| 话题 | 数据顺序 |
| --- | --- |
| `operator_arm_joint_angles` | 左肩、左肘、右肩、右肘，单位为度 |
| `operator_dual_hand_bends` | 左手五指、右手五指；每手顺序为拇指、食指、中指、无名指、小指 |
| `operator_dual_palm_xyz` | 左手掌 XYZ、右手掌 XYZ，相机坐标系，单位为米 |
| `operator_torso_depth` | 当前锁定操作者的稳健躯干深度，单位为米 |
| `operator_skeleton_3d` | MediaPipe 0～32 号骨架点，每点依次为 X、Y、Z、置信度 |

主要识别超参数：

| 参数 | 默认值 | 作用 |
| --- | --- | --- |
| `person_min_depth_m` / `person_max_depth_m` | `0.35` / `4.0` | 操作者允许出现的相机深度范围 |
| `operator_ready_confirm_frames` | `5` | 最近操作者连续稳定多少帧后进入可启动状态 |
| `operator_ready_max_depth_jump_m` | `0.25` | 准备阶段相邻帧允许的最大躯干深度跳变 |
| `operator_ready_max_center_jump_px` | `160` | 准备阶段人体中心允许的最大像素跳变 |
| `operator_torso_min_landmarks` | `2` | 肩/髋至少多少个点具有有效 Orbbec 深度 |
| `mp_max_num_hands` | `2` | 最大同时识别手数 |
| `mp_hand_assignment_max_distance_ratio` | `0.35` | 手腕与 Pose 左右腕关联距离上限 |
| `mp_handedness_swap` | `false` | 相机图像被镜像时交换左右手标签 |
| `control_hand_side` | `right` | 使用左手、右手或自动选择的手控制硬件 |
| `control_hand_fallback_enabled` | `false` | 控制手不可见时是否允许另一只手接管 |
| `landmark_depth_patch_radius_px` | `2` | 手部/骨架点深度中值采样半径 |

## 技术亮点

- **RGB-D 视觉感知**  
  通过 `pyorbbecsdk` 获取 Femto Mega 深度对齐彩色图像，支持最近人标定与跟踪。

- **神经网络手部识别**  
  使用 MediaPipe Hands / PoseLandmarker 模型识别手掌、手指关键点，并计算手掌位移与手指弯曲程度。

- **机械臂 CANFD 丝滑跟随**  
  RM65-B 机械臂通过 RealMan API 的 `rm_movep_canfd` 接口接收连续 TCP 目标位姿，适合实时跟随控制。

- **灵巧手独立 RS485 控制**  
  BrainCo Revo2 通过强脑官方 `bc_stark_sdk` 的 RS485/Modbus 接口控制，支持 `set_finger_positions_and_durations` 等位置控制方式。

- **多层滤波与安全保护**  
  包含 MIT 风格轨迹整形、手掌死区、手指弯曲去抖、误弯曲过滤、关节限位与奇异区保护。

- **GUI 在线调参**  
  程序启动后未按 `s` 前可通过 OpenCV GUI 调整关键参数，并可保存到默认参数 JSON，方便下一次启动复用。

## 仓库结构

```text
Jetson_Femto_Realman_Brainco_Demo_MediaPipe_Pose/
├── README.md
├── mediapipe_models/
│   ├── pose_landmarker_full.task
│   └── pose_landmarker_lite.task
└── scripts/
    ├── camera_arm_rm65b_control_pose_hands.py
    ├── camera_arm_rm65b_pose_hands.defaults.json
    ├── camera_arm_rm65b_control_neural.py
    └── camera_arm_rm65b_control.defaults.json
```

### 文件说明

| 路径 | 说明 |
| --- | --- |
| `scripts/camera_arm_rm65b_control_pose_hands.py` | Pose Lite、双手、躯干深度选人和硬件控制的优化版入口 |
| `scripts/camera_arm_rm65b_pose_hands.defaults.json` | 优化版 GUI 和响应参数默认值 |
| `scripts/camera_arm_rm65b_control_neural.py` | 从原仓库复制的基线主程序，仅用于对照 |
| `scripts/camera_arm_rm65b_control.defaults.json` | 原版默认参数，仅用于对照 |
| `mediapipe_models/pose_landmarker_full.task` | MediaPipe PoseLandmarker full 模型 |
| `mediapipe_models/pose_landmarker_lite.task` | MediaPipe PoseLandmarker lite 模型 |

## 硬件连接推荐

```text
Jetson
├── Ethernet -> RM65-B 控制器 192.168.1.18:8080
├── USB / Orbbec SDK -> Femto Mega RGB-D 相机
└── USB-RS485 -> BrainCo Revo2 RS485 A/B/GND

RM65-B 末端
└── 24V 工具端电源 -> BrainCo Revo2 供电
```

注意：推荐让 Revo2 的 **通信线** 接 Jetson 独立 USB-RS485，供电可以继续由 RM65-B 末端 24V 提供。这样机械臂 CANFD 和灵巧手 RS485 两条链路互不抢资源。

## 环境准备

### 1. RealMan RM_API2

确认 Jetson 上已经安装或放置 RM_API2，并且 Python 可以导入：

```bash
python3 -c "from Robotic_Arm.rm_robot_interface import *; print('RM_API2 OK')"
```

### 2. Orbbec pyorbbecsdk

确认 Femto Mega 可以正常出图。建议先运行 Orbbec 官方示例，确认彩色流与深度流正常。

### 3. BrainCo Hand SDK

推荐路径：

```bash
/home/jetson/ws/brainco-hand-sdk-main/python
```

确认强脑 SDK 可导入：

```bash
python3 -c "from bc_stark_sdk import main_mod as sdk; print('BrainCo SDK OK')"
```

### 4. 检查机械臂网络

```bash
ping 192.168.1.18
python3 -c "import socket; socket.create_connection(('192.168.1.18',8080),3); print('RM TCP OK')"
```

## 快速启动

进入仓库根目录：

```bash
cd /home/jetson/ws/Jetson_Femto_Realman_Brainco_Demo_MediaPipe_Pose
```

推荐运行命令：

```bash
python scripts/camera_arm_rm65b_control_pose_hands.py \
  --ros-args \
  -p robot_motion_enabled:=true \
  -p robot_ip:=192.168.1.18 \
  -p arm_control_protocol:=canfd \
  -p canfd_follow:=false \
  -p canfd_trajectory_mode:=2 \
  -p canfd_radio:=50 \
  -p arm_command_interval_s:=0.03 \
  -p max_arm_target_delta_m:=0.008 \
  -p min_command_delta_m:=0.002 \
  -p arm_motion_unit_m:=0.002 \
  -p hand_control_enabled:=true \
  -p hand_transport:=brainco-rs485 \
  -p hand_rs485_sdk_path:=/home/jetson/ws/brainco-hand-sdk-main/python \
  -p hand_rs485_port:=auto \
  -p hand_rs485_slave_id:=0 \
  -p hand_rs485_baudrate:=460800 \
  -p hand_rs485_duration_ms:=40 \
  -p mp_pose_enabled:=true \
  -p mp_pose_backend:=tasks \
  -p mp_pose_task_model_path:=mediapipe_models/pose_landmarker_lite.task \
  -p mp_max_num_hands:=2 \
  -p control_hand_side:=right \
  -p hand_wait_for_first_arm_command:=false \
  -p hand_pause_while_arm_sdk_busy:=false \
  -p hand_configure_tool_voltage:=true \
  -p hand_tool_voltage_type:=3
```

启动后等待日志出现：

```text
Hardware/software initialization complete
```

然后在图像窗口中：

- 按 `s`：开始识别和控制
- 按 `h`：回到安全原位
- 按 `p`：保存 GUI 调参结果
- 按 `q` / `Esc`：退出程序

## 常用调试命令

### 只测试机械臂 CANFD

```bash
python scripts/camera_arm_rm65b_control_pose_hands.py \
  --ros-args \
  -p robot_motion_enabled:=true \
  -p robot_ip:=192.168.1.18 \
  -p arm_control_protocol:=canfd \
  -p hand_control_enabled:=false
```

看到日志中出现 `protocol=CANFD`，说明机械臂控制协议正确。

### 只测试灵巧手 RS485

```bash
python scripts/camera_arm_rm65b_control_pose_hands.py \
  --ros-args \
  -p robot_motion_enabled:=true \
  -p robot_ip:=192.168.1.18 \
  -p arm_control_protocol:=canfd \
  -p max_arm_offset_m:=0.0 \
  -p hand_control_enabled:=true \
  -p hand_transport:=brainco-rs485 \
  -p hand_rs485_sdk_path:=/home/jetson/ws/brainco-hand-sdk-main/python \
  -p hand_rs485_port:=auto \
  -p hand_rs485_slave_id:=0 \
  -p hand_configure_tool_voltage:=true \
  -p hand_tool_voltage_type:=3
```

强脑 SDK 检测成功时应看到类似日志：

```text
Found device at baudrate Baud460800, slave_id: 0x7F
Device SN: "...", hw_type: Revo2Basic
Successfully detected device on port: /dev/ttyUSB0
```

## 常见问题

| 现象 | 可能原因 | 处理方式 |
| --- | --- | --- |
| 机械臂不动，但终端有识别输出 | 参数阈值过大，目标命令被过滤 | 降低 `min_command_delta_m`，增大 `max_arm_target_delta_m` |
| 日志显示 `protocol=RS485` | 忘记传 `arm_control_protocol:=canfd` | 补上 CANFD 参数 |
| 强脑 SDK 找不到手 | 串口错、A/B 接反、未共地、未供电 | 使用 `hand_rs485_port:=auto`，检查 A/B/GND/24V |
| `common_imports` 找不到 | SDK 路径没加进去 | 设置 `hand_rs485_sdk_path` |
| 灵巧手误弯曲 | 视觉关键点抖动或遮挡 | 提高 `hand_bend_deadzone`，增加确认帧 |
| 机械臂抖动 | 速度过快或滤波太弱 | 提高 `arm_mit_kd`，降低最大速度/加速度 |

## 关键参数速查

| 参数 | 推荐值 | 说明 |
| --- | --- | --- |
| `arm_control_protocol` | `canfd` | 机械臂使用 CANFD 控制 |
| `canfd_trajectory_mode` | `2` | CANFD 轨迹滤波模式 |
| `canfd_radio` | `50` | CANFD 平滑参数 |
| `arm_command_interval_s` | `0.03` | 与 30 FPS 感知周期匹配的机械臂目标发送间隔 |
| `hand_transport` | `brainco-rs485` | 灵巧手使用强脑官方 RS485 SDK |
| `hand_rs485_port` | `auto` | 自动检测灵巧手串口 |
| `hand_rs485_slave_id` | `0` | 自动使用检测到的从站 ID |
| `hand_rs485_duration_ms` | `40~60` | 灵巧手动作时长，越小响应越快 |
| `hand_bend_deadzone` | `0.10~0.20` | 手指弯曲死区 |
| `hand_send_hz` | `35~40` | 灵巧手命令发送频率 |

## 推荐调参方向

想让机械臂更跟手：

```bash
-p arm_command_interval_s:=0.03
-p arm_motion_unit_m:=0.002
-p max_arm_target_delta_m:=0.008
-p min_command_delta_m:=0.002
```

想让灵巧手更快：

```bash
-p hand_rs485_duration_ms:=40
-p hand_send_hz:=40
-p hand_smoothing:=0.75
```

想让灵巧手更稳：

```bash
-p hand_bend_deadzone:=0.18
-p hand_spike_confirm_frames:=5
-p hand_closure_confirm_frames:=3
```

## 项目结论

这个项目的核心经验是：**不要让机械臂和灵巧手在同一条通信链路上抢资源**。

推荐最终方案：

- RM65-B 机械臂：RealMan CANFD
- BrainCo Revo2 灵巧手：BrainCo 官方 RS485 SDK
- Femto Mega：RGB-D 视觉识别
- Jetson：统一调度与实时控制

这样机械臂可以保持 CANFD 的丝滑跟随，灵巧手也能通过独立 RS485 稳定响应。各走各的路，大家都体面。
