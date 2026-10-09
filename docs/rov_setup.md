# 水下遥控载具 (ROV) 设置与验证指南

本文档介绍了如何在 Air 中配置、构建并运行水下遥控载具 (remotely operated vehicle, ROV) 仿真，以及如何利用 Python API 验证自动控制功能。

---

## 1. 概述与物理模型

AirSim 为水下及海洋机器人（ROV/AUV）提供载具支持，其设计灵感源自 [UNav-Sim](https://github.com/open-airlab/UNav-Sim) 动力学模型（默认采用 BlueROV2 Heavy 8 推进器架构）：

* **流体动力学与物理模型**: 在 `AirLib/include/vehicles/rov/` 中实现，提供完整的六自由度（6-DOF）海洋动力学特性：

    * 流体浮力（根据水密度 \( \rho = 1028\,\text{kg/m}^3$ \) 和载具排水体积计算得出）；
    * 非线性流体阻力、二次阻尼及回复力矩（浮心与重心的相对位置）；
    * 基于推进器混合矩阵的8个推进器空间力分配。
    
* **固件 (`rov_simple`)**: 提供机载深度保持、姿态调平及机体坐标系下的速度控制功能。

* **虚幻引擎渲染**: 通过原生 C++ 类 `ARovPawn` 集成，支持 5 个机载摄像头挂载点（右前、左前、前中、后中、底部）以及推进器的动态旋转效果。

---

## 2. 先决条件

* **操作系统**: Windows 10/11 64 位 或 Linux (Ubuntu 20.04 / 22.04)
* **虚幻引擎**: [Engine](https://github.com/OpenHUTB/engine)
* **编译器工具链**:
  * Windows: Visual Studio 2019 (MSVC v142 C++ 构建工具)
  * Linux: GCC 9 / Clang 11+, CMake 3.19+
* **Python 环境**: Python 3.8 ~ 3.11 (`msgpack-rpc-python`, `numpy`)

---

## 3. 配置 (`settings.json`)

若要以 ROV 模式运行 AirSim，请相应地配置 `settings.json` 文件。

### 3.1 配置示例
```json
{
  "SeeDocsAt": "https://github.com/Microsoft/AirSim/blob/master/docs/settings.md",
  "SettingsVersion": 1.2,
  "SimMode": "Rov",
  "ClockSpeed": 1,
  "PawnPaths": {
    "DefaultQuadrotor": {"PawnBP": "Class'/AirSim/Blueprints/BP_FlyingPawn.BP_FlyingPawn_C'"},
    "DefaultRov": {"PawnBP": "Class'/Script/AirSim.RovPawn'"},
    "DefaultComputerVision": {"PawnBP": "Class'/AirSim/Blueprints/BP_ComputerVisionPawn.BP_ComputerVisionPawn_C'"}
  },
  "Vehicles": {
    "RovSimple": {
      "VehicleType": "RovSimple",
      "DefaultVehicleState": "Armed",
      "PawnPath": "DefaultRov",
      "EnableCollisions": true,
      "AllowAPIAlways": true,
      "RC": {
        "RemoteControlID": 0,
        "AllowAPIWhenDisconnected": false
      },
      "Cameras": {
        "front_center_custom": {
          "CaptureSettings": [
            {
              "PublishToRos": 1,
              "ImageType": 0,
              "FOV_Degrees": 90,
              "Width": 1280,
              "Height": 720
            }
          ],
          "X": 0.50, "Y": 0.00, "Z": 0.00,
          "Pitch": 0.0, "Roll": 0.0, "Yaw": 0.0
        }
      }
    }
  }
}
```

### 3.2 配置文件位置

AirSim 按以下顺序查找 `settings.json`：

1. 命令行参数: `-settings="path/to/settings.json"`
2. Unreal 项目根目录 (例如：[Unreal/CarlaUE4/settings.json](https://github.com/OpenHUTB/hutb/blob/hutb/Unreal/CarlaUE4/settings.json)、[Unreal/Environments/Blocks/settings.json](https://github.com/OpenHUTB/air/blob/main/Unreal/Environments/Blocks/settings.json))
3. 用户文档目录: Windows 系统上为 `~/Documents/AirSim/settings.json` (`C:\Users\<username>\Documents\AirSim\settings.json`)

---

## 4. 构建与运行（Blocks 项目）

以内置的 `Blocks` 项目为例：

### 步骤 1：构建插件


在 [air](https://github.com/OpenHUTB/air) 根目录下：

* **Windows**:
  ```cmd
  build.cmd
  ```
* 将插件同步到 `Blocks` 环境：
  ```cmd
  cd Unreal\Environments\Blocks
  update_from_git.bat
  ```

### 步骤 2：启动仿真
* 在[虚幻引擎编辑器](https://github.com/OpenHUTB/engine)中打开 `Unreal/Environments/Blocks/Blocks.uproject`；
* 点击编辑器工具栏中的 **Play**（运行）按钮；
* The ROV will spawn in the viewport. The log outputs:
  ```text
  LogTemp: StartupModule: AirSim plugin
  LogTemp: ARovPawn: Loaded ROV mesh from /AirSim/Models/RoV/...
  LogTemp: RovSimple
  LogTemp: SimModeWorldRov: Api server started on port 41451
  ```

---

## 5. Python API 验证

在仿真运行期间：

### 步骤 1：安装 Python 客户端
```bash
cd PythonClient
pip install -e .
```

### 步骤 2：运行验证脚本
```bash
python PythonClient/rov/hello_rov.py
```

### 步骤 3：预期输出
```text
Connecting to AirSim ROV...
Connected!
Client Ver:1 (Min Req: 1)
Server Ver:1 (Min Req: 1)

Arming the ROV...

ROV State:
  Kinematics Position: x=0.00, y=0.00, z=0.00
  Kinematics Orientation: w=1.00, x=0.00, y=0.00, z=0.00
  Linear Velocity: vx=0.00, vy=0.00, vz=0.00

IMU Data:
  Linear Acceleration: (0.00, 0.00, -9.81) m/s^2
  Angular Velocity: (0.00, 0.00, 0.00) rad/s

Barometer (Depth) Data:
  Altitude: 0.00 m
  Pressure: 101325.0 Pa

Magnetometer Data:
  Magnetic Field: (0.21, 0.02, 0.42) Gauss

Moving ROV by velocity in body frame (forward 0.5 m/s, dive 0.2 m/s for 3s)...
Updated ROV Position: x=1.48, y=0.00, z=0.59

Hovering...
Disarming and resetting...
Done!
```

---

## 6. 故障排查

1. **端口 41451 连接被拒绝**:
   - 确保 Unreal 仿真正在运行（处于“Play”模式）。
   - 检查 `settings.json` 中是否指定了 `"SimMode": "Rov"`。
2. **载具未生成或网格（mesh）缺失**:
   - 确认 `settings.json` 中的 `PawnPaths` 包含 `"DefaultRov": {"PawnBP": "Class'/Script/AirSim.RovPawn'"}`。
