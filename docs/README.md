## 低空模拟器

<!-- Badges (flat): https://github.com/pudding0503/github-badge-collection -->
<p align=center>
    <a href="https://github.com/OpenHUTB/air/actions">
        <img src="https://raw.githubusercontent.com/OpenHUTB/doc/refs/heads/master/docs/img/badge.svg" alt="Continuous Integration Badge">
    </a>
    <a href="https://github.com/OpenHUTB/air/releases">
        <img src="https://img.shields.io/github/v/release/OpenHUTB/air" alt="Releases Badge">
    </a>
    <a href="https://github.com/OpenHUTB/air/blob/master/LICENSE.txt">
        <img src="https://img.shields.io/badge/License-MIT-yellow.svg" alt="License Badge">
    </a>
    <a href="https://github.com/OpenHUTB/air">
        <img src="https://img.shields.io/badge/platform-windows%20%7C%20macos%20%7C%20linux-lightgrey" alt="Supported Platforms Badge">
    </a>
</p>

该项目基于 [AirSim](https://github.com/microsoft/AirSim) 项目，可单独使用，也可以将其作为一个插件集成到  HUTB 项目中，配置方法请参考 [链接](dev/add_plugin.md) 。

2017年，微软研究院创建了 AirSim，作为人工智能研究和实验的模拟平台。2022 年微软调整战略，放弃了该项目的维护，我们在 AirSim 的基础之上继续开发，经过多年发展，这个研究项目已实现其目标，并取得了长足进展——成为共享研究代码和测试空中人工智能开发与模拟新想法的常用方式。此外，时间也推动了我们将技术应用于现实世界的方式不断进步，尤其是在空中机动和自主系统方面。例如，无人机送货不再是科幻故事，而是商业现实，这意味着新的需求需要满足。我们在此过程中收获颇丰，并感谢社区成员一直以来的积极参与。


# 欢迎来到低空模拟器

低空模拟器 Air 是一款基于虚幻引擎的无人机、汽车等模拟器。它是开源的跨平台仿真器，支持使用 PX4 和 ArduPilot 等主流飞行控制器进行软件在环仿真，以及使用 PX4 进行硬件在环仿真，以实现物理和视觉上逼真的模拟。它以虚幻插件的形式开发，可以轻松集成到任何虚幻环境中。

我们的目标是将 Air 开发为一个人工智能研究平台，用于试验自动驾驶汽车的深度学习、计算机视觉和强化学习算法。为此，Air 还开放了 API，以便以独立于平台的方式检索数据和控制车辆。

**观看 1.5 分钟的快速演示**

Air 中的无人机

[![AirSim Drone Demo Video](images/demo_video.png)](https://youtu.be/-WfTr1-OBGQ)

Air 中的汽车

[![AirSim Car Demo Video](images/car_demo_video.png)](https://youtu.be/gnz1X3UNM5Y)


## 如何获取

### Windows
[![Build Status](https://github.com/OpenHUTB/air/actions/workflows/test_windows.yml/badge.svg)](https://github.com/OpenHUTB/air/actions/workflows/test_windows.yml)

* [下载二进制文件](https://github.com/OpenHUTB/air/releases)

* [构建它](build_windows.md)

### Linux
[![Build Status](https://github.com/OpenHUTB/air/actions/workflows/test_ubuntu.yml/badge.svg)](https://github.comOpenHUTB/air/actions/workflows/test_ubuntu.yml)

* [下载二进制文件](https://github.com/OpenHUTB/air/releases)

* [构建它](build_linux.md)

### macOS
[![构建状态](https://github.com/OpenHUTB/air/actions/workflows/test_macos.yml/badge.svg)](https://github.com/OpenHUTB/air/actions/workflows/test_macos.yml)

* [构建它](build_macos.md)

有关更多详细信息，请参阅 [使用预编译二进制文件](docs/use_precompiled.md) 文档。


## 如何使用

### 快捷键效果 <span id="shortcut"></span>

按 F10 查看各种可用的天气效果选项。您还可以使用 [APIs](apis#weather-apis) 控制天气。按 F1 查看其他可用选项。

![record screenshot](images/weather_menu.png)

| 键                                                                                                                                                                      | 动作                               | 
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------| 
| <font color="#f8805a">`F1`</font>                                                                                                                                       | 弹出帮助信息                               | 
| <font color="#f8805a">`F10`</font>                                                                                                                                       | 参数设置                               | 
| <font color="#f8805a">`F`</font>                                                                                                                                       | 切换到第一人称视角（FPV）                               | 
| <font color="#f8805a">`B`</font>                                                                                                                                       | 切换到"跟随"视角                               | 
| <font color="#f8805a">`\`</font>                                                                                                                                       | 切换到地面观察者视角（俯视）                               | 
| <font color="#f8805a">`/`</font>                                                                                                                                       | 切换到“弹簧臂跟随”模式（相对于“跟随”视角更平）                               | 
| <font color="#f8805a">`F3`</font>                                                                                                                                       | 按下F1时切换线框模式                               | 
| <font color="#f8805a">`M`</font>                                                                                                                                       | 切换到手动相机控制                               | 
| <font color="#f8805a">`Page Up/Down`</font>                                                                                                                                       | 向上/向下移动                               | 
| <font color="#f8805a">`W, S`</font>                                                                                                                                       | 俯仰控制                               | 
| <font color="#f8805a">`A, D`</font>                                                                                                                                       | 偏航控制                               | 
| <font color="#f8805a">`R`</font>                                                                                                                                       | 切换记录                               | 
| <font color="#f8805a">`;`</font>                                                                                                                                       | 切换调试报告（物理、动力学、旋转等实时数据）                               | 
| <font color="#f8805a">`0`</font>                                                                                                                                       | 切换所有子窗口                               | 
| <font color="#f8805a">`1`</font>                                                                                                                                       | 切换深度子窗口                               | 
| <font color="#f8805a">`2`</font>                                                                                                                                       | 切换分割子窗口                               | 
| <font color="#f8805a">`3`</font>                                                                                                                                       | 切换场景子窗口                               | 
| <font color="#f8805a">`T`</font>                                                                                                                                       | 切换轨迹线                               | 
| <font color="#f8805a">`Backspace`</font>                                                                                                                                       | 重置一切                               | 
| <font color="#f8805a">`方向键 [汽车模式]`</font>                                                                                                                                       | 驾驶汽车                               | 

### 手动控制

如果您拥有如下所示的遥控器 (Remote Control, RC)，则可以在模拟器中手动控制无人机。对于汽车，您可以使用箭头键手动驾驶。

[更多详细信息](remote_control.md)

![record screenshot](images/AirSimDroneManual.gif)

![record screenshot](images/AirSimCarManual.gif)


### 程序控制

Air 提供 API，方便您在模拟中以编程方式与飞行器进行交互。您可以使用这些 API 检索图像、获取状态、控制飞行器等等。这些 API 通过 RPC 公开，并可通过多种语言访问，包括 C++、Python、C# 和 Java。

这些 API 也可作为独立的跨平台库的一部分使用，因此您可以将它们部署到车辆的配套计算机上。这样，您就可以在模拟器中编写和测试代码，然后在真实车辆上执行。迁移学习及相关研究是我们的重点领域之一。

请注意，您可以使用 [SimMode 设置](settings/#simmode) 来指定默认车辆或新的 ComputerVision 模式，这样您就不会在每次启动 AirSim 时收到提示。

* [示例](./example/list.md)

* [更多详细 API 信息](apis.md)

### 收集训练数据

您可以通过两种方式从 AirSim 生成用于深度学习的训练数据。最简单的方法是直接按下右下角的“录制”按钮。这将开始记录每一帧的姿势和图像。数据记录代码非常简单，您可以根据自己的喜好进行修改。

![record screenshot](images/record_data.png)

按照您所需的方式生成训练数据的更好方法是访问 API。这使您可以完全控制记录数据的方式、内容、位置和时间。

### 计算机视觉模式

AirSim 的另一种使用方式是所谓的“计算机视觉”模式。在此模式下，您无需使用车辆或物理引擎。您可以使用键盘在场景中移动，或使用 API 将可用的摄像头定位到任意姿势，并收集诸如深度、视差、表面法线或物体分割等图像。

[更多详细信息](image_apis.md)



## 教程

- [视频 - 使用 Pixhawk 教程设置 AirSim](https://youtu.be/1oY8Qu5maQQ) 
- [视频 - 使用 AirSim 和 Pixhawk 教程](https://youtu.be/HNWdYrtw3f0)
- [视频 - 使用 AirSim 的现成环境](https://www.youtube.com/watch?v=y09VbdQWvQY) 
- [网络研讨会——利用高保真仿真实现自主系统](https://note.microsoft.com/MSR-Webinar-AirSim-Registration-On-Demand.html) 
- [使用 AirSim 进行强化学习](reinforcement_learning.md) 
- [自动驾驶烹饪手册](https://aka.ms/AutonomousDrivingCookbook) 
- [使用 TensorFlow 实现简单的防碰撞](https://github.com/simondlevy/AirSimTensorFlow) 

## 参加

### 论文

更多技术细节请参阅 [AirSim 论文（FSR 2017 会议）)](https://arxiv.org/abs/1705.05065) 。请引用：

```shell
@inproceedings{airsim2017fsr,
  author = {Shital Shah and Debadeepta Dey and Chris Lovett and Ashish Kapoor},
  title = {AirSim: High-Fidelity Visual and Physical Simulation for Autonomous Vehicles},
  year = {2017},
  booktitle = {Field and Service Robotics},
  eprint = {arXiv:1705.05065},
  url = {https://arxiv.org/abs/1705.05065}
}
```

### 贡献

如果您正在寻找可以做出贡献的领域，请查看 [未解决的问题](https://github.com/OpenHUTB/air/issues) 。


* [更多关于 AirSim 设计的信息](design.md)
* [有关代码结构的更多信息](code_structure.md)
* [贡献指南](CONTRIBUTING.md)

### 谁在使用 AirSim？

我们维护着一份我们所知的一些项目、人员和团体的 [列表](who_is_using.md) 。如果您希望被收录在此列表中，请 [在此处提交申请](https://github.com/OpenHUTB/air/issues) 。


## 联系

加入我们的 [GitHub 讨论组](https://github.com/OpenHUTB/air/discussions) 以了解最新动态或提出任何问题。



## 最新动态

* [融合 Carla 与 AirSim](./dev/Carla_Air.md)
* [模块分析](./module/modules_list.md)
* [使用AirSim的项目](./third/used_by.md)
* [中文文档](https://msdocs.cn/AirSim/)

* [电影摄影机](https://github.com/microsoft/AirSim/pull/3949)
* [ROS2 包装器](https://github.com/microsoft/AirSim/pull/3976)
* [列出所有资产的 API](https://github.com/microsoft/AirSim/pull/3940)
* [movetoGPS API](https://github.com/microsoft/AirSim/pull/3746)
* [光流相机](https://github.com/microsoft/AirSim/pull/3938)
* [simSetKinematics API](https://github.com/microsoft/AirSim/pull/4066)
* [从现有的 UE 材质或纹理 PNG 动态设置对象纹理](https://github.com/microsoft/AirSim/pull/3992)
* [能够生成/销毁灯光并控制灯光参数](https://github.com/microsoft/AirSim/pull/3991)
* [Unity 支持多架无人机](https://github.com/microsoft/AirSim/pull/3128)
* [通过键盘控制手动相机速度](https://github.com/microsoft/AirSim/pulls?page=6&q=is%3Apr+is%3Aclosed+sort%3Aupdated-desc#:~:text=1-,Control%20manual%20camera%20speed%20through%20the%20keyboard,-%233221%20by%20saihv) 

如需查看完整的变更列表，请查看我们的 [变更日志](docs/CHANGELOG.md)

## 常问问题

如果您遇到问题，请查看 [常见问题解答](faq.md) 并随时在 [AirSim](https://github.com/OpenHUTB/air/issues) 存储库中发布问题。

## 行为守则

本项目已采用 [微软开源行为准则](https://opensource.microsoft.com/codeofconduct/) 。更多信息请参阅行为准则常见问题解答，或联系 [open@hutb.edu.cn](mailto:open@hutb.edu.cn) 提出其他问题或意见。



## 许可证

本项目遵循 MIT 许可证发布。请查看 [许可证文件](LICENSE) 了解更多详情。
