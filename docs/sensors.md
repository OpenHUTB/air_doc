# Air 中的传感器

Air 目前支持以下传感器。
每个传感器都与一个指定其传感器类型的整数枚举相关联。

* 相机 Camera
* 气压计 Barometer = 1
* Imu = 2
* Gps = 3
* 磁力计 Magnetometer = 4
* 距离传感器 Distance Sensor = 5
* 激光雷达 Lidar = 6

!!! 笔记
    摄像头的配置与其他传感器不同，并且没有与之关联的枚举。请参阅 [常规设置](settings.md) 和 [图像 API](image_apis.md) ，了解摄像头的配置和 API。 

## 默认传感器

如果在`settings.json`中未指定任何传感器，则根据模拟模式默认启用以下传感器。


### 多旋翼
* Imu
* Magnetometer
* Gps
* Barometer

### 车
* Gps

### ComputerVision
* None

在后台，[AirSimSettings.hpp](https://github.com/Microsoft/AirSim/blob/main/AirLib/include/common/AirSimSettings.hpp) 中的 `createDefaultSensorSettings` 方法根据 `settings.json` 文件中指定的模拟模式，使用默认参数设置上述传感器。


## 配置默认传感器列表

可以在设置 json 中配置默认传感器列表：

```json
"DefaultSensors": {
    "Barometer": {
        "SensorType": 1,
        "Enabled" : true,
        "PressureFactorSigma": 0.001825,
        "PressureFactorTau": 3600,
        "UncorrelatedNoiseSigma": 2.7,
        "UpdateLatency": 0,
        "UpdateFrequency": 50,
        "StartupDelay": 0

    },
    "Imu": {
        "SensorType": 2,
        "Enabled" : true,
        "AngularRandomWalk": 0.3,
        "GyroBiasStabilityTau": 500,
        "GyroBiasStability": 4.6,
        "VelocityRandomWalk": 0.24,
        "AccelBiasStabilityTau": 800,
        "AccelBiasStability": 36
    },
    "Gps": {
        "SensorType": 3,
        "Enabled" : true,
        "EphTimeConstant": 0.9,
        "EpvTimeConstant": 0.9,
        "EphInitial": 25,
        "EpvInitial": 25,
        "EphFinal": 0.1,
        "EpvFinal": 0.1,
        "EphMin3d": 3,
        "EphMin2d": 4,
        "UpdateLatency": 0.2,
        "UpdateFrequency": 50,
        "StartupDelay": 1
    },
    "Magnetometer": {
        "SensorType": 4,
        "Enabled" : true,
        "NoiseSigma": 0.005,
        "ScaleFactor": 1,
        "NoiseBias": 0,
        "UpdateLatency": 0,
        "UpdateFrequency": 50,
        "StartupDelay": 0
    },
    "Distance": {
        "SensorType": 5,
        "Enabled" : true,
        "MinDistance": 0.2,
        "MaxDistance": 40,
        "X": 0, "Y": 0, "Z": -1,
        "Yaw": 0, "Pitch": 0, "Roll": 0,
        "DrawDebugPoints": false
    },
    "Lidar2": {
        "SensorType": 6,
        "Enabled" : true,
        "NumberOfChannels": 16,
        "RotationsPerSecond": 10,
        "PointsPerSecond": 100000,
        "X": 0, "Y": 0, "Z": -1,
        "Roll": 0, "Pitch": 0, "Yaw" : 0,
        "VerticalFOVUpper": -15,
        "VerticalFOVLower": -25,
        "HorizontalFOVStart": -20,
        "HorizontalFOVEnd": 20,
        "DrawDebugPoints": true,
        "DataFrame": "SensorLocalFrame"
    }
},
```

## 配置特定车辆传感器列表

车辆可以覆盖上面列出的部分默认传感器。激光雷达和距离传感器
默认情况下不会添加到车辆中，因此您需要通过这种方式添加它们。每个传感器必须具有有效的
“SensorType”，并且可以定义部分属性来覆盖上面显示的默认值，
并且您可以将 Enabled 设置为 false 以禁用特定类型的传感器。


```json
"Vehicles": {

    "Drone1": {
        "VehicleType": "SimpleFlight",
        "AutoCreate": true,
        ...
        "Sensors": {
            "Barometer":{
                "SensorType": 1,
                "Enabled": true,
                "PressureFactorSigma": 0.0001825
            },
            "MyLidar1": {
                "SensorType": 6,
                "Enabled" : true,
                "NumberOfChannels": 16,
                "PointsPerSecond": 10000,
                "X": 0, "Y": 0, "Z": -1,
                "DrawDebugPoints": true
            },
            "MyLidar2": {
                "SensorType": 6,
                "Enabled" : true,
                "NumberOfChannels": 4,
                "PointsPerSecond": 10000,
                "X": 0, "Y": 0, "Z": -1,
                "DrawDebugPoints": true
            }
        }
    }
}
```

### 传感器特定设置

有关这些传感器设置含义的详细信息，请参阅以下页面：

- [激光雷达传感器设置](lidar.md)
- [距离传感器设置](distance_sensor.md)

##### 用于调试的服务器端可视化

默认情况下，距离传感器触碰的点不会绘制在视口上。如需启用触碰点绘制功能，请通过设置 json 启用`DrawDebugPoints`设置。例如：


```json
"Distance": {
    "SensorType": 5,
    "Enabled" : true,
    ...
    "DrawDebugPoints": true
}
```

## 传感器 API

直接跳转到 [`hello_drone.py`](https://github.com/Microsoft/AirSim/blob/main/PythonClient/multirotor/hello_drone.py) 或 [`hello_drone.cpp`](https://github.com/Microsoft/AirSim/blob/main/HelloDrone/main.cpp) 查看示例用法，或者参阅下文了解完整 API。


### 气压计
```cpp
msr::airlib::BarometerBase::Output getBarometerData(const std::string& barometer_name, const std::string& vehicle_name);
```

```python
barometer_data = client.getBarometerData(barometer_name = "", vehicle_name = "")
```

### 惯性测量单元（IMU）
```cpp
msr::airlib::ImuBase::Output getImuData(const std::string& imu_name = "", const std::string& vehicle_name = "");
```

```python
imu_data = client.getImuData(imu_name = "", vehicle_name = "")
```

### GPS
```cpp
msr::airlib::GpsBase::Output getGpsData(const std::string& gps_name = "", const std::string& vehicle_name = "");
```
```python
gps_data = client.getGpsData(gps_name = "", vehicle_name = "")
```

### 磁力计
```cpp
msr::airlib::MagnetometerBase::Output getMagnetometerData(const std::string& magnetometer_name = "", const std::string& vehicle_name = "");
```
```python
magnetometer_data = client.getMagnetometerData(magnetometer_name = "", vehicle_name = "")
```

### 距离传感器
```cpp
msr::airlib::DistanceSensorData getDistanceSensorData(const std::string& distance_sensor_name = "", const std::string& vehicle_name = "");
```
```python
distance_sensor_data = client.getDistanceSensorData(distance_sensor_name = "", vehicle_name = "")
```

### 激光雷达
有关 Lidar API，请参阅 [lidar 页面](lidar.md)。

