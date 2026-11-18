# 空地一体协同智能载具虚拟仿真实验教程（Ubuntu / OpenHUTB）
> 本教程在 **Ubuntu** 上带你在 **OpenHUTB/hutb 官方包** 中跑通“地面车辆 + 无人机”的空地一体实验：先看到车机同框，再按六个模块完成场景构建、感知决策、无人机、空地协同、交通场景和 AI 辅助开发。
>
> **路径说明（务必先看）**：模拟器安装入口始终使用 OpenHUTB 主页的 `https://openhutb.github.io/dl.sh`，本文没有改成其他下载源。教程里有两套脚本根目录，不能混用：
> - `hutb/PythonAPI/...`：`dl.sh` 下载的模拟器包，空地一体脚本和 8 个分步教程在 `PythonAPI/examples/air/`。
> - `doc/src/...`：`OpenHUTB/doc` 文档仓库，感知、规划、交通类示例脚本在这里，**不在** `hutb/PythonAPI/examples/`。
>
> air_doc 上的 Carla-Air 预编译包使用 `CarlaAir.sh` 和 `examples/`，目录布局与本教程的 hutb 包不同；本教程仍按 OpenHUTB 主页的 hutb 包编写。
## 实验基本信息
| 项目 | 内容 |
| --- | --- |
| 实验名称 | 空地一体协同智能载具虚拟仿真实验 |
| 操作系统 | Ubuntu 18.04 / 20.04 / 22.04 / 24.04（64 位） |
| 面向专业 | 人工智能、机器人工程、低空经济 |
| 实验课时 | 课程 32 学时，实验 16 学时 |
| 实验模块 | 6 个模块、29 个交互步骤 |
| 知识点 | 8 个 |
| 仿真底座 | OpenHUTB/hutb（已集成 CARLA + AirSim 空地一体与无人机插件） |
| 模拟器获取 | `https://openhutb.github.io/dl.sh` |
| Python 接口 | 使用包内 `PythonAPI/carla/dist` 里的 wheel |
| 开源许可 | MIT |
## 学习目标
完成本实验后，你将能够：
- 理解 OpenHUTB/hutb 中 CARLA 与 AirSim 单进程空地一体仿真的基本架构，说清两者如何共享同一世界、同一套物理与传感器时间戳。
- 独立启动仿真环境，完成路网生成、场景保存、车辆与无人机连接。
- 配置相机、激光雷达、毫米波雷达等传感器，完成目标检测、点云可视化与多传感器数据采集。
- 编写 Python 脚本控制车辆与无人机，完成路径跟踪、航点飞行、多机编队与空地协同任务。
- 基于路口流量、车均延误、饱和度、排队长度、覆盖率、冲突次数等指标评价方案，并完成一次复测迭代。
- 按开源协作规范提交代码与文档，使用 GitHub Copilot 辅助读文档、改脚本、排查报错。
## 先修知识
- Python 基础：变量、循环、函数、常用库（numpy、matplotlib 均可）。
- Linux 基础：终端、路径、环境变量、`pip`/`venv`。
- ROS2 基础：节点、话题、服务的基本概念；没学过可先看平台前置微课。
- 控制与机器学习基础：PID、状态估计、目标检测的基本概念。
## 你将完成
- 启动 OpenHUTB/hutb，看到车辆与无人机在**同一个世界**中运行。
- 跑通 4 宫格传感器画面：RGB、深度图、语义分割、LiDAR 鸟瞰。
- 观察无人机跟踪地面车辆、天气切换、车机时间戳对齐。
- 按六个模块完成 29 个交互步骤，并提交实验报告与指标表。
## 开始前
### 硬件
- Ubuntu 18.04 / 20.04 / 22.04 / 24.04（64 位）。
- 推荐 NVIDIA RTX 2070 及以上、显存 8 GB 及以上。
- 推荐内存 32 GB 及以上、可用磁盘 100 GB 及以上；最低 16 GB 内存、50 GB 磁盘。
- 已安装 NVIDIA 驱动，终端执行 `nvidia-smi` 能看到显卡信息。
- 只有核显或 8 GB 内存的机器可以看文档、写脚本，但跑模拟器会非常卡。
### 软件
- Git、Python 3、pip、VS Code。
- **不需要单独下载额外发行包**：下面的 `dl.sh` 会下载 OpenHUTB/hutb 官方包，其中已集成空地一体与无人机插件。
- 浏览器（用于看文档和 GitHub）。
- 能访问 GitHub；如果无法访问，请使用可用的网络环境。
### 系统依赖
```bash
sudo apt update
sudo apt install -y git wget curl tar unzip build-essential python3-pip python3-venv vulkan-tools
```
### 环境自检
在开始前，先确认下面几条命令都能正常输出：

| 检查项 | 命令 | 正常表现 |
| --- | --- | --- |
| 显卡/驱动 | `nvidia-smi` | 显示 NVIDIA 显卡和驱动 |
| Vulkan | `vulkaninfo --summary \| grep -E 'apiVersion\|deviceName'` | `deviceName` 是 NVIDIA 显卡；出现 `llvmpipe` 表示在走软件渲染 |
| Python | `python3 --version` | 主次版本与 wheel 文件名里的 `cpXX` 一致 |
| pip | `python3 -m pip --version` | 能打印 pip 版本 |
| Git | `git --version` | 能打印 Git 版本 |
| Wheel | `ls ~/hutb/PythonAPI/carla/dist/hutb-*.whl` | 下载后至少有一个 `hutb-*.whl` |
| 端口 | `ss -ltnp \| grep -E ':2000\|:41451'` | 模拟器启动后 `2000` 在监听；模块三/四还要看到 `41451` |

- `nvidia-smi` 没有输出：先装 NVIDIA 驱动并重启。
- `pip3` 找不到：先装 `python3-pip`，或改用 `python3 -m pip`。
- `python3` 与 wheel 的 `cpXX` 不一致：换用匹配的 Python 版本。
> **第一次学习不要从数字孪生开始。** 数字孪生需要编译 HUTB，最重、最容易卡；先跑模块二—五，最后再做模块一和数字孪生。
### 脚本目录约定
下面命令里的脚本分布在不同的目录，**每条命令先 `cd` 到对应目录**，不要假设当前目录：
| 脚本 | 正确位置 |
| --- | --- |
| 空地一体展示脚本（quick_start_showcase、air_ground_sync、drone_car_chase、demo_drive_and_fly） | `hutb/PythonAPI/examples/air/` |
| 8 个分步教程（01_hello_world 等） | `hutb/PythonAPI/examples/air/` |
| hutb 自带示例（generate_traffic、manual_control 等） | `hutb/PythonAPI/examples/` |
| 相机、边界框、点云、路径点脚本（rgb_camera_example、sensor_bounding_box、3d_lidar、waypoint_follow） | `doc/src/examples/`（OpenHUTB/doc 仓库） |
| 检测、轨迹、选点、规划、导航、交通指标脚本（object_detection、Drive、locate_moving、main_v4_VO、automatic_control_revised、traffic_indicators、signal_control、auto_signal_control、spectator） | `doc/src/course/...`（OpenHUTB/doc 仓库） |
| AirSim 多机示例（multi_agent_drone、add_drone、hello_drone） | `air` 仓库的 `PythonClient/multirotor/` |
`hutb/PythonAPI/examples/` 里**没有** `rgb_camera_example.py`、`sensor_bounding_box.py`、`3d_lidar.py`、`waypoint_follow.py`、`traffic_indicators.py`、`spectator.py` 这些文件，不要在这里找。以你本地发行包/仓库的实际路径为准；找不到时先 `ls` 看目录。
## 快速开始：30 分钟看到“车机同框”
下面以 Ubuntu 上的 OpenHUTB/hutb 官方包为例。所有命令都在终端里执行；首次运行需要联网下载依赖。
### 1. 下载 OpenHUTB/hutb 官方包
```bash
curl -fsSL -o dl.sh https://openhutb.github.io/dl.sh && chmod +x dl.sh && ./dl.sh
```
> 如果这一步报 `BadZipFile`，先不要反复重试；这是下载到的包不完整/服务器没有预编译包，排查方法见文末“常见问题”。
脚本会下载模拟器包；解压后进入生成的目录（通常叫 `hutb`，以实际解压结果为准）：
```bash
cd ~/hutb
```
> 下面统一假设包目录是 `~/hutb`。如果你解压到其他位置，把命令里的 `~/hutb` 换成实际目录。
### 2. 启动模拟器

先确认 `CarlaUE4.sh` 的位置：`dl.sh` 下载包在根目录；自己编译的包在 `Build/UE4Carla/.../LinuxNoEditor/`。

```bash
find ~/hutb -name CarlaUE4.sh
```

进入 `CarlaUE4.sh` 所在目录后，用 Vulkan 启动：

```bash
cd ~/hutb
chmod +x CarlaUE4.sh
./CarlaUE4.sh -vulkan -windowed -ResX=1280 -ResY=720 -quality-level=Low
```

如果使用自己编译的包，进入 `Build/UE4Carla/.../LinuxNoEditor` 后执行同样命令。

第一次启动会加载地图，比较慢，属正常现象。本教程统一使用 Vulkan，不使用 `-opengl`。

如果启动报 `bind: Address already in use`，说明 2000 端口被占用，通常是上次崩溃残留的 `CarlaUE4` 进程。先查 PID：

```bash
ss -ltnp | grep ':2000'
```

假设查到 PID 是 162777，先确认是自己的进程，再结束它：

```bash
ps -o pid,user,etime,cmd -p 162777
kill -TERM 162777
sleep 3
ss -ltnp | grep ':2000' || echo "2000 已释放"
```

如果还占着，再用 `kill -9 162777`。命令里的 `162777` 要换成实际 PID，**不要带尖括号**；写成 `<PID>` 会被 bash 当成重定向，报“未预期的记号 newline”。如果占用者不是你的进程，改用 `-carla-rpc-port=2100` 启动，并在客户端加 `--port 2100`。
### 3. 安装 Python 接口

`dl.sh` 下载包里已经带了 Python 接口 wheel，先看一眼文件名：

```bash
cd ~/hutb
ls PythonAPI/carla/dist/hutb-*.whl
python3 --version
```

文件名里的 `cpXX` 要和 `python3 --version` 的主次版本一致。不要直接用 `hutb-*.whl` 通配符，否则 pip 可能挑到另一个 Python 版本的包并报 `is not a supported wheel on this platform`。

如果终端前缀有 `(base)`、`(ase)` 等 conda 环境，先执行 `conda deactivate`，再用独立虚拟环境安装：

```bash
python3 -m venv ~/venv-air
source ~/venv-air/bin/activate
```

按当前 Python 自动选 wheel：

```bash
PYTAG="cp$(python3 -c 'import sys; print(f"{sys.version_info.major}{sys.version_info.minor}")')"
python3 -m pip install ~/hutb/PythonAPI/carla/dist/hutb-*-${PYTAG}-*.whl
python3 -m pip install pygame-ce numpy "opencv-contrib-python<5"
```

如果 `ls` 里没有与当前 Python 匹配的 wheel，换一个匹配的 Python 版本，或让 `dl.sh` 重新下载完整包。

跑 doc 仓库示例或需要 `agents` 的脚本时，还要加：

```bash
export CARLA_ROOT=$HOME/hutb
export PYTHONPATH=$CARLA_ROOT/PythonAPI/carla:$HOME/doc/src:$PYTHONPATH
```

如果提示找不到文件，先确认 `PythonAPI/carla/dist/` 下确实有 `hutb-*.whl`。

装好后先做一次连接自检：

```bash
cd ~/hutb
python3 -c "import carla; c=carla.Client('localhost',2000); c.set_timeout(10); w=c.get_world(); print(w.get_map().name, len(w.get_map().get_spawn_points()))"
```

正常会打印当前地图名和出生点数量。连不上先检查模拟器是否还在运行、2000 端口是否监听。
### 4. 运行车机展示（另开一个终端）
```bash
cd ~/hutb
python3 PythonAPI/examples/air/quick_start_showcase.py
```
你应该看到：一辆车在城市中自动巡航，无人机从空中追踪；4 宫格同时显示 RGB、深度图、语义分割、LiDAR 鸟瞰；天气自动轮换。按 `N` 手动切天气，按 `Esc` 退出。如果 4 宫格全黑，先按模块四的“同步复位”命令恢复异步，再重新运行。
> 这个脚本本身**不支持 WASD 接管驾驶**（源码里只注册了 `N` 和 `ESC`）。要键盘驾驶，另开终端运行 `python3 PythonAPI/examples/air/drive_vehicle.py`，那个脚本才是 `WASD` 驾驶、`Space` 手刹、`R` 倒车。
### 5. 运行空地同步
```bash
python3 PythonAPI/examples/air/air_ground_sync.py
```
同一画面中，地面车辆与空中无人机处于同一世界、同一场天气；按 `N` 切换天气，按 `Esc` 退出。
### 成功标志
- 车与无人机出现在同一画面。
- 4 路传感器画面同时刷新。
- 切换天气时，地面和空中同时变化。
- 没有端口占用；如果出现 API 版本不匹配的警告，不影响本教程功能。
## 六个实验模块
| 模块 | 内容 | 关键脚本 / 操作 | 对应步骤 |
| --- | --- | --- | --- |
| 模块一 | 环境与数字孪生底座 | 启动模拟器、OSM 生成路网、保存地图 | 1—3 |
| 模块二 | 自动驾驶感知—决策—控制 | 相机、边界框、点云、目标检测、路径跟踪 | 4—12 |
| 模块三 | 无人机与低空资产 | 双客户端连接、起飞、多机、倾斜摄影 | 13—17 |
| 模块四 | 空地协同 | 车机同屏、联合展示、无人机跟踪车 | 18—19 |
| 模块五 | 交通场景与多载具协同 | 交通流、路口指标、信号控制、多载具协同 | 20—25 |
| 模块六 | 协作与 AI 辅助开发 | Fork、分支、Copilot、Pull Request | 26—29 |
## 模块一：环境与数字孪生底座（建议最后做）
**目标**：把真实路网变成仿真场景，并保存为可重复载入的地图。
1. 启动模拟器服务端并连接客户端：
   ```bash
   cd ~/hutb
   ./CarlaUE4.sh -vulkan -windowed -ResX=1280 -ResY=720 -quality-level=Low
   ```
   服务端启动后，用**另一个终端**执行 `config.py` 加载地图（在 hutb 包根目录执行），不要关掉模拟器窗口：
   ```bash
   python3 PythonAPI/util/config.py -m Town01
   ```
2. 由 OpenStreetMap 生成路网：打开数字孪生工具（CarlaTools 内容中的 `UW_OnRoadMainWidget`），填入 OSM 网址或导入本地 `.osm`，框选区域后点击生成。生成时间取决于机器性能和区域大小。
3. 加载生成场景并保存：
   ```bash
   # 在 hutb 包根目录
   python3 PythonAPI/util/config.py -x *.xodr
   ```
   如果按文档仓库的写法运行，则要先进入 `OpenHUTB/doc` 仓库根目录，再执行 `python3 src/util/config.py -x *.xodr`；`src/` 是 doc 仓库的目录，不在 hutb 包里。
   保存时间取决于机器性能和地图大小；保存后在编辑器中检查车道与路口。
**运行效果**：加载 `.xodr` 后，模拟器里能看到道路、路口和车辆出生点；保存地图后重新加载，路网仍然存在。
> 这一步依赖 HUTB 编译。第一次做建议在教师演示后完成，不要自行硬编。
## 模块二：自动驾驶感知—决策—控制

**目标**：让车辆“看得见、判得准、走得稳”。

**说明**：下面命令按 `dl.sh` 下载的官方包编写，已在 Vulkan 启动方式下实际跑通；地图使用 `Town10HD` / `Town01_Opt`，成功标志按实测结果写。

**运行效果**：跑通后你会看到 6 路相机画面、车辆上的三维边界框、Open3D 点云、YOLO 检测框，以及车辆沿路点或规划路径行驶。

如果还没有 doc 仓库，先克隆到 `~/doc`：

```bash
git clone https://github.com/OpenHUTB/doc.git ~/doc
```

再进入 doc 仓库并设置环境变量：

```bash
conda deactivate
source ~/venv-air/bin/activate
export CARLA_ROOT=$HOME/hutb
export PYTHONPATH=$CARLA_ROOT/PythonAPI/carla:$HOME/doc/src:$PYTHONPATH
cd ~/doc/src
```

### 4. 多路相机

脚本会自动生成 `hero`，不需要先运行 `manual_control.py`：

```bash
python3 examples/rgb_camera_example.py
```

成功标志：弹出 `All cameras` 窗口，RGB、语义分割、实例分割、深度、DVS、光流 6 路画面同时刷新，按 `q` 退出。请用 `q` 正常退出，直接 `Ctrl+C` 可能留下传感器，需要重启模拟器清理。

### 5. 三维边界框

需要场景里已有 `hero`。第 4 步加 `--keep_ego_vehicle` 可保留；否则另开终端运行：

```bash
cd ~/hutb
python3 PythonAPI/examples/manual_control.py
```

再在 doc 终端运行：

```bash
cd ~/doc/src
python3 examples/sensor_bounding_box.py --host localhost --port 2000 --rolename hero --view above --distance 100 --res 800x600 --fov 90
```

成功标志：俯视窗口里车辆被红橙色三维框包住。

### 6. 激光雷达点云

需要 Open3D；新版 matplotlib 删除了 `cm.get_cmap`，要装兼容旧接口的版本：

```bash
python3 -m pip install open3d "matplotlib<3.9"
python3 examples/3d_lidar.py --host localhost --port 2000 --rolename hero --keep_ego_vehicle
```

成功标志：`Carla Lidar` 窗口出现点云，能看到道路边界和车辆轮廓。如果 matplotlib 版本过新，会报 `AttributeError: module 'matplotlib.cm' has no attribute 'get_cmap'`。

### 7. YOLOv3 目标检测

OpenCV 必须是 4.x，并要改一行兼容代码：

```bash
cd ~/doc/src/course/object_detection
python3 -m pip install imutils "opencv-contrib-python<5"
curl -L -o weights/yolov3.weights https://pjreddie.com/media/files/yolov3.weights
sed -i 's/layers_names\[i\[0\]-1\]/layers_names[i-1]/' object_detection.py
python3 object_detection.py
```

成功标志：车辆视角窗口中，车辆、行人、自行车上出现检测框和类别标签。OpenCV 5 已移除 Darknet 后端，会报 `Unknown layer type in function 'readNet'`；`layers_names[i[0]-1]` 在 NumPy 2 下会报 `IndexError`，两个问题都要处理。画面里没有目标时不会出现框，可先运行 `python3 examples/generate_traffic.py -n 30 -w 10 --safe`。

### 8. 路径点跟踪

需要 Town01 路点。普通 `Town01` 在部分环境里会卡住，优先改用 `Town01_Opt`：

```bash
# 终端 A
cd ~/hutb
python3 PythonAPI/util/config.py -m Town01_Opt
# 终端 B
cd ~/doc/src
python3 examples/waypoint_follow.py --host localhost --port 2000 --json_file examples/data/town01_waypoints.json --sync --fixed_delta_seconds=0.05 --spawn --visualize --camera_follow
```

成功标志：车辆从第一个路点出发，窗口中出现路点编号，车辆沿路点行驶。脚本结束后把同步模式复位：

```bash
python3 - <<'PY'
import carla
c = carla.Client("localhost", 2000); c.set_timeout(20)
w = c.get_world(); s = w.get_settings()
s.synchronous_mode = False; s.fixed_delta_seconds = None
w.apply_settings(s)
print("async restored")
PY
```

### 9. 轨迹跟踪

```bash
cd ~/doc/src/course/trajectory_tracking
python3 Drive.py
```

成功标志：车辆按 `Waypoints.txt` 行驶，场景中出现红色路点编号，`Results/` 下生成 CSV。**不要先运行 `generate_traffic.py`**，否则 spawn point 216/11 被占用，`try_spawn_actor` 返回 `None` 后脚本会崩。该脚本需要至少 217 个 spawn point，`Town01_Opt` 有 255 个，可用。

### 10. 地图选点移动

```bash
cd ~/doc/src/course
python3 locate_moving.py
```

成功标志：俯视地图窗口显示道路和建筑，鼠标点击处生成一辆自动驾驶车辆。脚本把相机固定在 `(0,0,50)`，在 `Town01_Opt` 原点可能是空地/水面，窗口会全黑；实测在 `Town10HD` 下正常，建议这一步用 `Town10HD`。

### 11. 路线规划（湖工商场景）

```bash
cd ~/doc/src/course/Motion_Planning
python3 main_v4_VO.py
```

### 12. 校园轨迹（湖工商场景）

```bash
cd ~/doc/src/course/navigation
python3 automatic_control_revised.py --s 北门 --e 西门 --speed 5
```

11、12 的坐标写死在湖工商场景里，换到 Town01/Town10HD 没有意义；先按模块一加载校园地图。

**模块二排错**

- `ModuleNotFoundError: cv2`：确认在 `venv-air` 里，且装的是 OpenCV 4.x（不要用 5.x）。
- `AttributeError: module 'matplotlib.cm' has no attribute 'get_cmap'`：`python3 -m pip install "matplotlib<3.9"`。
- `Unknown layer type in function 'readNet'`：OpenCV 5 已移除 Darknet，降到 OpenCV 4.x。
- `IndexError` 出现在 `output_layers`：把 `layers_names[i[0]-1]` 改成 `layers_names[i-1]`。
- 脚本退出后传感器/车辆残留、或画面不动：重启模拟器，并把 `synchronous_mode` 复位为 `False`（见第 8 步代码）。
- 成功标志汇总：6 路相机同步、边界框贴合车辆、点云有道路/车辆轮廓、检测框随视角变化、车辆能沿路点或规划路径行驶。
## 模块三：无人机与低空资产

**目标**：连接 CARLA 与 AirSim 双客户端，完成起飞、多机添加和航拍建模。

**前置**：`~/Documents/AirSim/settings.json` 里设置 `"EnableRpc": true`，重启模拟器，确认两个端口都在监听。如果这个文件不存在，先启动一次模拟器让 AirSim 生成，再修改。

```bash
ss -ltnp | grep -E ':2000|:41451'
```

如果 `import airsim` 报 `ModuleNotFoundError: No module named 'backports'`：

```bash
python3 -m pip install backports.ssl_match_hostname
```

1. 双客户端连接：

```bash
cd ~/hutb
python3 PythonAPI/examples/air/01_hello_world.py
```

如果你在纯 SSH、没有图形桌面的环境里运行，先设置 `DISPLAY` 或使用虚拟显示；有桌面的机器一般不用设置。

实测成功输出：`CARLA connection successful: Carla/Maps/...`、`Number of available spawn points: ...`、`AirSim connection successful: Drone is ready to fly.`

2. 起飞、固定航点、正方形航线、降落：

```bash
python3 PythonAPI/examples/air/05_drone_takeoff.py
```

实测成功输出：`已起飞!` → `当前位置 NED: (80.1, -0.0, -30.0)` → `降落完成!`。

3. 天气与车流：

```bash
python3 PythonAPI/examples/air/02_weather_control.py
python3 PythonAPI/examples/air/03_spawn_traffic.py
```

实测输出：5 种天气依次切换并恢复晴天；生成 10 辆车、13 个行人，观察 20 秒后清理。

4. 多机：需要 `OpenHUTB/air` 仓库的 `PythonClient/multirotor/`，并在 `settings.json` 里增加 `Drone2`；第一次做先完成单机，再按 air 仓库示例增加第二架。

**运行效果**：终端先打印 CARLA 地图名和出生点数量，再打印 AirSim 已就绪；起飞脚本会依次打印起飞、NED 坐标和降落完成；天气脚本能看到 5 种天气切换；车流脚本会生成车辆和行人。

**注意**：连上无人机后不要用 `config.py` 切地图。部分构建在切换地图时可能因 41451 端口重绑崩溃；要换地图，先关模拟器、改 `settings.json`、再启动。

**成功标志**：无人机能起飞、悬停、按航点飞行、降落；多机能独立控制；重建模型能导入虚幻引擎。
## 模块四：空地协同

**目标**：让无人机和地面车辆在同一世界协同。

```bash
cd ~/hutb
python3 PythonAPI/examples/air/air_ground_sync.py
python3 PythonAPI/examples/air/quick_start_showcase.py
python3 PythonAPI/examples/air/drone_car_chase.py
python3 PythonAPI/examples/air/demo_drive_and_fly.py
```

一次只跑一个，脚本会自己生成车辆和相机。

**运行效果**：

- `air_ground_sync.py` 能进入 30 Hz 同步模式，显示无人机跟车画面。
- `quick_start_showcase.py` 显示 2×2 四宫格（RGB、深度、语义、LiDAR）。如果四宫格全黑，先检查上一轮是否把 `synchronous_mode` 留成了 `True`，执行下面的复位命令再跑：

```bash
python3 - <<'PY'
import carla
c = carla.Client("localhost", 2000); c.set_timeout(20)
w = c.get_world(); s = w.get_settings()
s.synchronous_mode = False; s.fixed_delta_seconds = None
w.apply_settings(s)
print("async restored")
PY
```

- `drone_car_chase.py`、`demo_drive_and_fly.py` 是交互式长循环，按 `Esc`/`q` 退出；建议先跑前两个，熟悉后再试后两个。
- 脚本不会打印固定的“坐标对齐误差 0.0000 m”，不要把这个数字写进报告；以画面同步和 `calibrate_offset` 的输出为准。

**成功标志**：车与无人机同屏；四路传感器画面刷新；切换天气时两侧同步变化。
## 模块五：交通场景与多载具协同

**目标**：在交通场景中加入多架无人机，评价车—机耦合效果。

实验按 **20 项算法测试矩阵**组织：感知 5、定位与地图 3、规划与决策 3、控制 1、交通 3、无人机与协同 3、学习 2。以下脚本在 `OpenHUTB/doc` 仓库的 `src/` 下执行：

```bash
cd ~/doc/src
# 生成车流与行人
python3 examples/generate_traffic.py -n 50 -w 20 --safe
# 统计路口指标（依赖湖工商场景）
python3 course/traffic_indicators.py
# 手动信号控制
python3 course/signal_control.py
# 另开终端请求
curl "http://127.0.0.1:5000/set_traffic_light?traffic_id=-5&color_id=3&color_time=30"
# 自动感应控制
python3 course/auto_signal_control.py
curl "http://127.0.0.1:5000/set_traffic_light"
# 四个预设路口视角轮换
python3 course/spectator/spectator.py
```

**运行效果**：

- `generate_traffic.py` 按 `-n`/`-w` 生成指定数量的车辆和行人；数量多时部分 spawn point 会因碰撞失败，日志中的 `Spawn failed because of collision at spawn position` 是正常现象。
- 模块三的 `03_spawn_traffic.py` 默认会生成 10 辆车 + 13 个行人。
- `traffic_indicators.py` 的两个路口坐标写死在湖工商场景（桐梓坡路—西二环、桐梓坡—望岳），在 Town10HD 上会输出 0，必须先加载湖工商地图。
- `signal_control.py` / `auto_signal_control.py` 的 Flask 接口可在任意地图启动，但信号灯 ID 和 spectator 视角依赖湖工商场景。
- `spectator.py` 只是 4 个固定视角轮换，不是多机巡检；多机覆盖率/冲突次数目前没有现成脚本，需要自己补统计。

再做两组对比：

- 改变车辆数（20、50、100 辆），记录排队长度、车均延误、饱和度。
- 改变无人机数量（1、2、3 架），记录覆盖率、任务完成时间、空地冲突次数。

**成功标志**：改配时后指标实时变化；车流增加时排队、延误、饱和度上升；能找到饱和度/协同负载的拐点。
## 模块六：协作与 AI 辅助开发
**目标**：按开源协作流程提交成果，并用大模型辅助开发。
1. Fork 并克隆仓库，新建分支（把 `你的用户名`、`你的仓库名` 换成实际值，例如 `hutb` 或 `doc`）：
   ```bash
   git clone https://github.com/你的用户名/你的仓库名.git
   cd 你的仓库名
   git checkout -b docs-fix-contribution-steps
   git status
   git diff
   ```
2. 按 GitHub Copilot 文档启用 Copilot，在 VS Code 中登录，让它解释一段 OpenHUTB 文档中的调用链。
3. 让模型补全或改写 `examples` 中的示例脚本（如增加传感器、改变车速），在模拟器中运行验证，逐行核对改动。
4. 提交：
   ```bash
   git add CONTRIBUTING.md
   git diff --cached
   git commit -m "docs: 说明本次修改"
   git push -u origin docs-fix-contribution-steps
   ```
   把 `CONTRIBUTING.md` 换成你实际修改的文件；贡献指南要求 `git add` 具体文件，而不是 `git add .`；提交前用 `git diff --cached` 再核对一次。创建目标为 OpenHUTB 的 Pull Request，按评审意见修改后合并，并提交实验报告与指标对比表。
## 实验报告要求
实验报告建议包含以下内容：
1. **实验环境**：操作系统、显卡、Vulkan、Python 与 wheel 的匹配情况、模拟器获取方式。
2. **过程记录**：每个模块的关键命令、运行结果（截图或终端输出）、遇到的问题与解决办法。
3. **指标表**：路口流量、车均延误、饱和度、排队长度；多机实验的覆盖率、任务完成时间、冲突次数。
4. **结果分析**：不同参数下指标如何变化，是否与预期一致；失败情形的原因与改进。
5. **成果链接**：代码分支、Pull Request、实验数据或录屏（如有）。
报告以“能复现、能解释、有证据”为标准，不接受只有截图、没有指标与分析的报告。
## 结果与评价
完成实验后，你应该能复现以下结果：
- 数字孪生：路网可保存、可重复载入；生成和保存时间取决于机器性能与区域大小。
- 感知：雨雾/夜间下检测框数量和置信度下降；点云远处变稀疏。
- 控制：步长越小、采样越密，轨迹误差越小。
- 无人机：多机分层飞行时，覆盖率、冲突次数可比较；FPS 以本机实测为准。
- 空地协同：车机同屏、传感器画面同步；当前版本未打印固定坐标误差数值。
- 交通场景：流量上升时排队、延误、饱和度上升，可找临界流量。
成绩构成：
| 维度 | 权重 | 看什么 |
| --- | --- | --- |
| 参与度 | 40% | 前测/出勤、角色承担、Commits、Issues 讨论、PR 数量与审核评论 |
| 提交规范性 | 20% | 分支命名、提交信息、PR 描述、代码风格、文档格式 |
| 作业质量 | 40% | 步骤完成度、指标表与复算、实验报告、创新与复测 |
等级：不及格、及格、中、良、优。
## 常见问题
**Q：只有核显/8 GB 内存能跑吗？**
可以看文档、写脚本，跑模拟器会非常卡。建议用带 NVIDIA 独显、32 GB 内存以上的机器做模块三—五。
**Q：运行 `./dl.sh` 报 `BadZipFile`？**
`dl.sh` 只是下载器，真正的模拟器包由它从服务器拉取。报 `BadZipFile` 通常不是 Ubuntu 的问题，而是下载到的文件不是完整 zip：服务器返回了 404/HTML 页面、下载中断，或网络加速器改写了内容。
按下面顺序排查：
1. 重新执行一次 `./dl.sh`，不要连续重试十几次。
2. 找到刚下载的 zip，检查它是不是真的 zip：
   ```bash
   find ~ -name '*.zip' -mmin -30 -ls
   file 上一步输出的文件
   ```
   如果 `file` 显示 HTML、ASCII text，或者文件很小，说明下载失败，重新下载。
3. 解压后确认 `CarlaUE4.sh` 存在：
   ```bash
   find ~/hutb -name CarlaUE4.sh
   ```
4. 仍然失败时，换网络环境，或从 OpenHUTB 主页手动下载完整包。

**Q：启动报 `bind: Address already in use`？**
说明 2000 端口被占用，通常是上次崩溃残留的 `CarlaUE4` 进程。先查 PID，再结束自己的旧进程：
```bash
ss -ltnp | grep ':2000'
kill -TERM 162777
```
把 `162777` 换成 `ss` 查到的实际 PID，不要写尖括号。还占着就 `kill -9 162777`。

**Q：安装 wheel 报 `is not a supported wheel on this platform`？**
wheel 文件名里的 `cpXX` 和当前 `python3 --version` 不一致。用第 3 节的 `PYTAG` 命令按当前 Python 自动选，或换一个匹配的 Python。

**Q：启动后黑屏、窗口一闪而过或报 `VK_ERROR_DEVICE_LOST`？**
先确认 `vulkaninfo --summary` 里的 `deviceName` 是 NVIDIA 显卡，不是 `llvmpipe`；再降低画质：
```bash
./CarlaUE4.sh -vulkan -windowed -ResX=1280 -ResY=720 -quality-level=Low
```
仍然失败时，换一台有 NVIDIA 独显的机器；不要改用 `-opengl`。

**Q：`import airsim` 报 `No module named 'backports'`？**
```bash
python3 -m pip install backports.ssl_match_hostname
```

**Q：`quick_start_showcase.py` 四宫格全黑？**
上一个脚本可能把 `synchronous_mode` 留成了 `True`。先复位为异步：
```bash
python3 - <<'PY'
import carla
c = carla.Client("localhost", 2000); c.set_timeout(20)
w = c.get_world(); s = w.get_settings()
s.synchronous_mode = False; s.fixed_delta_seconds = None
w.apply_settings(s)
print("async restored")
PY
```
再重新运行脚本。

**Q：`3d_lidar.py` 报 `cm.get_cmap`？**
```bash
python3 -m pip install "matplotlib<3.9"
```

**Q：YOLO 报 `Unknown layer type` 或 `IndexError`？**
OpenCV 5 已移除 Darknet，降到 OpenCV 4.x：
```bash
python3 -m pip install "opencv-contrib-python<5"
```
再把 `layers_names[i[0]-1]` 改成 `layers_names[i-1]`。

**Q：普通 `Town01` 一直卡住？**
改用 `Town01_Opt`：
```bash
python3 PythonAPI/util/config.py -m Town01_Opt
```

**Q：`locate_moving.py` 窗口全黑？**
脚本把俯视相机固定在原点，部分地图原点没有道路。换 `Town10HD` 再运行。

**Q：`./CarlaUE4.sh: No such file or directory`？**
说明当前目录不对。`dl.sh` 包在根目录，自己编译的包在 `Build/UE4Carla/.../LinuxNoEditor`：
```bash
find ~/hutb -name CarlaUE4.sh
```
进入该文件所在目录再运行。

**Q：`Permission denied`？**
```bash
chmod +x CarlaUE4.sh
```

**Q：`ModuleNotFoundError: No module named 'carla'`？**
确认已激活 `venv-air`，并且装的是与当前 Python 匹配的 wheel：
```bash
source ~/venv-air/bin/activate
python3 -c "import carla; print(carla.__file__)"
```

**Q：`ModuleNotFoundError: No module named 'agents'`？**
没有设置 `PYTHONPATH`：
```bash
export PYTHONPATH=$HOME/hutb/PythonAPI/carla:$HOME/doc/src:$PYTHONPATH
```

## 参考
- OpenHUTB 项目主页：https://openhutb.github.io/
- OpenHUTB/air_doc 教程：https://openhutb.github.io/air_doc/
- OpenHUTB/hutb 仓库：https://github.com/OpenHUTB/hutb
- OpenHUTB/doc 示例与课程仓库：https://github.com/OpenHUTB/doc
- OpenHUTB/air 仓库：https://github.com/OpenHUTB/air
- 模拟器下载脚本：https://openhutb.github.io/dl.sh
- CARLA 官方文档：https://carla.readthedocs.io/
- CARLA Python API：https://carla.readthedocs.io/en/latest/python_api/
- AirSim 官方文档：https://microsoft.github.io/AirSim/
- AirSim Python API：https://microsoft.github.io/AirSim/apis/
- Open3D 文档：https://www.open3d.org/docs/
- OpenStreetMap：https://www.openstreetmap.org/
- OpenDRIVE 规范：https://www.asam.net/standards/detail/opendrive/
- YOLO/Darknet：https://pjreddie.com/darknet/yolo/
