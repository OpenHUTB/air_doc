# 载具模拟器

该项目主要介绍 [四旋翼无人机](https://openhutb.github.io/air/wiki/moveOnPath-demo/) 、[六旋翼无人机](https://openhutb.github.io/air/wiki/hexacopter/) 、[直升机](https://openhutb.github.io/air_doc/ardupilot/sitl_with_airsim/#copter)、[飞行汽车](https://github.com/OpenHUTB/hutb/issues/3196)、[水下机器人](https://openhutb.github.io/mujoco_plugin/underwater/) 等载具，为 [人车模拟器文档](https://openhutb.github.io)  的一部分。


## 部署

依赖安装：
```shell
pip install -r requirements.txt
# 启动服务
mkdocs serve --livereload
```

详细部署方案请参考 [人车模拟器文档](https://github.com/OpenHUTB/doc) 。


## 文档贡献方向与计划

本仓库介绍低空载具仿真，是人车模拟器文档的一部分。欢迎围绕可复现的使用问题持续补充说明，适合优先处理的方向包括：

- **构建与环境：**按操作系统、模拟器分支和 Python 版本记录前置条件及构建命令，避免把不同版本的步骤混写。
- **API 与传感器示例：**补充可运行的最小示例，并注明所需的模拟器模式、载具和预期输出。
- **集成工作流：**完善 AirSim 与 ROS、PX4/ArduPilot、HUTB 的连接说明，保持本仓库与主文档之间的入口一致。
- **导航与链接：**将新增页面加入 mkdocs.yml 的对应目录，并确认相关页面之间可以互相到达。

提交前可在仓库根目录安装 requirements.txt 中的依赖并运行 `mkdocs serve --livereload`，检查页面能否生成、链接是否有效。涉及构建或接口行为时，请在 issue 中记录分支、操作系统和复现步骤，并在 PR 中说明实际核对的平台或运行方式；未经实测的示例应明确标注。

详见 [贡献指南](CONTRIBUTING.md)。

### 其他问题和解决方案

##### Codespace运行报错：ERROR: No matching distribution found for pymdownx

> 解决：
> 
> `pip install pymdown-extensions --force`


##### 编译 api_docs 

运行 `docs/api_docs/make.bat`需要安装：
```shell
# 
pip install sphinx_rtd_theme
# ModuleNotFoundError: No module named 'airsim'
pip install hutb
```

报错：sphinx.errors.ConfigError: Invalid `intersphinx_mapping` configuration (1 error)

解决：注释掉 conf.py 中的 intersphinx_mapping：
```python
# intersphinx_mapping = {'https://docs.python.org/': None}
```



