# 下载二进制文件

您只需下载预编译的二进制文件并运行即可立即开始使用。如果您想设置自己的虚幻环境，请参阅 [这些说明](https://github.com/Microsoft/AirSim/#how-to-get-it) 。

### 虚幻引擎

**Windows, Linux**: 从 [最新版本](https://github.com/Microsoft/AirSim/releases) 中下载适合您所选择的环境的二进制文件。

一些预编译的二进制环境文件可能包含多个文件（例如 `City.zip.001` 和 `City.zip.002`）。请确保在启动环境之前下载这 2 个文件。使用 [7zip 软件](https://www.7-zip.org/download.html) （包含 7z 命令）解压这些文件。在 Linux 上，将第一个文件名`City.zip.001`作为参数传递给 7z 命令，它会检测所有其他对应的压缩文件，解压命令为：`7z x City.zip.001`。


**macOS**:  你需要 [自己构建它](build_linux.md) 

### Unity (实验)

[Unity Asset Store](https://assetstore.unity.com/) 现已提供一个名为 Windridge City 的免费环境，作为 AirSim 在 Unity 上的实验版本。

**注意**：这是一个旧版本，许多功能和 API 可能无法使用。


## 控制车辆

我们的大多数用户通常使用 [APIs](apis.md) 来控制车辆。当然，您也可以手动控制车辆。您可以使用键盘、游戏手柄或 [方向盘](steering_wheel_installation.md) 来驾驶车辆。要手动操控无人机，您需要 XBox 控制器或遥控器（欢迎 [贡献](CONTRIBUTING.md) 键盘支持）。更多详情，请参阅遥控器设置。或者，您可以使用 [APIs](apis.md) 进行编程控制，或使用所谓的 [计算机视觉模式](image_apis.md) ，通过键盘在环境中移动。

## 没有好的 GPU？

AirSim 二进制文件（例如 CityEnviron）需要强大的 GPU 才能流畅运行。您可以在 Windows 上编辑 `run.bat` 文件（如果该文件不存在，请创建并添加以下内容）以低分辨率模式运行它们，如下所示：

```batch
start CityEnviron -ResX=640 -ResY=480 -windowed
```

对于 Linux 二进制文件，请使用`Blocks.sh`或相应的 shell 脚本，如下所示：

```shell
./Blocks.sh -ResX=640 -ResY=480 -windowed
```

查看所有其他 [命令行选项](https://docs.unrealengine.com/en-US/ProductionPipelines/CommandLineArguments/index.html) 

UE 4.24 默认使用 Vulkan 驱动程序，但它们会消耗更多 GPU 内存。如果您遇到内存分配错误，可以尝试使用 `-opengl` 切换到 OpenGL。


您还可以使用 `simRunConsoleCommand()` API 限制最大 FPS，如下所示：

```python
>>> import airsim
>>> client = airsim.VehicleClient()
>>> client.confirmConnection()
Connected!
Client Ver:1 (Min Req: 1), Server Ver:1 (Min Req: 1)

>>> client.simRunConsoleCommand("t.MaxFPS 10")
True
```
