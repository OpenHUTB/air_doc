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



