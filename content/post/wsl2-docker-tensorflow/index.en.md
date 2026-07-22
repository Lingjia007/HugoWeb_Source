---
title: "WSL2 + Docker + TensorFlow: Complete Tutorial from Environment Setup to STM32F407 Model Deployment"
date: 2026-07-06
description: "Windows Subsystem for Linux 2 combined with Docker containerized TensorFlow development environment, from installing WSL, configuring Docker Desktop, pulling NVIDIA TensorFlow official container, to training keyword detection model, TensorBoard visualization, model quantization, and finally deploying to STM32F407VGT6"
image: WSL2+Docker+TensorFlow教程博客封面.png
categories:
  - "Deep Learning"
  - "Edge AI"
tags:
  - "WSL2"
  - "Docker"
  - "TensorFlow"
  - "STM32"
  - "Model Training"
  - "TensorBoard"
  - "Edge AI"
---

## Introduction

When developing deep learning on Windows, the traditional approach is to install CUDA + cuDNN + TensorFlow directly, but version compatibility issues occur frequently, and environment configuration is time-consuming and error-prone. The **WSL2 + Docker** combination provides a more elegant solution:

- **WSL2**: Windows kernel-level Linux subsystem, near-native Linux performance
- **Docker**: Containerized isolation, one-click environment pull, version lock, cross-platform reuse
- **NVIDIA TensorFlow Official Container**: Pre-installed CUDA, cuDNN, TensorFlow with strictly matched versions

This article uses **keyword detection model training** as an example to demonstrate the complete workflow from environment setup to STM32F407 deployment.

> [!NOTE]
> This article is **not a step-by-step tutorial**, but rather a simple record of the entire process and experience from solution attempts. Some steps assume the reader has basic Linux/Docker/TensorFlow knowledge. If you encounter issues, please refer to the official documentation or search for specific error messages.

---

## 1. Installing WSL and Linux Subsystem

### 1.1 Enable WSL Feature

Open PowerShell as administrator and execute:

```powershell
wsl --install
```

This command will automatically:

1. Enable the WSL feature
2. Enable the Virtual Machine Platform
3. Download and install Ubuntu (default)

![安装wsl与linux子系统](安装wsl与linux子系统.png)

Restart the computer after installation completes.

### 1.2 Set WSL2 as Default Version

```powershell
wsl --set-default-version 2
```

### 1.3 Verify WSL Version

```powershell
wsl -l -v
```

The output should show Ubuntu running on WSL 2.

---

## 2. Installing Docker Desktop

### 2.1 Download Docker Desktop

Visit the Docker official download page:

> [Docker Desktop for Windows Download](https://docs.docker.com/desktop/setup/install/windows-install/)

Select the Windows amd64 version to download the installer.

![安装docker桌面版](安装docker桌面版.png)

### 2.2 Installation and Configuration

During installation, check the **"Use WSL 2 instead of Hyper-V"** option. Restart the computer after installation completes.

When Docker Desktop starts for the first time, it will prompt whether to enable WSL 2 integration — select **Yes**.

---

## 3. Enabling Docker WSL Integration

### 3.1 Docker Desktop Settings

Open Docker Desktop → Settings → Resources → WSL Integration:

![开启docker的wsl扩展](开启docker的wsl扩展.png)

Ensure:

- **Enable integration with my default WSL distro** is checked
- The Ubuntu option is enabled

This allows the `docker` command to be used directly within the Ubuntu subsystem.

### 3.2 Verify Docker Integration

Enter the Ubuntu terminal:

```bash
docker --version
docker run hello-world
```

![验证docker集成](验证docker集成.png)

If the Docker version is displayed and the hello-world container is successfully pulled and run, the integration is complete.

---

## 4. Pulling the NVIDIA TensorFlow Official Container

### 4.1 Container URL

NVIDIA NGC provides official TensorFlow containers with the complete GPU acceleration environment pre-installed:

> [NVIDIA TensorFlow 25.02-tf2-py3 Container](https://catalog.ngc.nvidia.com/orgs/nvidia/-/containers/tensorflow/25.02-tf2-py3)

### 4.2 Pull the Container

Execute in the Ubuntu terminal:

```bash
docker pull nvcr.io/nvidia/tensorflow:25.02-tf2-py3
```

![下载官方tensorflow的docker容器](下载官方tensorflow的docker容器.png)

The image size is approximately 5GB, containing:

- TensorFlow 2.x
- CUDA 12.x
- cuDNN 9.x
- Python 3.x
- Jupyter Notebook
- TensorBoard

### 4.3 Verify Container Installation

```bash
docker images
```

![验证容器安装](验证容器安装.png)

Confirm that the `nvcr.io/nvidia/tensorflow` image exists.

---

## 5. Starting the TensorFlow Container

### 5.1 Container Start Command

Execute in the project directory (`${PWD}` will mount the current directory to the container's `/workspace`):

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

Parameter explanation:

| Parameter                   | Description                                          |
| --------------------------- | ---------------------------------------------------- |
| `--gpus all`                | Allow the container to access all GPUs               |
| `--ipc=host`                | Use host IPC namespace for more efficient shared memory communication |
| `--ulimit memlock=-1`       | Unlimited memory lock, suitable for large model training |
| `--ulimit stack=67108864`   | 64MB stack size                                      |
| `--rm`                      | Automatically remove the container on exit           |
| `-it`                       | Interactive mode + pseudo-TTY                        |
| `-p 8888:8888`              | Jupyter Notebook port mapping                        |
| `-p 6006:6006`              | TensorBoard port mapping                             |
| `-v ${PWD}:/workspace`      | Mount current directory to container workspace       |

![运行docker](运行docker.png)

### 5.2 Container Running Status

```bash
docker ps
```

![容器运行状态](容器运行状态.png)

Once the container is running, you can execute Python scripts directly inside the container.

---

## 6. Preparing the Training Project

### 6.1 Project Directory Structure

![开始训练文件目录结构](开始训练文件目录结构.png)

Typical TensorFlow Lite Micro training project structure:

```
/workspace/Projects/tflm_train/
├── train_micro_speech_model.py   # Training script
├── data/                          # Dataset directory
├── logs/                          # TensorBoard logs
├── models/                        # Output models
└── train_log.txt                  # Training log
```

### 6.2 Dataset Preparation

The keyword detection model uses the Google Speech Commands dataset, which contains thousands of speech samples across 30+ keywords.

---

## 7. Starting Model Training

### 7.1 Training Command

Execute inside the container:

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

Parameter explanation:

| Parameter                               | Description                                                    |
| --------------------------------------- | -------------------------------------------------------------- |
| `--skip_clone --skip_download`          | Skip repository cloning and data download (already prepared)   |
| `--clean`                               | Clean previous training output                                 |
| `--wanted_words`                        | Target keywords: yes/no/up/down/left/right/on/off/stop/go      |
| `--model_architecture=tiny_conv`        | Use tiny_conv architecture (suitable for embedded)             |
| `--training_steps=8000,10000,7000`      | Three-stage training steps                                    |
| `--learning_rate=0.001,0.0005,0.0001`   | Three-stage learning rate decay                                |
| `2>&1 \| tee train_log.txt`             | Output to both stdout and log file                             |

![开始训练](开始训练.png)

### 7.2 Observe Training Status

During training, the following will be printed:

```
Step 1000: loss=1.23, accuracy=0.78
Step 2000: loss=0.89, accuracy=0.85
...
```

![观察训练状态](观察训练状态.png)

---

## 8. TensorBoard Training Visualization

### 8.1 Start TensorBoard

Execute inside the container:

```bash
tensorboard --logdir=/workspace/Projects/tflm_train/logs --bind_all --port=6006
```

![开启tensorboard](开启tensorboard.png)

### 8.2 Access TensorBoard

Open in browser:

> [http://localhost:6006](http://localhost:6006)

You can see:

- **SCALARS**: loss, accuracy curves
- **IMAGES**: model structure visualization
- **GRAPHS**: computation graph topology
- **DISTRIBUTIONS**: weight distributions
- **HISTOGRAMS**: weight histograms

---

## 9. Training Results and Model Quantization

### 9.1 Final Training Results

![最终训练结果](最终训练结果.png)

After training completes, the outputs are:

- `.tflite` floating-point model (approximately 50KB)
- Training log `train_log.txt`
- TensorBoard log directory

### 9.2 Model Quantization

TensorFlow Lite supports **post-training quantization**:

```bash
python3 quantize_model.py \
  --input_model=models/micro_speech.tflite \
  --output_model=models/micro_speech_quantized.tflite
```

Quantized model:

- Size compressed to approximately **18KB**
- Accuracy loss of about 1-2%
- Suitable for MCU execution

![量化结果](量化结果.png)

---

## 10. STM32F407 Deployment Testing

### 10.1 Model Integration

Convert the quantized `.tflite` model to a C array:

```bash
xxd -i models/micro_speech_quantized.tflite > micro_speech_model_data.cc
```

### 10.2 STM32CubeAI Import

In STM32CubeMX:

1. Add the X-CUBE-AI component
2. Import the `.tflite` model
3. Generate code

![cubemx导入模型解析](cubemx导入模型解析.png)

CubeMX will automatically parse the model structure and generate the inference code framework.

### 10.3 Test Results

![stm32f407测试结果](stm32f407测试结果.png)

Running keyword detection on STM32F407VGT6:

- **Inference time**: approximately 30ms (excluding audio preprocessing)
- **Memory usage**: approximately 50KB RAM
- **Accuracy**: measured 85%+

---

## 11. Environment Cleanup and Common Issues

### 11.1 Container Cleanup

After training completes, the container will be automatically removed due to the `--rm` parameter. The image is retained:

```bash
# View images
docker images

# Remove image (if no longer needed)
docker rmi nvcr.io/nvidia/tensorflow:25.02-tf2-py3
```

### 11.2 Common Issues

**Q: WSL2 cannot detect GPU?**

Ensure:

- NVIDIA driver (latest version) is installed on Windows
- Docker Desktop WSL integration is enabled
- `--gpus all` is added when starting the container

**Q: TensorBoard cannot be accessed?**

Check:

- Container correctly maps `-p 6006:6006`
- TensorBoard uses `--bind_all`
- Windows firewall allows the port

**Q: Training speed is very slow?**

Investigate:

- GPU is correctly detected (`nvidia-smi`)
- Dataset is inside the container (avoid cross-mount read/write)
- `--ipc=host` parameter is added

---

## References

- [WSL2 Official Documentation](https://docs.microsoft.com/en-us/windows/wsl/)
- [Docker Desktop for Windows](https://docs.docker.com/desktop/setup/install/windows-install/)
- [NVIDIA TensorFlow Container](https://catalog.ngc.nvidia.com/orgs/nvidia/-/containers/tensorflow/25.02-tf2-py3)
- [TensorFlow Lite Micro](https://www.tensorflow.org/lite/microcontrollers)
- [STM32CubeAI](https://www.st.com/en/embedded-software/x-cube-ai.html)
