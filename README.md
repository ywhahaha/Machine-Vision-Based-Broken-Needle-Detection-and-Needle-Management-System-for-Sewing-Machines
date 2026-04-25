基于机器视觉的断针检测系统项目文档

## 1. 项目简介

本项目是一个面向缝纫机机针管理场景的断针检测与补针控制系统。系统通过 USB 相机采集机针图像，使用 YOLO 实例分割模型识别机针区域，并结合图像后处理算法计算机针的长度、针柄直径、针杆直径等几何特征。系统将检测结果与库存配置中的机针型号参数进行匹配，判断断针完整性，随后通过串口控制单片机执行断针回收、补针仓释放、LED 光源调节等动作。

系统采用 PyQt5 构建全屏触控式交互界面，集成了相机实时预览、断针检测、库存管理、系统设置、日志查看、屏保、断线重连等功能，适合部署在树莓派设备上。

[![演示视频](https://github.com/user-attachments/assets/cfa4ae42-123d-41a7-84b0-423010bd3841)](https://www.bilibili.com/video/BV1tro9BVE9o?t=11.3)

## 2. 项目目标

- 通过机器视觉自动识别断针图像中的机针结构。
- 根据分割结果计算断针总长、针柄长度、针柄直径、针杆直径等关键尺寸。
- 将检测尺寸与库存中的机针型号进行自动匹配。
- 判断断针是否完整，降低断针残留风险。
- 自动扣减库存并驱动补针机构释放对应型号机针。
- 提供可视化触控界面，支持人工录入、设置、日志和异常处理。

## 3. 主要功能

### 3.1 实时图像采集

系统通过 OpenCV 调用摄像头设备，默认使用 Linux V4L2 接口，并自动查找名称包含 `PC Camera` 的视频设备。相机线程持续读取图像，裁剪 ROI 区域后在主界面实时显示。

### 3.2 运动检测与稳定检测

系统在相机线程中对连续帧进行灰度差分检测：

- 检测到画面运动时，认为机针进入检测区域。
- 连续多帧稳定后，触发断针检测流程。
- 支持手动点击“检测断针”按钮触发检测。
- 检测后设置冷却时间，避免短时间内重复检测。

### 3.3 YOLO 实例分割检测

系统使用 `ultralytics.YOLO` 加载 `weight/best-seg_ncnn_model` 模型，对机针 ROI 图像进行实例分割。分割结果交由 `NeedleDetection` 后处理模块计算几何尺寸。

检测相关参数位于 `default.yaml`：

- `conf`：模型置信度阈值。
- `iou`：IOU 阈值。
- `roi`：相机画面中的检测区域。
- `save_results`：是否保存模型预测可视化结果。
- `show_boxes`：是否显示检测框。
- `save_postprocess_image`：是否保存后处理可视化图。

### 3.4 几何尺寸计算

`function/ncnn_detect.py` 中的 `NeedleDetection` 类负责对分割掩码进行后处理：

- 提取分割掩码轮廓。
- 计算最小外接旋转矩形。
- 沿机针长轴方向采样，计算平均宽度。
- 将像素尺寸按比例转换为毫米尺寸。
- 输出断针总长、针柄长度、针柄直径、针杆直径。

当前代码中使用的像素到毫米比例为：

```python
ratio_1920 = 0.045803
```

同时支持通过 `default.yaml` 中的 `width_correcting_pixels` 和 `length_correcting_pixels` 对宽度、长度进行像素级补偿。

### 3.5 机针型号匹配与完整性判断

系统从 `config.json` 的 `repertory` 字段读取库存机针数据。每条机针记录包含：

- 机针型号。
- 库存数量。
- 仓位编号。
- 机针总长。
- 针柄长度。
- 针柄直径。
- 针杆直径。

检测完成后，系统会根据尺寸误差范围进行匹配：

- 针柄长度误差默认不超过 `1 mm`。
- 针柄直径误差默认不超过 `0.1 mm`。
- 针杆直径误差默认不超过 `0.07 mm`。
- 断针总长需达到完整度阈值，即 `completeness_threshold * 标准总长`。

如果尺寸匹配但断针总长不足，系统提示断针不完整；如果无法匹配任何型号，系统提示未找到匹配型号。

### 3.6 库存管理

库存数据保存在 `config.json` 中。系统界面支持对不同补针仓进行管理：

- 新增机针型号。
- 录入库存数量。
- 自动测量新机针尺寸。
- 修改库存数量。
- 删除机针记录。
- 检测成功后自动扣减库存。

库存界面逻辑主要位于：

```text
function/needleInventoryManagement.py
```

### 3.7 单片机控制

系统通过 `pyserial` 与单片机控制板通信，协议封装位于：

```text
function/ttl.py
```

主要控制能力包括：

- 查询心跳。
- 查询状态。
- 驱动回收电机。
- 驱动补针仓电机。
- 驱动电磁阀。
- 设置 LED 亮度。
- 查询超时。
- 重置系统。

串口设备默认自动搜索：

```text
/dev/ttyUSB*
/dev/ttyACM*
```

### 3.8 异常重连与日志

系统包含相机重连和单片机重连逻辑：

- 相机断开时触发 `CameraReconnectManager`。
- 单片机心跳异常时触发 `ReconnectManager`。
- 默认最多自动重连 10 次。
- 用户可在界面中手动重连。

程序运行日志通过 `function/EmittingStream.py` 重定向到 `logs` 目录，并可在界面中查看。

## 4. 项目目录结构

```text
.
├── main.py                         # 程序主入口，负责主窗口、线程、信号和整体流程
├── config.json                     # 登录、相机、库存、检测阈值等业务配置
├── default.yaml                    # 检测、日志、ROI、LED、屏保等系统参数
├── ncnn_detect_demo.py             # 模型分割与后处理测试示例
├── export.py                       # YOLO 模型导出脚本
├── main.spec                       # PyInstaller 打包配置
├── function/                       # 核心业务逻辑模块
│   ├── camera.py                   # 相机线程、运动检测、断针检测主流程
│   ├── ncnn_detect.py              # 分割结果分析与尺寸计算
│   ├── ttl.py                      # 单片机串口通信协议
│   ├── needleInventoryManagement.py# 机针库存管理界面和配置写入
│   ├── setting.py                  # 系统设置与仓体控制界面
│   ├── HeartbeatThread.py          # 单片机心跳检测与重连
│   ├── camera_reconnect.py         # 相机重连逻辑
│   ├── read_config.py              # YAML 配置读取
│   └── get_resource_path.py        # 资源路径处理
├── ui/                             # PyQt5 界面组件
│   ├── homepage.py                 # 主界面逻辑
│   ├── Ui_homepage.py              # UI 自动生成文件
│   ├── QLogViewer.py               # 日志查看窗口
│   ├── QScreensaver.py             # 屏保组件
│   └── QSliderWithButtons.py       # 滑块控件
├── weight/                         # 模型权重与 NCNN 模型文件
│   ├── best.onnx
│   └── best-seg_ncnn_model/
├── images/                         # 图标资源
├── figs/                           # Logo、屏保和测试图片
├── captured_images/                # 采集保存的图片
├── runs/                           # YOLO 推理输出目录
└── logs/                           # 程序运行日志
```

## 5. 运行环境

### 5.1 推荐硬件环境

- Linux 工控机、嵌入式主机或 ARM Linux 设备。
- USB 相机或工业相机，支持 V4L2 访问。
- 单片机控制板，支持 USB 串口通信。
- LED 光源。
- 断针回收机构与补针仓执行机构。
- 触摸屏或显示器。

### 5.2 推荐软件环境

- Python 3.8 及以上版本。
- Linux 系统，建议 Ubuntu 或嵌入式 Linux。
- OpenCV。
- PyQt5。
- Ultralytics。
- PyYAML。
- pyserial。
- NumPy。

可参考安装命令：

```bash
pip install pyqt5 opencv-python ultralytics pyyaml pyserial numpy ncnn
```

如果部署在无桌面或嵌入式环境，需要确认 Qt 平台插件路径、显示服务、触摸输入和摄像头权限均已正确配置。

## 6. 启动方式

在项目根目录下执行：

```bash
python main.py
```

程序启动后会依次完成：

1. 读取 `default.yaml`。
2. 初始化日志重定向。
3. 加载 PyQt5 主界面。
4. 初始化屏保。
5. 连接单片机控制板。
6. 启动心跳检测线程。
7. 启动相机采集线程。
8. 加载 YOLO 分割模型。

如果需要测试模型分割效果，可运行：

```bash
python ncnn_detect_demo.py
```

## 7. 配置文件说明

### 7.1 `config.json`

该文件保存业务数据，主要包括登录信息、相机配置、库存数据和完整度阈值。

示例结构：

```json
{
    "login": {
        "username": "admin",
        "password": "123"
    },
    "camera": {
        "normal_exposure_value": 0,
        "detect_exposure_value": -10,
        "calibration": 1
    },
    "repertory": [
        {
            "model": "DBx1--11#",
            "quantity": 50,
            "bin_num": "1",
            "needle_total_length": 37.48,
            "needle_handle_length": 15.13,
            "needle_handle_diameter": 1.57,
            "needle_middle_diameter": 0.8
        }
    ],
    "setting": {
        "completeness_threshold": 0.9
    }
}
```

字段说明：

| 字段 | 说明 |
| --- | --- |
| `login` | 登录用户名、密码和记住登录状态 |
| `camera` | 相机曝光、标定等参数 |
| `repertory` | 机针库存列表 |
| `model` | 机针型号 |
| `quantity` | 当前库存数量 |
| `bin_num` | 补针仓编号 |
| `needle_total_length` | 机针总长，单位 mm |
| `needle_handle_length` | 针柄长度，单位 mm |
| `needle_handle_diameter` | 针柄直径，单位 mm |
| `needle_middle_diameter` | 针杆直径，单位 mm |
| `completeness_threshold` | 断针完整度阈值 |

### 7.2 `default.yaml`

该文件保存系统运行参数和算法参数。

主要参数说明：

| 参数 | 说明 |
| --- | --- |
| `low_light` | 待机状态 LED 亮度 |
| `high_light` | 断针检测时 LED 亮度 |
| `shank_length_error` | 针柄长度允许误差 |
| `shank_diameter_error` | 针柄直径允许误差 |
| `shaft_diameter_error` | 针杆直径允许误差 |
| `max_buffered_lines` | 日志缓存行数 |
| `flush_interval` | 日志定时写入间隔 |
| `max_log_days` | 日志保留天数 |
| `roi` | 检测区域 `[x, y, w, h]` |
| `conf` | YOLO 置信度阈值 |
| `iou` | YOLO IOU 阈值 |
| `save_results` | 是否保存推理结果 |
| `show_boxes` | 是否显示检测框 |
| `save_postprocess_image` | 是否保存后处理可视化结果 |
| `width_correcting_pixels` | 宽度像素补偿 |
| `length_correcting_pixels` | 长度像素补偿 |
| `inactive_timeout` | 屏保触发时间 |
| `switch_interval` | 屏保图片切换间隔 |
| `show_details` | 是否打印详细日志 |

## 8. 核心检测流程

系统自动检测流程如下：

```text
启动系统
  ↓
初始化相机、模型、单片机、UI
  ↓
相机线程持续采集画面
  ↓
裁剪 ROI 区域并显示到主界面
  ↓
灰度差分判断画面是否有运动
  ↓
画面稳定达到阈值
  ↓
切换 LED 亮度和相机曝光
  ↓
采集低曝光清晰图像
  ↓
YOLO 实例分割
  ↓
分割掩码后处理
  ↓
计算断针尺寸
  ↓
匹配库存型号
  ↓
判断断针完整性
  ↓
扣减库存
  ↓
驱动回收机构和补针仓
  ↓
更新界面库存状态
```

## 9. 关键模块说明

### 9.1 `main.py`

主程序入口，负责系统级对象初始化和信号连接：

- 创建主窗口。
- 初始化配置。
- 启动相机线程。
- 启动单片机心跳线程。
- 处理登录、设置、库存管理、日志查看等窗口。
- 处理相机或单片机断连事件。
- 捕获程序异常并安全退出。

### 9.2 `function/camera.py`

系统检测流程的核心模块，主要包含：

- 摄像头自动连接。
- 图像采集线程。
- ROI 裁剪。
- 运动检测。
- 稳定后触发断针检测。
- 调用 YOLO 模型。
- 调用后处理计算尺寸。
- 匹配库存型号。
- 更新库存。
- 控制 LED、回收电机和补针仓电机。

### 9.3 `function/ncnn_detect.py`

模型结果分析模块，主要职责：

- 调用模型进行分割预测。
- 读取分割掩码。
- 根据轮廓计算旋转矩形。
- 沿长轴计算平均宽度。
- 计算各类别的长度和宽度。
- 输出用于型号匹配的尺寸数据。

### 9.4 `function/ttl.py`

单片机通信模块，封装串口协议：

- 协议起始码：`AA55`。
- 协议结束码：`EF`。
- 支持校验和验证。
- 支持电机、电磁阀、LED、心跳、状态查询等命令。

### 9.5 `function/needleInventoryManagement.py`

库存管理模块，负责：

- 新增机针型号。
- 修改库存数量。
- 删除库存记录。
- 调用相机线程测量新机针尺寸。
- 将库存数据写回 `config.json`。

### 9.6 `function/setting.py`

系统设置模块，负责：

- 设置断针完整度阈值。
- 手动打开回收仓门。
- 手动打开补针仓门。
- 控制回收轮转动。
- 控制 1 至 4 号补针仓释放。
- 重置系统。

## 10. 模型与标定说明

### 10.1 模型文件

当前程序加载的模型路径为：

```text
weight/best-seg_ncnn_model
```

同时项目中保留了 ONNX 模型：

```text
weight/best.onnx
```

如需重新导出模型，可参考 `export.py`，该脚本使用 Ultralytics 的 `model.export()` 接口导出 ONNX 模型。

### 10.2 ROI 标定

ROI 参数位于 `default.yaml`：

```yaml
roi: [277, 472, 1270, 315]
```

含义为：

```text
[x, y, width, height]
```

如果相机位置、镜头焦距或检测区域发生变化，需要重新调整 ROI。

### 10.3 尺寸标定

尺寸转换依赖像素到毫米比例：

```python
ratio_1920 = 0.045803
```

如果更换相机、分辨率、镜头或安装距离，需要重新标定该比例。建议使用已知长度的标准机针或标定尺进行测量，计算：

```text
毫米/像素 = 实际长度(mm) / 图像中测得像素长度(px)
```

## 11. 常见问题与排查

### 11.1 相机无法打开

可检查：

- 相机是否被系统识别。
- 是否存在 `/dev/video*` 设备。
- `v4l2-ctl --list-devices` 是否能看到 `PC Camera`。
- 当前用户是否有摄像头访问权限。
- 相机是否被其他程序占用。

### 11.2 单片机连接失败

可检查：

- 是否存在 `/dev/ttyUSB*` 或 `/dev/ttyACM*`。
- 串口波特率是否为 `9600`。
- 当前用户是否有串口权限。
- 控制板协议是否与 `function/ttl.py` 一致。
- USB 线和供电是否稳定。

### 11.3 检测不到机针

可检查：

- ROI 是否覆盖机针区域。
- LED 亮度是否合适。
- 相机曝光是否合适。
- 模型文件是否存在。
- `conf` 阈值是否过高。
- 机针图像是否清晰。

### 11.4 型号匹配失败

可检查：

- `config.json` 中库存尺寸是否准确。
- 像素到毫米比例是否正确。
- `width_correcting_pixels` 和 `length_correcting_pixels` 是否需要调整。
- 允许误差范围是否过小。
- 断针是否确实完整。

### 11.5 中文显示或终端乱码

项目源码包含中文界面文本和注释。若在 Windows 终端中出现乱码，应检查：

- 文件编码是否为 UTF-8。
- 终端编码是否为 UTF-8。
- 编辑器是否以正确编码打开文件。


## 13. 总结

本项目将机器视觉检测、深度学习实例分割、图像几何测量、库存管理和机电控制结合起来，实现了断针检测与补针执行的一体化流程。系统的核心优势在于能够自动采集断针图像、计算断针关键尺寸、匹配库存型号并联动硬件机构，从而提升断针管理效率和安全性。
