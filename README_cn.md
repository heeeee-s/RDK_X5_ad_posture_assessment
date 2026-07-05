# 青少年体态健康智能筛查系统

> 基于地平线 RDK X5 嵌入式平台与 132GS 双目深度相机的青少年体态健康筛查系统。
> 融合 ROS2 深度视觉节点、2D 骨骼关键点算法与 3D 点云分析，30 秒完成 5 项体态指标的毫米级量化检测，自动生成 HTML 报告并推送至手机浏览器。

---

## 项目组成

本仓库包含三个子目录，共同构成完整的系统：

| 子目录 | 类型 | 说明 |
|--------|------|------|
| `hobot_stereonet/` | ROS2 功能包 | 地平线双目深度估计节点，输出 16UC1 深度图（单位 mm） |
| `mono2d_body_detection/` | ROS2 功能包 | 地平线板端骨骼关键点检测节点，输出 17 个人体关键点 |
| `ad_posture_assessment/` | 应用程序 | 体态分析算法、评分引擎、AI 建议、报告生成、Windows 上位机 |

---

## 功能特性

- **5 项体态指标**：高低肩、驼背、头部倾斜、骨盆倾斜、圆肩，全部输出毫米级量化结果
- **双路融合算法**：2D 骨骼关键点法（高低肩 / 头倾 / 骨盆 / 圆肩）+ 3D 点云分层法（驼背），互补提高准确率
- **板端实时推理**：骨骼检测模型运行于 RDK X5 BPU，无需云端算力
- **AI 健康建议**：优先调用通义千问大模型 API 生成个性化建议，离线时自动切换本地 18 组规则模板
- **HTML 报告**：含体态评分、深度图快照、历史趋势图、AI 建议，手机扫码即可查看
- **图形上位机**：Windows Tkinter GUI，SSH 一键远程启动所有算法进程，无需命令行操作
- **历史档案**：SQLite 本地数据库，支持按姓名查询历史 5 次记录

---

## 系统架构

```
Windows PC 端
└── ad_posture_assessment/gui_controller.py（Tkinter GUI + paramiko SSH）
    └── 一键启动三路进程 / 实时日志过滤 / 报告二维码展示

          │  SSH 指令下发 / 日志回传
          ▼

RDK X5 板端（tROS Humble）
│
├── hobot_stereonet/                     ← ROS2 功能包
│   └── hobot_stereonet 节点
│         输入：132GS 双目相机原始图像
│         输出：/StereoNetNode/stereonet_depth（16UC1 深度图，mm）
│               /StereoNetNode/rectify_left_image（矫正左目 RGB）
│
├── mono2d_body_detection/               ← ROS2 功能包
│   └── mono2d_body_detection 节点
│         输入：左目 RGB 图像
│         输出：/hobot_mono2d_body_detection（17 个骨骼关键点 + 置信度）
│
└── ad_posture_assessment/                 ← 应用程序
    ├── cloud_analysis/
    │   ├── CloudPostureNode             关键点深度换算（高低肩/头倾/骨盆/圆肩）
    │   └── PointCloudProcessor          ROI 点云分层（驼背）
    │       └── PostureResult（5项mm值 + 状态标签 + 评分）
    └── src/
        ├── calc_status_and_score()      0-100 分加权评分
        ├── PostureCoach                 AI 建议（千问 API / 本地模板）
        ├── database.py                  SQLite 历史记录
        └── report_generator.py          HTML 报告 + HTTP :8080
```

---

## 目录结构

```
源码/
├── hobot_stereonet/                # 地平线深度估计 ROS2 包
│   ├── config/                     # 深度网络模型文件（.bin/.hbm）
│   ├── script/
│   │   ├── run_stereo.sh           # 启动深度相机节点
│   │   └── depth_to_pcd/           # 深度图转点云工具脚本
│   └── package.xml
│
├── mono2d_body_detection/          # 地平线骨骼检测 ROS2 包
│   ├── config/                     # 检测模型配置文件
│   ├── launch/
│   │   └── mono2d_from_stereonet.launch.py  # 从深度相机启动骨骼检测
│   ├── script/                     # 关键点滤波等辅助脚本
│   └── package.xml
│
└── ad_posture_assessment/            # 体态分析应用
    ├── main_cloud_rdkx5.py         # 板端主程序入口
    ├── gui_controller.py           # Windows 上位机
    ├── requirements.txt            # 板端 Python 依赖
    ├── .env.example                # 环境变量模板
    ├── remote_runner.sh            # 远程一键启动脚本
    ├── cloud_analysis/             # 算法核心
    │   ├── __init__.py             # analyze_posture_from_cloud()
    │   ├── posture_from_cloud.py   # CloudPostureNode（ROS2 订阅 + 关键点换算）
    │   └── pointcloud_processor.py # 点云分层驼背算法
    ├── config/                     # 关键点模型配置
    └── src/
        ├── interfaces.py           # PostureResult 数据结构
        ├── llm_coach.py            # AI 建议引擎
        ├── database.py             # SQLite 数据库
        └── report_generator.py     # HTML 报告生成器
```

---

## 环境要求

### 板端（RDK X5）

| 项目 | 要求 |
|------|------|
| 操作系统 | Ubuntu 22.04（地平线 tROS 镜像） |
| ROS 版本 | tROS Humble |
| Python | 3.10+ |
| 相机 | 132GS 双目深度相机 |

```bash
pip install dashscope python-dotenv jinja2 qrcode opencv-python
```

### 上位机（Windows PC）

| 项目 | 要求 |
|------|------|
| 操作系统 | Windows 10 / 11 |
| Python | 3.8+ |

```bash
pip install paramiko qrcode pillow
```

---

## 快速开始

### 1. 配置环境变量

```bash
cp ad_posture_assessment/.env.example ad_posture_assessment/templates/.env
# 编辑 .env，填入：
# DASHSCOPE_API_KEY=your_api_key_here   （可选，不填则使用本地规则模板）
```

### 2. 配置上位机连接信息

编辑 `ad_posture_assessment/gui_controller.py` 顶部的 `CONFIG`：

```python
CONFIG = {
    "host":        "192.168.xxx.xxx",   # RDK X5 的局域网 IP
    "port":        22,
    "username":    "root",
    "password":    "your_password",
    "project_dir": "/ad_posture_assessment",
    "report_port": 8080,
}
```

### 3. 启动系统（推荐：上位机一键启动）

在 Windows PC 上运行：

```bash
python ad_posture_assessment/gui_controller.py
```

点击 **连接** → 填写被测者姓名/年龄/性别 → 点击 **一键启动**，系统依次自动启动：

1. `hobot_stereonet` 深度相机节点
2. `mono2d_body_detection` 骨骼检测节点
3. `ad_posture_assessment` 体态分析主程序

检测完成后界面右侧自动显示报告二维码，手机扫码即可查看完整报告。

### 4. 手动启动（板端命令行）

```bash
# 终端 1：启动深度相机
source /opt/tros/humble/setup.bash
bash hobot_stereonet/script/run_stereo.sh

# 终端 2：启动骨骼检测
source /opt/tros/humble/setup.bash
ros2 launch mono2d_body_detection mono2d_from_stereonet.launch.py

# 终端 3：启动体态分析
source /opt/tros/humble/setup.bash
cd ad_posture_assessment
python3 main_cloud_rdkx5.py --name 张三 --age 16 --gender 男
```

---

## 体态评分说明

### 各项指标阈值

| 指标 | 正常 | 轻度 | 中度 | 重度 |
|------|------|------|------|------|
| 高低肩 | < 12 mm | < 25 mm | < 40 mm | ≥ 40 mm |
| 驼背 | < 8 mm | < 20 mm | < 35 mm | ≥ 35 mm |
| 头部倾斜 | < 12 mm | < 22 mm | < 32 mm | ≥ 32 mm |
| 骨盆倾斜 | < 12 mm | < 25 mm | < 38 mm | ≥ 38 mm |
| 圆肩 | < 15 mm | < 30 mm | < 50 mm | ≥ 50 mm |

### 综合评分权重

```
综合分 = 高低肩×25% + 驼背×25% + 骨盆×20% + 头倾×15% + 圆肩×15%
```

| 分数段 | 等级 |
|--------|------|
| 90 – 100 | 优秀 |
| 75 – 89 | 良好 |
| 60 – 74 | 一般 |
| 45 – 59 | 较差 |
| < 45 | 重度异常 |

---

## 算法说明

### 关键点深度换算（面内指标）

```
diff_mm(A, B) = |pixel_yA - pixel_yB| × z_ref / fy
```

`z_ref` 为肩部关键点处深度中位数，`fy` 为相机纵向焦距（来自 CameraInfo）。
采用 **40 帧滑动窗口中位数滤波**消除单帧抖动。

### 点云分层驼背算法

1. 以肩、髋关键点确定躯干 ROI 区域
2. 过滤背景：仅保留与肩部深度差在 200 mm 以内的点
3. 将背部区段（ROI 行归一化 0.50~0.85）分为 20 层，取各层深度中位数
4. 取最近 20% 层均值 → `back_z_min`
5. **驼背量 = max(0, 肩部深度 − back_z_min − 60 mm)**

其中 **60 mm 为解剖基准值**，扣除正常人体肩背自然深度差。

---

## 常见问题

**Q：检测时没有输出数据？**
确认三个进程均已正常启动；被测者需正面站立，距相机 0.8~1.2 米，画面中有完整人体。

**Q：AI 建议显示模板文案？**
检查 `.env` 中 `DASHSCOPE_API_KEY` 是否填写正确，或网络是否可访问阿里云 API。未填写时系统自动使用本地规则模板，属正常降级行为。

**Q：上位机二维码无法打开报告？**
确认手机与 RDK X5 处于同一局域网。如需外网访问，可配置 `CONFIG["ngrok_url"]` 填入 ngrok 公网地址。

**Q：深度图噪点过多？**
检查相机标定参数是否正确加载；确认环境光照充足，避免强逆光；建议以纯色墙面作为背景。

---

## 依赖列表

### 板端 `requirements.txt`

```
mediapipe==0.10.14
opencv-python==4.10.0.84
dashscope==1.20.11
python-dotenv==1.0.1
jinja2==3.1.4
qrcode==7.4.2
```

### 上位机

```
paramiko
qrcode
pillow
```

---

## License

MIT License
