# 面向自主系统的统一空地仿真

![](../images/dev/CarlaAir.gif)

---


## 1. 引言和动机

### 1.1 问题

自主系统研究日益需要**联合空地仿真**——即无人机（UAV）与地面车辆/行人在共享的城市环境中共存和交互的场景。应用领域包括：

- **对自主交通进行空中监视**: 无人机监控和跟踪地面车辆
- **协同感知**: 无人机提供鸟瞰视角，以补充车辆级感知
- **社交导航**: 行人、车辆和无人机共享城市空间
- **多模态数据集生成**: 同步地面和空中传感器数据
- **搜救**: 无人机侦察，地面机器人穿梭于碎片之中


然而，这两个主流的开源模拟器服务于不同的领域：

- **Carla** (英特尔实验室，巴塞罗那自治大学CVC): 高保真自动驾驶模拟器，包含车辆、行人、交通管理、10多种传感器类型和 OpenDRIVE 道路网络——但**不支持原生无人机**。 
- **AirSim** (微软): 无人机和汽车模拟，具备基于物理的飞行动力学、空中传感器和多种飞行控制器后端——但**不包含交通管理、行人AI或城市道路网络**。

将它们作为**单独的进程**运行，并进行进程间同步是不切实际的：没有共享的物理引擎，没有共享的渲染引擎，没有坐标系对齐，而且由于复制了 UE4 渲染管线，会造成严重的性能开销。

### 1.2 解决方案

该解决方案将 Carla 0.9.16 和 AirSim 1.8.1 整合到**一个虚幻引擎进程**中，使得两个 Python API（`carla.Client` 位于 2000 端口，`airsim.MultirotorClient` 位于 41451 端口）能够同时在同一个模拟世界中运行。该系统**无需对面向用户的 Python API 进行任何修改**——现有的 Carla 和 AirSim 脚本无需修改即可正常工作。

### 1.3 主要贡献

1. **统一游戏模式(GameMode)架构**: 一种新颖的单继承+组合模式（`ASimWorldGameMode`），解决了UE4每个世界只能有一个游戏模式的基本限制。
2. **双 API 服务器共存**: 两个独立的 RPC 服务器（Carla rpclib + AirSim rpclib）在同一进程内运行于不同的端口。
3. **观察者棋子(Spectator Pawn)解耦**: 一种在满足 Carla 观察者要求的同时保留 AirSim 棋子占有模型的技术
4. **全面的空地联动功能**: 超过 220 个 actor 蓝图、超过 16 种传感器类型、14 种天气预设、13 张地图以及基于物理的无人机飞行——所有这些都可同时访问。
5. **离屏数据集生成**: 结合地面和空中视角的无头多模态数据采集

---

## 2. 系统架构

### 2.1 核心挑战：UE4 的单一游戏模式限制

虚幻引擎 4 强制执行一项严格的架构限制：**每个世界（关卡）只能有一个游戏模式**。游戏模式控制着基本的模拟生命周期——玩家生成、游戏规则、比赛状态和初始化顺序。

- Carla 使用 `ACarlaGameModeBase` 作为其游戏模式，该模式初始化了Episode 系统、天气系统、交通灯管理器、Actor 工厂、记录器和 RPC 服务器。
- AirSim 使用 `AAirSimGameMode` 作为其游戏模式，该模式会初始化 SimHUD、读取 settings.json 文件并生成 SimMode 参与者。


这两种游戏模式无法通过标准的UE4机制**共存**。之前的各种方法（例如，关卡流式传输、仅插件集成）都失败了，因为这两个系统都假定游戏模式关卡(GameMode-level)对模拟生命周期拥有控制权。


### 2.2 解决方案：单一继承 + 组合模式

![](../images/calar_air/resolving_single_mode.png)

该解决方案利用了这两个系统之间的架构不对称性：

- Carla 的子系统（Episode、天气、交通管理器、Actor 工厂）通过继承和友元(`friend`)声明与 `ACarlaGameModeBase` **紧密耦合**。
- AirSim 的模拟逻辑位于 `ASimModeBase` 中，它继承自 `AActor`（而非 `AGameModeBase`），因此可以作为常规世界 Actor 生成。

这里使用统一的 `ASimWorldGameMode` 类：

```
                    UE4 单游戏模式(GameMode)插槽
                            |
                    AGameModeBase (UE4)
                            |
                    ACarlaGameModeBase          ← 通过继承的 Carla 子系统
                            |
                    ASimWorldGameMode           ← 占用游戏模式(GameMode)插槽
                            |
                            |--- [拥有] ASimModeBase (由 AActor 生成)  ← AirSim 通过组合
                            |              |
                            |              |--- FastPhysicsEngine (333 Hz 异步线程)
                            |              |--- MultirotorRpcLibServer (端口 41451)
                            |              |--- AFlyingPawn (无人机)
                            |              |--- ApiProvider (载具 APIs)
                            |
                            |--- [继承] UCarlaEpisode
                            |--- [继承] AWeather
                            |--- [继承] ACarlaRecorder
                            |--- [继承] ATrafficLightManager
                            |--- [继承] ActorDispatcher + ActorFactories
```

!!! 关键思想
    通过让 `ASimWorldGameMode` **继承**自 `ACarlaGameModeBase`，可以让所有 Carla 子系统都经历标准的 UE4 生命周期（`InitGame` → `BeginPlay` → `Tick`）。在 Carla 初始化完成后，AirSim 的 `ASimModeBase` 会在 `BeginPlay` 期间作为**普通 Actor 生成**到游戏世界中。这样就避免了任何插槽冲突，因为 `ASimModeBase` 不会与 GameMode 插槽竞争。

### 2.3 类层次结构

```cpp
// 统一游戏模式 GameMode（在 AirSim 插件模块中，依赖于 Carla 模块）
UCLASS()
class AIRSIM_API ASimWorldGameMode : public ACarlaGameModeBase {
    // Carla: 继承了 Episode, 天气Weather, 记录器Recorder, ActorFactories, 交通 Traffic
    // AirSim: 包含 SimMode, HUD 空间, 输入绑定
private:
    UPROPERTY() ASimModeBase* SimMode_;        // AirSim 仿真 (生成参与者)
    UPROPERTY() USimHUDWidget* AirSimWidget_;  // 调试叠加层
    TSubclassOf<ASimModeBase> SimModeClass_;   // 来自 settings.json
};
```

### 2.4 初始化序列

```
UE4 引擎开始
│
├── ASimWorldGameMode 构造函数
│   ├── ACarlaGameModeBase() → 创建 Episode, Recorder
│   ├── DefaultPawnClass = nullptr  ← 对 AirSim 至关重要
│   ├── 加载 BP_Weather 蓝图类
│   ├── 注册 8 个参与者工厂 (5 个 C++ 和 3 个蓝图)
│   │   ├── ASensorFactory, AStaticMeshFactory, ATriggerFactory
│   │   ├── AUtilActorFactory, AAIControllerFactory
│   │   ├── BP VehicleFactory, WalkerFactory, PropFactory
│   └── 初始化 AirSim logger 和 ImageWrapper 模块
│
├── ACarlaGameModeBase::InitGame()
│   ├── 生成 Weather 参与者 (从 BP_Weather)
│   ├── 使用 Episode 生成并注册所有 ActorFactories
│   ├── 解析 OpenDRIVE 路网 (.xodr)
│   ├── 根据道路拓扑结构生成生成点
│   └── 初始化 CarlaGameInstance → FCarlaEngine → 开始 Carla RPC (port 2000)
│
├── ASimWorldGameMode::BeginPlay()
│   ├── 阶段 1: ACarlaGameModeBase::BeginPlay()
│   │   ├── 对世界所有参与者进行语义标记
│   │   ├── 交通灯管理器初始化
│   │   ├── Episode::InitializeAtBeginPlay()
│   │   ├── 应用默认天气参数
│   │   ├── 开始记录器/重放器(Recorder/Replayer)
│   │   └── 注册环境对象
│   │
│   ├── 阶段 2: 创建观察者棋子
│   │   ├── 生成 ASpectatorPawn (未占有)
│   │   ├── 注册成为 Episode->Spectator (友元访问权限)
│   │   └── 在 ActorDispatcher 注册表中注册
│   │
│   └── 阶段 3: AirSim Bootstrap
│       ├── InitializeAirSimSettings() → 读取配置文件 settings.json
│       ├── SetUnrealEngineSettings() → 禁用运动模糊，自定义景深
│       ├── CreateSimMode() → SpawnActor<ASimModeWorldMultiRotor>()
│       │   └── ASimModeBase::BeginPlay()
│       │       ├── NED 坐标系设置
│       │       ├── 创建 ApiProvider 和 WorldSimApi
│       │       ├── 生成 BP_FlyingPawn (无人机棋子)
│       │       ├── 创建 MultirotorPawnSimApi
│       │       │   ├── MultiRotorPhysicsBody (4 个旋翼 + 6 阻力面)
│       │       │   ├── SimpleFlight 控制器
│       │       │   └── 传感器: IMU, Barometer, GPS, Magnetometer
│       │       ├── 创建 FastPhysicsEngine
│       │       └──以 333 赫兹开始异步物理线程
│       ├── CreateAirSimWidget() → UMG 调试覆盖
│       ├── SetupAirSimInputBindings()
│       └── SimMode_->startApiServer() → 端口 41451
│
└── 两个API均已准备就绪
    ├── Carla Python API 在端口 2000 上运行
    └── AirSim Python API 在端口 41451 上运行
```

### 2.5 观察者棋子问题

在 AirSim 中，**必须**将 `DefaultPawnClass` 设置为 `nullptr`，因为 UE4 的默认行为是自动生成并拥有一个玩家控制器控制的棋子。如果任一一个棋子被控制，AirSim 的 `SimpleFlightApi` 输入管道就会中断，那么无人机将无法接收控制命令。

但是，Carla 希望 `Episode->Spectator` 指向一个有效的棋子（用于相机控制、场景观察，以及作为 Python API 的 `world.get_spectator()` 返回值）。

**解决方案**：生成一个观察者棋子(`ASpectatorPawn`)对象，但**不拥有它的控制权**。它作为一个“孤儿” 棋子存在（Carla 的 API 可以访问它），但 UE4 的玩家控制器系统无法访问它。我们使用 C++ 友元类(`friend class`)声明直接设置 `Episode->Spectator`（一个私有成员），绕过了 Carla 的正常初始化流程。

```cpp
// 在 ASimWorldGameMode::BeginPlay() 中，在 Super::BeginPlay() 之后
FActorSpawnParameters SpawnParams;
SpawnParams.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AlwaysSpawn;
ASpectatorPawn* Spectator = GetWorld()->SpawnActor<ASpectatorPawn>(
    ASpectatorPawn::StaticClass(), &SpawnTransform, SpawnParams);
// 不要调用 PlayerController->Possess(Spectator)
Episode->Spectator = Spectator;  // 直接友元访问
Episode->ActorDispatcher->GetActorRegistry().Register(
    FCarlaActor::IdType(Spectator->GetUniqueID()), *Spectator);
```

### 2.6 构建系统集成

这两个插件是作为同一个UE4项目的一部分编译的：

```
CarlaUE4.uproject
├── Plugins/Carla/    (LoadingPhase: PostConfigInit)
│   └── Carla.Build.cs  → libcarla_server.a, librpc.a
└── Plugins/AirSim/   (LoadingPhase: PreDefault)
    └── AirSim.Build.cs → PrivateDependency: "Carla"
                         → AirLib (header-only), rpclib, Eigen3, MavLinkCom
```

**单向依赖关系**（AirSim 依赖 Carla：AirSim → Carla）至关重要：`AirSim.Build.cs` 声明了 `PrivateDependencyModuleNames.Add("Carla")`，允许 `SimWorldGameMode.cpp` 包含 Carla 头文件。反向依赖关系不存在，即 Carla 的代码无法识别 AirSim。Carla 源代码仅做了两处修改：

1. `CarlaGameModeBase.h`: `WeatherClass` 和 `ActorFactories` 从私有的成员 `private` 更改为受保护的成员 `protected`
2. `CarlaEpisode.h`: 添加 `friend class ASimWorldGameMode`


### 2.7 配置

**`Unreal/CarlaUE4/Config/DefaultEngine.ini`** (关键条目):
```ini
GlobalDefaultGameMode=/Script/AirSim.SimWorldGameMode
GameInstanceClass=/Script/Carla.CarlaGameInstance
TransitionMap=/AirSim/AirSimAssets
r.CustomDepth=3           ; 语义分割所需
bEnableEnhancedDeterminism=True
```

**`Unreal/CarlaUE4/Config/DefaultGame.ini`** (打包):
```ini
; 13 Carla 地图 + AirSim 资产地图
+MapsToCook=(FilePath="/Game/Carla/Maps/Town01")
+MapsToCook=(FilePath="/AirSim/AirSimAssets")
; 6 AirSim 烘焙内容目
+DirectoriesToAlwaysCook=(Path="AirSim/Blueprints")
+DirectoriesToAlwaysCook=(Path="AirSim/Models")
+DirectoriesToAlwaysCook=(Path="AirSim/Weather")
```

**`~/Documents/AirSim/settings.json`** (运行时):
```json
{
    "SettingsVersion": 1.2,
    "SimMode": "Multirotor",
    "Vehicles": {
        "SimpleFlight": {
            "VehicleType": "SimpleFlight",
            "AutoCreate": true,
            "Cameras": {
                "0": { "CaptureSettings": [{"ImageType": 0, "Width": 1280, "Height": 960}] }
            }
        }
    }
}
```

---

## 3. 双通信架构

### 3.1 Carla 服务端 (端口 2000)

Carla 是用 **三端口架构**：

| 端口 | 协议 | 目的 |
|------|----------|---------|
| 2000 | rpclib (MessagePack-RPC) | 同步命令: spawn, destroy, weather, tick |
| 2001 | TCP 流式传输 | 高通量传感器数据（相机图像、激光雷达） |
| 2002 | TCP 辅助 | 多 GPU 同步 |

**RPC绑定模式**: 修改 UE4 状态的操作通过`boost::asio::io_context` 的 post-and-wait 机制同步绑定(**sync-bound**)到游戏线程。只读操作异步绑定(**async-bound**)到 RPC 工作线程。

**传感器数据管线**:

1. 传感器在 UE4 游戏线程节拍周期内捕获数据
2. `FPixelReader::SendPixelsInRenderThread()` 将渲染命令加入渲染命令队列
3. GPU 像素通过 `FRHICommandListImmediate` 读取
4. 数据通过 `SensorRegistry::Serialize()`（类型分发）进行序列化。
5. 通过专用 TCP 流发送（每个传感器都有一个唯一的流令牌）

### 3.2 AirSim 服务端 (端口 41451)

AirSim 使用 rpclib 实现的**单端口 RPC 架构**：

| 端口 | 协议 | 目的 |
|------|----------|---------|
| 41451 | rpclib (MessagePack-RPC) | 所有指令：飞行控制、图像采集、传感器查询 |

**图像采集**: 与 Carla 的流式传输模型不同，AirSim 图像是通过 RPC 调用（`simGetImages`）按需请求的。服务器同步读取 UE4 渲染目标，并在 RPC 响应中返回原始像素数据。

**物理线程交互**: 飞行指令（例如 `moveToPositionAsync`）被发布到 AirSim 的内部指令队列。运行在专用异步线程上、频率为 333 Hz 的 `FastPhysicsEngine` 读取指令并更新物理状态。RPC 服务器按需读取状态。

### 3.3 共存

这两个 RPC 服务器运行在同一个进程中，共享同一个 UE4 世界。它们之间不会相互干扰，因为：

- 它们绑定到不同的端口（2000 与 41451）

- 它们使用独立的 rpclib 服务器实例。

- UE4 的单线程游戏循环自然地实现了游戏线程操作的串行化。

- AirSim 的异步物理线程运行在其自身的运动学状态上，读取 UE4 碰撞几何​​体，但不会修改 Carla 的参与者。

---

## 4. 传感器系统

### 4.1 Carla 地面传感器 (10 种类型)

Carla 提供了一套全面的真实载具传感器套件，所有传感器都可以作为 Actor 附加到任何车辆上：

| 传感器 | 类层次结构 | 数据格式 | 关键参数 |
|--------|----------------|-------------|----------------|
| RGB 相机 | `ASceneCaptureCamera` : `ASceneCaptureSensor` : `ASensor` | BGRA uint8 | 分辨率、视场角、镜头畸变 |
| 深度相机 | `ADepthCamera` : `AShaderBasedSensor` : `ASceneCaptureSensor` | 编码格式为 R+G×256+B×65536, 以米为单位进行标准化 | 与 RGB 相同 |
| 语义分割 | `ASemanticSegmentationCamera` : `AShaderBasedSensor` | R 通道中的标签索引（20 多个与 Cityscapes 兼容的类） | 与 RGB 相同 |
| 实例分割 | `AInstanceSegmentationCamera` : `AShaderBasedSensor` | 每个实例的唯一颜色 | 与 RGB 相同 |
| 激光雷达 | `ARayCastLidar` : `ARayCastSemanticLidar` : `ASensor` | 每个点为 float32 [x, y, z, 强度] | 通道数（16-128），范围，点数/秒，旋转频率 |
| 毫米波雷达 | `ARadar` : `ASensor` | 每次探测结果：方位角、高度角、深度、速度 | 水平视场角、垂直视场角、范围、点数/秒 |
| IMU | `AInertialMeasurementUnit` : `ASensor` | 加速度计 (3D), 陀螺仪 (3D), 罗盘 | 噪声标准差，偏差 |
| GNSS | `AGnssSensor` : `ASensor` | 纬度、经度、海拔 | 噪声偏差 |
| 碰撞 | `ACollisionSensor` : `ASensor` | 事件驱动：其他参与者，脉冲 | -- |
| 压线 | `ALaneInvasionSensor` : `ASensor` | 事件驱动：和车道线交叉 | -- |

**基于着色器的渲染**: 深度、分割、法线和光流像机将后处理材质应用于 UE4 的场景捕获管线。材质写入几何 (Geometry, G) 缓冲区通道，然后作为像素数据读取。这以极低的额外渲染成本实现了与 RGB 图像的像素级完美对齐。

**激光雷达实现**: 使用 UE4 `LineTraceSingleByChannel` 进行光线投射。点云以旋转模式生成，与真实的激光雷达扫描模式相匹配。强度通过衰减模型 `α × I₀ + β` 计算，其中 I₀ 取决于探测距离和材质反射率。支持雨雾等大气衰减。

### 4.2 AirSim 空中传感器（6 种类型）

| 传感器 | 实现 | 数据格式 | 噪声模型 |
|--------|---------------|-------------|-------------|
| RGB 相机 | `APIPCamera` + `USceneCaptureComponent2D` | uint8 BGRA | -- |
| Depth 相机 | 同一组件上的后处理材料 | float32（米）或 uint8（可视化） | -- |
| 分割 | 同一组件上的后处理材料 | uint8 标签颜色 | -- |
| IMU | `ImuSimple` (AirLib) | 加速度计、陀螺仪、指南针 | 角度随机游走、速度随机游走、高斯-马尔可夫偏差漂移 |
| GPS | `GpsSimple` (AirLib) | 纬度、经度、海拔、速度 | EPH/EPV 收敛滤波器、更新速率限制、延迟 |
| 气压计 | `BarometerSimple` (AirLib) | 海拔（米），气压（帕） | 高斯-马尔可夫漂移（~10米/小时），不相关噪声（~0.2米σ） |
| 磁力计 | `MagnetometerSimple` (AirLib) | 体坐标系磁场（三维） | 高斯噪声 + 均匀偏差 |

**相机系统**: 每架无人机有 5 个相机安装点（前中心、前右、前左、下中心、后中心）。每个安装点都会创建多个 `USceneCaptureComponent2D` 实例，用于拍摄不同类型的图像。图像通过 RPC API 按需采集。默认分辨率：1280×960。

**传感器噪声模型**: AirSim 实现了基于 Oliver Woodman 的《惯性导航导论》（剑桥 TR-696）的物理噪声模型：

- **IMU**: 采用角度随机游走 (angle random walk, ARW) 算法校正陀螺仪漂移，采用速度随机游走 (velocity random walk, VRW) 算法校正加速度计偏差，并可配置开启偏差。

- **GPS**: 用于水平/垂直位置误差收敛的一阶低通滤波器，模拟冷启动行为

- **气压计**: 采用高斯-马尔可夫过程模拟缓慢的压力漂移，并考虑高频测量噪声

### 4.3 组合传感器能力

在SimWorld中，**所有16种传感器类型可以同时运行**：

| 角度 | 可用传感器 | 流的总数 |
|-------------|-------------------|---------------|
| 地面车辆 | RGB、深度、分割、实例分割、激光雷达、雷达、IMU、GNSS、碰撞、 压线 | 10 |
| 无人机 | RGB、深度、分割、红外、IMU、GPS、气压计、磁力计 | 8 |
| **组合** | **以上所有内容均与同一世界状态同步** | **18** |

这是一项独特的功能，因为没有其他模拟器能够在一次渲染过程中提供同步的地面+空中多模态传感器数据。


---

## 5. 物理学和飞行动力学

### 5.1 地面车辆物理

Carla 使用 UE4 的原生 PhysX 车辆物理引擎（`AWheeledVehicle`）：

- 4轮（标准轿车）和N轮（卡车、公共汽车）配置

- 扭矩曲线、转向曲线、质量、阻力、质心调校

- 采用PID控制器的阿克曼转向模型

- 与 Project Chrono 集成，实现更高保真度的轮胎模型（可选）

### 5.2 无人机飞行动力学 (FastPhysicsEngine)

AirSim 实现了一个**自定义的刚体动力学引擎**，该引擎运行在专用的异步线程上，频率为 333 Hz：

**积分**: 采用二阶 Verlet 方法计算线运动和角运动：
$$
v(t+dt) = v(t) + a(t) \times dt
$$
$$
x(t+dt) = x(t) + v(t+dt) \times dt
$$

**力模型**: 每个旋翼贡献一个推力矢量和一个扭矩。总扭矩是四个旋翼扭矩之和。空气动力阻力采用六面盒模型，其与速度的平方成正比：
$$
F_{drag} = -0.5 \times \rho \times Cd \times A \times |v_{rel}|^2 \times \hat{v}_{rel}
$$
其中 \(\hat{v}_{rel}\) 考虑风的影响。

**碰撞响应**: 基于库仑摩擦的脉冲法：
$$
j = - \frac{(1 + e) \times (v \cdot n)}{ \frac{1}{m} + ((I^{-1}(r \times n)) × r) \cdot n}
$$
地面锁定功能可防止飞机在着陆面上发生微小弹跳。

**飞控**:

- **SimpleFlight** (默认): 内置 PID 控制器，用于位置、速度和姿态控制。运行于AirLib库内，无外部依赖。

- **PX4** (可选): 通过 MAVLink 与 PX4 自动驾驶仪实现软件在环集成

- **ArduPilot** (可选): SITL集成

### 5.3 坐标系

| 系统 | 约定 | 原点 | 单位 |
|--------|-----------|--------|-------|
| UE4 / Carla | 左手系，Z 轴向上 | 地图原点 (0, 0, 0) | 厘米（内部单位），米（API单位） |
| AirSim | 北东地（North-East-Down, NED） | 玩家起始位置 | 米 |

**坐标映射**: AirSim 在 BeginPlay 时创建一个 `NedTransform`，用于在 UE4 世界坐标和以无人机生成点为锚点的 NED 坐标系之间进行映射。对于无人机跟随车辆的应用，我们使用**相对偏移方法**：

1. 记录 Carla 汽车的初始位置 \( (cx_o, cy_o) \) 和 AirSim 无人机 NED 的位置 \( (dx_o, dy_o) \)

2. 每帧: \( \Delta = (cx - cx_o, cy - cy_o) \) ，目标无人机 NED = \( (dx_o + \Delta x, dy_o, \Delta y) \) 。

---

## 6. 交通与环境

### 6.1 交通管理器

Carla 的流量管理器是一个运行在专用线程上的**客户端管线架构**：

| 阶段 | 功能 |
|-------|----------|
| ALSM | 参与者生命周期和状态管理(Actor Lifecycle and State Management, ALSM)——同步模拟状态 |
| 定位 | 航点视界维护，变道决策 |
| 碰撞 | 基于多边形的碰撞预测 (boost::geometry) |
| 交通灯 | 信号遵守逻辑 |
| 运动规划 | PID控制器：分隔的城市/高速公路参数 |
| 车灯 | 自动前大灯/刹车灯管理 |

交通管理器维护一个用于 O(1) 空间查询的 **内存地图(InMemoryMap)**（缓存的路径点图）和一个跟踪所有受管参与者运动学的 **模拟状态(SimulationState)**。


### 6.2 天气系统

控制大气条件的 15 个连续参数：

| 参数 | 范围 | 效果 |
|-----------|-------|--------|
| Cloudiness | 0-100 | 云覆盖密度 |
| Precipitation | 0-100 | 降雨强度 |
| PrecipitationDeposits | 0-100 | 水坑积水 |
| WindIntensity | 0-100 | 风速 |
| SunAzimuthAngle | 0-360 | 太阳水平角 |
| SunAltitudeAngle | -90 to 90 | 太阳高度角（负值 = 夜晚） |
| FogDensity | 0-100 | 体积雾厚度 |
| FogDistance | 0+ | 雾起始距离（米） |
| Wetness | 0-100 | 表面湿度（反射） |
| DustStorm | 0-100 | 沙尘暴/沙暴强度 |

4 个预设模式，涵盖白天/夜晚、晴天/阴天/雨天/暴风雨等天气状况。天气会**同时**影响 Carla 渲染和 AirSim 的航拍视图。


### 6.3 地图和路网

| 地图 | 描述 | 生成点 | 面积 |
|-----|-------------|--------------|------|
| Town01 | 小镇，T 字路口 | 252 | ~1 km² |
| Town02 | 居住区 | ~100 | ~0.5 km² |
| Town03 | 城市中心区，高速公路匝道 | 265 | ~2 km² |
| Town04 | 高速公路旁的小镇 | ~300 | ~5 km² |
| Town05 | 城市网格，多车道道路 | 302 | ~3 km² |
| Town10HD | 高清市中心，建筑细节丰富 | 155 | ~1.5 km² |

每张地图包含：

- **OpenDRIVE** (.xodr): 完整的道路网络定义，包括车道几何形状、交叉口和信号灯。

- **导航网格**: 重新生成的行人步行区域

- **交通基础设施**: 交通信号灯（带定时）、停车标志、限速标志

- **环境对象**: 建筑物、植被、街道设施（Town10HD 中包含 13000 多个对象）

---

## 7. 参与者系统

### 7.1 参与者蓝图库

| 类别 | 数量 | 例子 |
|----------|-------|---------|
| 载具 | 41 | 特斯拉 Model 3、宝马 Gran Tourer、奥迪 A2、巴士、卡车、摩托车、自行车 |
| 行人 | 52 | 男女成年人、儿童，穿着各式各样的服装和体型 |
| 传感器 | 25 | 第 4 节中列出的所有传感器类型 |
| 道具 | 99 | 围栏、集装箱、垃圾桶、摊位、长椅 |
| 其他 | 3 | 控制器类型 |
| **Total** | **220** | |

### 7.2 参与者生成管线

```cpp
Python: world.spawn_actor(blueprint, transform)
  → RPC: carla::rpc::Command::SpawnActor
    → FCarlaServer::FPimpl (game thread)
      → UCarlaEpisode::SpawnActorWithInfo()
        → UActorDispatcher::SpawnActor()
          → ACarlaActorFactory::SpawnActor()  (type-specific factory)
            → UWorld::SpawnActor<>()  (UE4 native spawn)
          → FActorRegistry::Register(actor_id, actor)
        → return carla::rpc::Actor (serialized to client)
```

### 7.3 行人 AI 导航

行人使用 UE4 的 **Recast 导航网格**进行寻路

- 导航网格由地图几何体预先构建

- `AWalkerAIController` 提供 Python API 接口

- `controller.go_to_location(target)` 触发导航网格上的 A* 寻路算法

- `controller.set_max_speed(speed)` 控制移动速度

---

## 8. 应用领域和研究能力

### 8.1 自动驾驶

- 完整的自车传感器套件（RGB、深度、分割、激光雷达、雷达、IMU、GNSS）
- 具备车辆行为参数的交通管理器
- 配备 PID 控制器的阿克曼转向模型
- 涵盖 6 张以上地图的 41 种车辆类型，并采用 OpenDRIVE 路网
- 用于可复现实验的确定性仿真模式

### 8.2 无人机导航与控制

- 六自由度飞行控制：位置、速度、机体坐标系速度、偏航率、路径跟踪
- 基于物理的动力学模型，频率为 333 Hz，考虑风阻和空气动力阻力
- 多摄像头空中感知（每架无人机 5 个安装点）
- 可配置的飞行控制器（SimpleFlight、PX4、ArduPilot）
- 传感器噪声模型（IMU、GPS、气压计、磁力计），实现逼真的感知

### 8.3 空地协同系统 (新颖)

这是 SimWorld 独有的研究能力：

- **无人机-车辆跟踪**: 无人机通过实时坐标变换跟踪地面车辆
- **空中交通监控**: 鸟瞰式车辆检测与计数
- **协同感知**: 地面+空中传感器融合数据集
- **多智能体协同**: 多辆 Carla 车辆 + AirSim 无人机在共享场景中
- **应急响应**: 城市环境中的空中侦察 + 地面导航

### 8.4 社交导航

- 第一人称射击游戏风格，采用鼠标视角的第一人称行人控制
- 52 种行人类型，配备 AI 控制的导航
- 大规模人群模拟（50辆以上车辆 + 20名以上行人）
- 交通环境中的行人-车辆交互
- 多变的天气和光照条件

### 8.5 多模态数据集生成

SimWorld 能够生成具有**前所未有的模态覆盖范围**的数据集：

| 模态 | 源 | 格式 | 分辨率 |
|----------|--------|--------|------------|
| 地面 RGB | Carla 相机 | PNG/JPG | 可配置（最高可达 4K） |
| 地面深度 | Carla 深度相机 | float32 NPY (米) | 与 RGB 相同 |
| 地面分隔 | Carla 分割相机 | uint8 NPY (20 多个类) | 与 RGB 相同 |
| 地面实例分割 | Carla 实例分割相机 | RGB (每个实例) | 与 RGB 相同 |
| 地面激光雷达 | Carla ray-cast | float32 NPY (x,y,z,强度) | 16-128 个通道 |
| 地面雷达 | Carla radar | float32 (方位角、深度、速度) | 可配置 |
| 地面 IMU | Carla IMU | JSON (加速度计、陀螺仪、指南针) | 100+ Hz |
| 地面 GNSS | Carla GNSS | JSON (纬度、经度、海拔) | 10+ Hz |
| 空中 RGB | AirSim camera | PNG/NPY | 1280×960 (可配置) |
| 空中深度 | AirSim depth | float32 NPY (米) | 1280×960 |
| 空中分隔 | AirSim seg | PNG (标签颜色) | 1280×960 |
| 空中 IMU | AirSim IMU | JSON (带噪音) | 333 Hz |
| 空中 GPS | AirSim GPS | JSON (with convergence) | 可配置 |
| 空中气压计 | AirSim barometer | JSON (海拔高度，气压) | 可配置 |
| 自车姿态 | Carla transform | JSON (x,y,z,俯仰,偏航,横滚) | 每帧 |
| 无人机姿态 | AirSim kinematics | JSON (NED 位置, 四元素) | 每帧 |

所有模态均**同步到同一仿真帧**并共享同一世界状态。离屏渲染模式（`-RenderOffScreen`）支持在 GPU 服务器上进行无头操作，以进行大规模数据处理。

---

## 9. ROS 集成

SimWorld 支持双 ROS2 桥接器，用于与机器人技术栈集成：

### 9.1 Carla ROS 桥

- 工作空间: `carla-ros-bridge` (ROS2 Humble)
- 已发布的主题：`/carla/world_info`, `/carla/status`, `/clock`, 每个车的传感器主体
- 服务：`/carla/spawn_object`, `/carla/destroy_object`, `/carla/get_blueprints`
- 命名空间：`/carla/*`

### 9.2 AirSim ROS 封装器

- 包：`airsim_ros_pkgs` (ROS2 Humble)
- 已发布的主题：`/airsim_node/SimpleFlight/imu/imu`, `/airsim_node/SimpleFlight/front_center/Scene`, GPS, barometer, magnetometer, odometry
- 服务: takeoff, land, reset
- 命名空间: `/airsim_node/*`

### 9.3 同时运行

这两个桥接器在同一个 ROS 主节点下同时运行，**不存在命名空间冲突**（`/carla/` 与 `/airsim_node/`）。这使得标准的 ROS2 工具（rviz2、rosbag2、nav2）能够同时使用地面和空中数据流。

---

## 10. 性能特征

### 10.1 系统要求

| 部件 | 最低配置 | 推荐配置 |
|-----------|---------|-------------|
| GPU | 4 GB VRAM (Vulkan) | 8+ GB VRAM (NVIDIA RTX) |
| 内存 | 16 GB | 32 GB |
| CPU | 4 核 | 8+ 核 |
| 硬盘 | 20 GB (打包) | 40 GB (源代码构建) |
| 操作系统 | Ubuntu 18.04+ | Ubuntu 20.04/22.04 |

### 10.2 显存使用 (在 RTX A4000, 16 GB 上测试)

| 场景 | 显存（Video RAM, VRAM）使用 |
|----------|------------|
| 空闲（Town10HD 已加载） | ~3700 MiB |
| + 3 辆车 + 2 个行人 + 无人机 | ~3870 MiB |
| 3 小时持续操作 | ~3878 MiB (稳定，无泄漏) |
| 地图加载期间的峰值 | ~5000 MiB |

### 10.3 稳定性

| 测试 | 持续时间 | 周期 | 错误 | 结果 |
|------|----------|--------|--------|--------|
| 持续生成/销毁 + 无人机 | 3 小时 | 357 个 | 0 | 通过 |
| 地图切换 (Town01→03→05) | ~15 min | 3 地图 | 0 | 通过 |
| 包含 89 项的综合 API 测试 | ~5 分钟 | 89 项 | 0 | 通过 |

### 10.4 已知局限性

| 局限 | 原因 | 变通方案 |
|-----------|-------|-----------|
| 配备无人机时自动驾驶车辆 ≤ 10 辆 | AirSim 物理 ↔ Carla 交通管理器四元数冲突 | 限制流量或禁用自动驾驶 |
| 地图切换延迟：2-5分钟 | Shipping 构建中的完全资产重新加载 | 使用 API 设置超时时间为 300 秒 |
| 单无人机实例 | AirSim settings.json 限制 | 可配置用于多无人机 |

---

## 11. 构建和部署

### 11.1 编译

```bash
# 模块级编译（避免 features.h 交叉编译错误）
export UE4_ROOT=/path/to/UnrealEngine-4.26
${UE4_ROOT}/Engine/Build/BatchFiles/Linux/Build.sh CarlaUE4Editor Linux Development \
    -project="/path/to/CarlaUE4.uproject" -module=Carla    # ~290s
${UE4_ROOT}/Engine/Build/BatchFiles/Linux/Build.sh CarlaUE4Editor Linux Development \
    -project="/path/to/CarlaUE4.uproject" -module=AirSim   # ~100s
```

### 11.2 打包

```bash
./Util/BuildTools/Package.sh --config=Shipping --no-zip
# 输出: Dist/CARLA_Shipping_<version>/LinuxNoEditor/ (~19 GB)
# 编译: ~800s (656 units), Cook: ~2h (13459 packages)
```

### 11.3 发布

打包后的版本：**7.3 GB**（压缩后，不含调试符号），包含：

- 适用于 Linux x86_64 的二进制文件

- 13 张已烘焙好的地图 + AirSim 资产

- 17 个 Python 脚本示例

- Carla PythonAPI 源代码

- AirSim 配置模板

---

## 12. 与相关工作的比较

| 特性 | Carla | AirSim | LGSVL | ISAAC Sim | **SimWorld** |
|---------|-------|--------|-------|-----------|-------------|
| 地面车辆 | ✅ 41 类 | ✅ Limited | ✅ | ✅ | ✅ 41 类 |
| 行人 | ✅ 52 类 | ❌ | ✅ | ✅ | ✅ 52 类 |
| 无人机 (基于物理) | ❌ | ✅ | ❌ | ❌ | ✅ |
| 交通管理器 | ✅ | ❌ | ✅ | ❌ | ✅ |
| OpenDRIVE 路网 | ✅ | ❌ | ✅ | ❌ | ✅ |
| 天气系统 | ✅ 15 个参数 | ✅ 静态 | ✅ | ✅ | ✅ 15 个参数 |
| 地面传感器 (10+) | ✅ | ❌ | ✅ | ✅ | ✅ |
| 空中传感器 (6+) | ❌ | ✅ | ❌ | ❌ | ✅ |
| 传感器噪声模型 | 基本 | ✅ 详细 | 基本 | ✅ | ✅ 详细 |
| ROS 整合 | ✅ | ✅ | ✅ | ✅ | ✅ 双桥 |
| 空地联合仿真 | ❌ | ❌ | ❌ | ❌ | ✅ |
| 单进程渲染 | N/A | N/A | N/A | N/A | ✅ |
| 开源 | ✅ MIT | ✅ MIT | ❌ 停止使用 | ❌ 专利 | ✅ |

**主要区别**：SimWorld 是唯一一个在单一渲染过程中提供基于物理的无人机飞行、全面的地面交通管理以及多模态同步空地传感器数据的平台。

---

## 13. 潜在的论文贡献

### 提交至 NeurIPS / ICLR

1. **系统论文**: 解决模拟器统一中单一游戏模式限制的新型架构
2. **基准论文**: 空地协同感知的新基准
3. **数据集论文**: 具有同步地面+空中标注的大规模多模态数据集
4. **应用论文**: 基于空地数据的空中辅助自动驾驶/社交导航

### 潜在评估实验

- **目标检测**: 比较仅使用地面数据和使用地面+航空数据的检测精度
- **轨迹预测**: 基于空中环境的行人/车辆轨迹预测
- **多模态融合**: 跨 16 种以上同步模态的传感器融合基准测试
- **仿真到现实迁移**: 从 SimWorld 合成数据到真实世界数据集的域自适应
- **协同感知**: 受通信约束的空地感知融合
- **社会导航**: 基于无人机存在的行人交互建模
- **基于强化学习的无人机控制**: 训练无人机策略，用于车辆跟踪、监视和搜索任务

### 在顶级场合发表的类似作品

- **Carla**: "CARLA: An Open Urban Driving Simulator" (CoRL 2017)
- **AirSim**: "AirSim: High-Fidelity Visual and Physical Simulation for Autonomous Vehicles" (FSR 2017)
- **Habitat**: "Habitat: A Platform for Embodied AI Research" (ICCV 2019)
- **ISAAC Sim**: "Isaac Sim: An Extensible Robotics Simulator" (ICRA 2022)
- **MetaDrive**: "MetaDrive: Composing Diverse Driving Scenarios for Generalizable RL" (TPAMI 2022)

SimWorld 填补了驾驶模拟器和航空模拟器之间的空白，在自动驾驶、无人机导航和多智能体系统交叉领域开辟了一类新的研究方向。

---

## 附录A：完整文件清单

### 核心架构文件

| 文件 | 描述 |
|------|-------------|
| [Plugins/AirSim/Source/SimWorldGameMode.h/.cpp](https://github.com/OpenHUTB/air/blob/main/Unreal/Plugins/AirSim/Source/SimWorldGameMode.cpp) | 统一的游戏模式 (432 行) |
| [Plugins/Carla/Source/Carla/Game/CarlaGameModeBase.h/.cpp](https://github.com/OpenHUTB/hutb/blob/hutb/Unreal/CarlaUE4/Plugins/Carla/Source/Carla/Game/CarlaGameModeBase.cpp) | Carla 基础游戏模式 GameMode |
| [Plugins/Carla/Source/Carla/Game/CarlaEpisode.h/.cpp](https://github.com/OpenHUTB/hutb/blob/hutb/Unreal/CarlaUE4/Plugins/Carla/Source/Carla/Game/CarlaEpisode.cpp) | Episode 管理器 |
| [Plugins/AirSim/Source/SimMode/SimModeBase.h/.cpp](https://github.com/OpenHUTB/air/blob/main/Unreal/Plugins/AirSim/Source/SimMode/SimModeBase.cpp) | AirSim 仿真模式 |
| [Plugins/AirSim/Source/AirSim.Build.cs](https://github.com/OpenHUTB/air/blob/main/Unreal/Plugins/AirSim/Source/AirSim.Build.cs) | 构建配置（Carla 依赖） |
| [Config/DefaultEngine.ini](https://github.com/OpenHUTB/air/blob/main/Unreal/Environments/Blocks/Config/DefaultEngine.ini) | 游戏模式注册 |
| [Config/DefaultGame.ini](https://github.com/OpenHUTB/air/blob/main/Unreal/Environments/Blocks/Config/DefaultGame.ini) | 打包配置 |

### Carla 传感器文件

| 文件（位于 [hutb/Unreal/CarlaUE4/Plugins/Carla/Source/Carla/](https://github.com/OpenHUTB/hutb/tree/hutb/Unreal/CarlaUE4/Plugins/Carla/Source/Carla/Sensor) 目录） | 传感器 |
|------|--------|
| [Sensor/SceneCaptureCamera.h/.cpp](https://github.com/OpenHUTB/hutb/blob/hutb/Unreal/CarlaUE4/Plugins/Carla/Source/Carla/Sensor/SceneCaptureCamera.cpp) | RGB 相机 |
| [Sensor/DepthCamera.h/.cpp](https://github.com/OpenHUTB/hutb/blob/hutb/Unreal/CarlaUE4/Plugins/Carla/Source/Carla/Sensor/DepthCamera.cpp) | Depth 相机 |
| [Sensor/SemanticSegmentationCamera.h/.cpp](https://github.com/OpenHUTB/hutb/blob/hutb/Unreal/CarlaUE4/Plugins/Carla/Source/Carla/Sensor/SemanticSegmentationCamera.cpp) | 语义分割 |
| [Sensor/InstanceSegmentationCamera.h/.cpp](https://github.com/OpenHUTB/hutb/blob/hutb/Unreal/CarlaUE4/Plugins/Carla/Source/Carla/Sensor/InstanceSegmentationCamera.cpp) | 实例分割 |
| [Sensor/RayCastLidar.h/.cpp](https://github.com/OpenHUTB/hutb/blob/hutb/Unreal/CarlaUE4/Plugins/Carla/Source/Carla/Sensor/RayCastLidar.cpp) | 激光雷达 |
| [Sensor/Radar.h/.cpp](https://github.com/OpenHUTB/hutb/blob/hutb/Unreal/CarlaUE4/Plugins/Carla/Source/Carla/Sensor/Radar.cpp) | 毫米波雷达 |
| [Sensor/InertialMeasurementUnit.h/.cpp](https://github.com/OpenHUTB/hutb/blob/hutb/Unreal/CarlaUE4/Plugins/Carla/Source/Carla/Sensor/InertialMeasurementUnit.cpp) | IMU |
| [Sensor/GnssSensor.h/.cpp](https://github.com/OpenHUTB/hutb/blob/hutb/Unreal/CarlaUE4/Plugins/Carla/Source/Carla/Sensor/GnssSensor.cpp) | GNSS |

### AirSim 核心文件

| 文件（位于 [air/Unreal/Plugins/AirSim/Source](https://github.com/OpenHUTB/air/tree/main/Unreal/Plugins/AirSim/Source) 目录） | 部件 |
|------|-----------|
| [Vehicles/Multirotor/FlyingPawn.h/.cpp](https://github.com/OpenHUTB/air/tree/main/Unreal/Plugins/AirSim/Source/Vehicles/Multirotor) | 无人机棋子 |
| [Vehicles/Multirotor/MultirotorPawnSimApi.h/.cpp](https://github.com/OpenHUTB/air/tree/main/Unreal/Plugins/AirSim/Source/Vehicles/Multirotor) | 无人机 API 封装器 |
| [Vehicles/Multirotor/SimModeWorldMultiRotor.h/.cpp](https://github.com/OpenHUTB/air/tree/main/Unreal/Plugins/AirSim/Source/Vehicles/Multirotor) | 多旋翼飞行器模拟模式 |
| [AirLib/include/physics/FastPhysicsEngine.hpp](https://github.com/OpenHUTB/air/blob/main/AirLib/include/physics/FastPhysicsEngine.hpp) | 无人机物理引擎 |
| [AirLib/include/vehicles/multirotor/MultiRotorPhysicsBody.hpp](https://github.com/OpenHUTB/air/blob/main/AirLib/include/vehicles/multirotor/MultiRotorPhysicsBody.hpp) | 刚体模型 |
| [AirLib/include/vehicles/multirotor/api/MultirotorRpcLibServer.hpp](https://github.com/OpenHUTB/air/blob/main/AirLib/include/vehicles/multirotor/api/MultirotorRpcLibServer.hpp) | API 服务端 |
| [AirLib/include/sensors/imu/ImuSimple.hpp](https://github.com/OpenHUTB/air/blob/main/AirLib/include/sensors/imu/ImuSimple.hpp) | 带噪声的 IMU |
| [AirLib/include/sensors/gps/GpsSimple.hpp](https://github.com/OpenHUTB/air/blob/main/AirLib/include/sensors/gps/GpsSimple.hpp) | 带噪声的 GPS |
| [AirLib/include/sensors/barometer/BarometerSimple.hpp](https://github.com/OpenHUTB/air/blob/main/AirLib/include/sensors/barometer/BarometerSimple.hpp) | 带噪声的气压计 |

## 附录 B：Python API 快速参考

### Carla API (端口 2000)
```python
import carla
client = carla.Client('localhost', 2000)
world = client.get_world()

# 环境
world.set_weather(carla.WeatherParameters.ClearNoon)
world.get_map().get_spawn_points()

# 参与者
bp = world.get_blueprint_library().find('vehicle.tesla.model3')
vehicle = world.spawn_actor(bp, spawn_point)
vehicle.set_autopilot(True)

# 传感器
cam_bp = world.get_blueprint_library().find('sensor.camera.rgb')
camera = world.spawn_actor(cam_bp, transform, attach_to=vehicle)
camera.listen(lambda image: process(image))
```

### AirSim API (端口 41451)
```python
import airsim
client = airsim.MultirotorClient(port=41451)
client.confirmConnection()
client.enableApiControl(True)
client.armDisarm(True)

# 飞行
client.takeoffAsync().join()
client.moveToPositionAsync(x, y, z, speed).join()
client.moveByVelocityAsync(vx, vy, vz, duration).join()

# 传感器
responses = client.simGetImages([
    airsim.ImageRequest("0", airsim.ImageType.Scene, False, False),
    airsim.ImageRequest("0", airsim.ImageType.DepthPerspective, True),
    airsim.ImageRequest("0", airsim.ImageType.Segmentation, False, False),
])
imu = client.getImuData()
gps = client.getGpsData()
```

### 组合使用
```python
import carla, airsim, threading

# 两个 API 都在同一个世界
carla_client = carla.Client('localhost', 2000)
air_client = airsim.MultirotorClient(port=41451)

# 在飞行模拟无人机时生成 Carla 交通
world = carla_client.get_world()
vehicle = world.spawn_actor(vehicle_bp, spawn_point)
vehicle.set_autopilot(True)

air_client.takeoffAsync().join()
air_client.moveToZAsync(-30, 5).join()  # 30 米高度

# 收集同步地面+空中数据
ground_image = carla_camera.listen(callback)
aerial_images = air_client.simGetImages([...])
```
