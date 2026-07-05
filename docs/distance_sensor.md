## 距离传感器

默认情况下，距离传感器指向载具前方。通过修改设置，可以将其指向任何方向。

可配置参数：

参数           | 描述
--------------------|------------
X Y Z               | 传感器相对于载具的位置（NED，单位为米）（默认值：(0,0,0)-多旋翼飞行器，(0,0,-1)-汽车）
Yaw Pitch Roll      | 传感器相对于载具的朝向（度）（默认值 (0,0,0)）
MinDistance         | 距离传感器测量的最小距离（米，仅用于填充 PX4 的 Mavlink 消息）（默认值 0.2 米）
MaxDistance         | 距离传感器测量的最大距离（米）（默认值：40.0米）
ExternalController  | 是否将数据发送到外部控制器，例如 ArduPilot 或 PX4（如果使用）（默认为 `true`）

例如，要使传感器指向地面（用于类似气压计的高度测量），可以按如下方式修改其方向 - 


```json
"Distance": {
    "SensorType": 5,
    "Enabled" : true,
    "Yaw": 0, "Pitch": -90, "Roll": 0
}
```

!!! 注意
    对于汽车，传感器默认放置在载具中心上方 1 米处。这是必需的，因为如果传感器位于车内，则可能会给出异常数据。但这不会影响传感器的测量值，例如测量两辆车之间的距离。有关使用示例，请参阅 [获取多个车辆之间的距离 PythonClient/car/distance_sensor_multi.py](https://github.com/OpenHUTB/air/blob/main/PythonClient/car/distance_sensor_multi.py)。
