---
title: "WSL2 + Docker + TensorFlow：从环境搭建到 STM32F407 模型部署完整教程"
date: 2026-07-06
description: "Windows Subsystem for Linux 2 配合 Docker 容器化 TensorFlow 开发环境，从安装 WSL、配置 Docker Desktop、拉取 NVIDIA TensorFlow 官方容器，到训练关键词检测模型、TensorBoard 可视化、模型量化，最终部署到 STM32F407VGT6 的完整流程"
image: WSL2+Docker+TensorFlow教程博客封面.png
categories:
  - "深度学习"
  - "边缘AI"
tags:
  - "WSL2"
  - "Docker"
  - "TensorFlow"
  - "STM32"
  - "模型训练"
  - "TensorBoard"
  - "边缘AI"
---

## 前言

在 Windows 上进行深度学习开发，传统方案是直接安装 CUDA + cuDNN + TensorFlow，但版本兼容性问题频发，环境配置耗时且容易出错。**WSL2 + Docker** 的组合提供了更优雅的解决方案：

- **WSL2**：Windows 内核级 Linux 子系统，接近原生 Linux 性能
- **Docker**：容器化隔离，环境一键拉取、版本锁定、跨平台复用
- **NVIDIA TensorFlow 官方容器**：预装 CUDA、cuDNN、TensorFlow，版本严格匹配

本文以**关键词检测模型训练**为例，完整演示从环境搭建到 STM32F407 部署的全流程。

> [!NOTE]
> 本文**并非保姆级教程**，而是对整个流程的简单记录与方案尝试经验。部分步骤假设读者已具备基础的 Linux/Docker/TensorFlow 知识。如遇问题，请参考官方文档或搜索具体错误信息。

---

## 一、安装 WSL 与 Linux 子系统

### 1.1 启用 WSL 功能

以管理员身份打开 PowerShell，执行：

```powershell
wsl --install
```

这条命令会自动：

1. 启用 WSL 功能
2. 启用虚拟机平台
3. 下载并安装 Ubuntu（默认）

![安装wsl与linux子系统](安装wsl与linux子系统.png)

安装完成后重启计算机。

### 1.2 设置 WSL2 为默认版本

```powershell
wsl --set-default-version 2
```

### 1.3 验证 WSL 版本

```powershell
wsl -l -v
```

输出应显示 Ubuntu 运行在 WSL 2。

---

## 二、安装 Docker Desktop

### 2.1 下载 Docker Desktop

访问 Docker 官方下载页面：

> [Docker Desktop for Windows 下载地址](https://docs.docker.com/desktop/setup/install/windows-install/)

选择 Windows amd64 版本下载安装包。

![安装docker桌面版](安装docker桌面版.png)

### 2.2 安装与配置

安装时勾选 **"Use WSL 2 instead of Hyper-V"** 选项。安装完成后重启计算机。

首次启动时，Docker Desktop 会提示是否启用 WSL 2 集成——选择 **Yes**。

---

## 三、开启 Docker 的 WSL 扩展

### 3.1 Docker Desktop 设置

打开 Docker Desktop → Settings → Resources → WSL Integration：

![开启docker的wsl扩展](开启docker的wsl扩展.png)

确保：

- **Enable integration with my default WSL distro** 已勾选
- Ubuntu 选项已启用

这样 Ubuntu 子系统内可以直接使用 `docker` 命令。

### 3.2 验证 Docker 集成

进入 Ubuntu 终端：

```bash
docker --version
docker run hello-world
```

![验证docker集成](验证docker集成.png)

如果显示 Docker 版本并成功拉取运行 hello-world 容器，说明集成完成。

---

## 四、拉取 NVIDIA TensorFlow 官方容器

### 4.1 容器地址

NVIDIA NGC 提供官方 TensorFlow 容器，预装完整 GPU 加速环境：

> [NVIDIA TensorFlow 25.02-tf2-py3 容器地址](https://catalog.ngc.nvidia.com/orgs/nvidia/-/containers/tensorflow/25.02-tf2-py3)

### 4.2 拉取容器

在 Ubuntu 终端执行：

```bash
docker pull nvcr.io/nvidia/tensorflow:25.02-tf2-py3
```

![下载官方tensorflow的docker容器](下载官方tensorflow的docker容器.png)

镜像大小约 5GB，包含：

- TensorFlow 2.x
- CUDA 12.x
- cuDNN 9.x
- Python 3.x
- Jupyter Notebook
- TensorBoard

### 4.3 验证容器安装

```bash
docker images
```

![验证容器安装](验证容器安装.png)

确认 `nvcr.io/nvidia/tensorflow` 镜像已存在。

---

## 五、启动 TensorFlow 容器

### 5.1 容器启动命令

在项目目录下执行（`${PWD}` 会挂载当前目录到容器 `/workspace`）：

```bash
docker run --gpus all \
  --ipc=host \
  --ulimit memlock=-1 \
  --ulimit stack=67108864 \
  --rm \
  -it -p 8888:8888 \
  -p 6006:6006 \
  -v ${PWD}:/workspace \
  nvcr.io/nvidia/tensorflow:25.02-tf2-py3
```

参数解析：

| 参数                      | 作用                                       |
| ------------------------- | ------------------------------------------ |
| `--gpus all`              | 允许容器访问所有 GPU                       |
| `--ipc=host`              | 使用主机 IPC namespace，共享内存通信更高效 |
| `--ulimit memlock=-1`     | 无限锁定内存，适合大模型训练               |
| `--ulimit stack=67108864` | 64MB 栈大小                                |
| `--rm`                    | 容器退出后自动删除                         |
| `-it`                     | 交互模式 + 伪终端                          |
| `-p 8888:8888`            | Jupyter Notebook 端口映射                  |
| `-p 6006:6006`            | TensorBoard 端口映射                       |
| `-v ${PWD}:/workspace`    | 当前目录挂载到容器工作空间                 |

![运行docker](运行docker.png)

### 5.2 容器运行状态

```bash
docker ps
```

![容器运行状态](容器运行状态.png)

容器运行后，可以直接在容器内执行 Python 脚本。

---

## 六、准备训练项目

### 6.1 项目目录结构

![开始训练文件目录结构](开始训练文件目录结构.png)

典型 TensorFlow Lite Micro 训练项目结构：

```
/workspace/Projects/tflm_train/
├── train_micro_speech_model.py   # 训练脚本
├── data/                          # 数据集目录
├── logs/                          # TensorBoard 日志
├── models/                        # 输出模型
└── train_log.txt                  # 训练日志
```

### 6.2 数据集准备

关键词检测模型使用 Google Speech Commands 数据集，包含 30+ 关键词的数千条语音样本。

---

## 七、开始模型训练

### 7.1 训练命令

在容器内执行：

```bash
python3 train_micro_speech_model.py \
  --skip_clone --skip_download \
  --clean \
  --wanted_words=yes,no,up,down,left,right,on,off,stop,go \
  --model_architecture=tiny_conv \
  --training_steps=8000,10000,7000 \
  --learning_rate=0.001,0.0005,0.0001 \
  2>&1 | tee train_log.txt
```

参数解析：

| 参数                                  | 作用                                                 |
| ------------------------------------- | ---------------------------------------------------- | ---------------------- |
| `--skip_clone --skip_download`        | 跳过仓库克隆和数据下载（已准备好）                   |
| `--clean`                             | 清理旧的训练输出                                     |
| `--wanted_words`                      | 目标关键词：yes/no/up/down/left/right/on/off/stop/go |
| `--model_architecture=tiny_conv`      | 使用 tiny_conv 架构（适合嵌入式）                    |
| `--training_steps=8000,10000,7000`    | 三阶段训练步数                                       |
| `--learning_rate=0.001,0.0005,0.0001` | 三阶段学习率衰减                                     |
| `2>&1                                 | tee train_log.txt`                                   | 输出同时打印和写入日志 |

![开始训练](开始训练.png)

### 7.2 观察训练状态

训练过程中会打印：

```
Step 1000: loss=1.23, accuracy=0.78
Step 2000: loss=0.89, accuracy=0.85
...
```

![观察训练状态](观察训练状态.png)

---

## 八、TensorBoard 可视化训练

### 8.1 启动 TensorBoard

在容器内执行：

```bash
tensorboard --logdir=/workspace/Projects/tflm_train/logs --bind_all --port=6006
```

![开启tensorboard](开启tensorboard.png)

### 8.2 访问 TensorBoard

浏览器打开：

> [http://localhost:6006](http://localhost:6006)

可以看到：

- **SCALARS**：loss、accuracy 曲线
- **IMAGES**：模型结构可视化
- **GRAPHS**：计算图拓扑
- **DISTRIBUTIONS**：权重分布
- **HISTOGRAMS**：权重直方图

---

## 九、训练结果与模型量化

### 9.1 最终训练结果

![最终训练结果](最终训练结果.png)

训练完成后，输出：

- `.tflite` 浮点模型（约 50KB）
- 训练日志 `train_log.txt`
- TensorBoard 日志目录

### 9.2 模型量化

TensorFlow Lite 支持 **训练后量化**：

```bash
python3 quantize_model.py \
  --input_model=models/micro_speech.tflite \
  --output_model=models/micro_speech_quantized.tflite
```

量化后模型：

- 大小压缩至约 **18KB**
- 精度损失约 1-2%
- 适合 MCU 运行

![量化结果](量化结果.png)

---

## 十、STM32F407 部署测试

### 10.1 模型集成

将量化后的 `.tflite` 模型转换为 C 数组：

```bash
xxd -i models/micro_speech_quantized.tflite > micro_speech_model_data.cc
```

### 10.2 STM32CubeAI 导入

在 STM32CubeMX 中：

1. 添加 X-CUBE-AI 组件
2. 导入 `.tflite` 模型
3. 生成代码

![cubemx导入模型解析](cubemx导入模型解析.png)

CubeMX 会自动解析模型结构并生成推理代码框架。

### 10.3 测试结果

![stm32f407测试结果](stm32f407测试结果.png)

在 STM32F407VGT6 上运行关键词检测：

- **推理时间**：约 30ms（不含音频预处理）
- **内存占用**：约 50KB RAM
- **准确率**：实测 85%+

---

## 十一、环境清理与常见问题

### 11.1 容器清理

训练完成后，容器会因 `--rm` 参数自动删除。镜像保留：

```bash
# 查看镜像
docker images

# 删除镜像（如不再需要）
docker rmi nvcr.io/nvidia/tensorflow:25.02-tf2-py3
```

### 11.2 常见问题

**Q: WSL2 无法识别 GPU？**

确保：

- Windows 已安装 NVIDIA 驱动（最新版）
- Docker Desktop 的 WSL 集成已开启
- 容器启动时添加 `--gpus all`

**Q: TensorBoard 无法访问？**

检查：

- 容器是否正确映射 `-p 6006:6006`
- TensorBoard 是否使用 `--bind_all`
- Windows 防火墙是否允许端口

**Q: 训练速度很慢？**

排查：

- GPU 是否被正确识别（`nvidia-smi`）
- 数据集是否在容器内（避免跨挂载读写）
- `--ipc=host` 参数是否添加

---

## 参考资料

- [WSL2 官方文档](https://docs.microsoft.com/en-us/windows/wsl/)
- [Docker Desktop for Windows](https://docs.docker.com/desktop/setup/install/windows-install/)
- [NVIDIA TensorFlow 容器](https://catalog.ngc.nvidia.com/orgs/nvidia/-/containers/tensorflow/25.02-tf2-py3)
- [TensorFlow Lite Micro](https://www.tensorflow.org/lite/microcontrollers)
- [STM32CubeAI](https://www.st.com/en/embedded-software/x-cube-ai.html)
