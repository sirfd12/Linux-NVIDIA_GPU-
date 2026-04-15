# NVIDIA A100 安装 CUDA 工具包与配置全流程（生产环境标准）

> **前置条件**：NVIDIA 企业级驱动已正确安装（`nvidia-smi` 可正常显示 A100 显卡信息）
> **适用显卡**：NVIDIA A100-SXM4-80GB (Ampere 架构，计算能力 8.0)
> **推荐 CUDA 版本**：CUDA 12.4（与驱动 550+ 搭配，稳定性最佳）
> **安装方式**：Runfile 本地安装（跳过驱动安装）

------

## 📌 第一部分：生产环境背景知识

### 1. A100 与 RTX 系列的关键差异

| 对比维度       | RTX 2080 Ti / 3090 (消费级)        | A100 (数据中心级)                            |
| :------------- | :--------------------------------- | :------------------------------------------- |
| **显存类型**   | GDDR6 / GDDR6X                     | HBM2e (高带宽内存)                           |
| **显存容量**   | 11GB - 24GB                        | 40GB / 80GB                                  |
| **ECC 纠错**   | 不支持                             | **原生支持 ECC**                             |
| **NVLink**     | 双卡桥接，带宽 ~100 GB/s           | 多卡全互联，通过 NVSwitch 实现 600 GB/s      |
| **MIG 多实例** | 不支持                             | **支持 MIG**，可切分为最多 7 个独立 GPU 实例 |
| **驱动模式**   | WDDM / 默认模式                    | **TCC 模式**（纯计算模式，剥离图形栈）       |
| **散热方式**   | 主动风扇                           | 被动散热，依赖机箱暴力扇风道                 |
| **保修政策**   | 3 年有限保修，数据中心使用可能拒保 | 5 年 NBD 企业级维保                          |

### 2. 驱动 vs CUDA 工具包的区别（重申）

| 维度             | NVIDIA 驱动  | CUDA 工具包                        |
| :--------------- | :----------- | :--------------------------------- |
| **运行层级**     | 内核态       | 用户态                             |
| **是否必须安装** | **是**       | 仅当需要编译或运行 CUDA 程序时需要 |
| **验证命令**     | `nvidia-smi` | `nvcc --version`                   |

### 3. 安装前检查

bash

```
# 1. 确认驱动已正常工作
nvidia-smi

# 2. 记录当前驱动最高支持的 CUDA 版本
# 输出右上角 "CUDA Version: 12.4" 表示该驱动最高支持 CUDA 12.4

# 3. 确认 GPU 处于 TCC 模式（生产环境推荐）
nvidia-smi -q | grep -i "Compute Mode"
# 期望输出：Compute Mode : Default (多进程共享)
```



------

## 📌 第二部分：下载 CUDA 工具包

### 1. 访问 NVIDIA CUDA 官方归档页

网址：https://developer.nvidia.com/cuda-toolkit-archive

### 2. 选择版本与平台

以 **CUDA 12.4.0** 为例，按以下参数选择：

| 选项             | 值                                                       |
| :--------------- | :------------------------------------------------------- |
| Operating System | Linux                                                    |
| Architecture     | x86_64                                                   |
| Distribution     | 根据你的系统选择（如 CentOS 7 / RHEL 8 / Rocky Linux 9） |
| Installer Type   | **runfile (local)**                                      |

### 3. 获取下载链接并下载

bash

```
# 示例链接，请从官网复制实际地址
wget https://developer.download.nvidia.com/compute/cuda/12.4.0/local_installers/cuda_12.4.0_550.54.15_linux.run
```



------

## 📌 第三部分：安装 CUDA 工具包

### 1. 赋予执行权限

bash

```
chmod +x cuda_12.4.0_550.54.15_linux.run
```



### 2. 执行安装（**关键：取消勾选 Driver**）

bash

```
sudo sh cuda_12.4.0_550.54.15_linux.run
```



**交互式界面操作指南**：

text

```
┌────────────────────────────────────────────────────────────────┐
│ CUDA Toolkit 12.4.0                                            │
│                                                                │
│ [ ] Driver           <─── 务必取消勾选！                        │
│ [X] CUDA Toolkit 12.4.0                                        │
│ [X] CUDA Samples 12.4.0                                        │
│ [X] CUDA Documentation 12.4.0                                  │
│                                                                │
│ Options:                                                       │
│   Toolkit Options:                                             │
│     Change Toolkit Install Path  [/usr/local/cuda-12.4]        │
│                                                                │
│                         [ Install ]                            │
└────────────────────────────────────────────────────────────────┘
```



**为什么必须取消 Driver？**

- 生产环境通常已通过企业级镜像或厂商工具包预装并验证了驱动版本。
- 若勾选 Driver，安装程序会用自带的驱动覆盖现有驱动，可能导致版本不匹配、内核模块冲突，甚至触发 **Xid 43** 或 **掉卡**。

------

## 📌 第四部分：配置环境变量

### 1. 编辑 Shell 配置文件

bash

```
vi ~/.bashrc
```



### 2. 在文件末尾添加以下内容

bash

```
# CUDA 12.4 Environment Variables
export CUDA_HOME=/usr/local/cuda-12.4
export PATH=$CUDA_HOME/bin:$PATH
export LD_LIBRARY_PATH=$CUDA_HOME/lib64:$LD_LIBRARY_PATH
```



### 3. 创建版本无关的软链接（生产环境必备）

bash

```
sudo ln -sf /usr/local/cuda-12.4 /usr/local/cuda
```



- **目的**：大多数 AI 框架和容器镜像默认在 `/usr/local/cuda` 查找库文件。
- **版本切换**：未来升级 CUDA 版本时，仅需修改软链接指向，无需改动所有环境变量。

### 4. 使配置生效

bash

```
source ~/.bashrc
```



------

## 📌 第五部分：验证安装

### 1. 验证 CUDA 编译器

bash

```
nvcc --version
```



**预期输出**：

text

```
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2024 NVIDIA Corporation
Built on Wed_Feb_21_10:33:58_PST_2024
Cuda compilation tools, release 12.4, V12.4.131
Build cuda_12.4.r12.4/compiler.33961245_0
```



### 2. 编译并运行 CUDA 示例（推荐）

bash

```
cd ~/NVIDIA_CUDA-12.4_Samples/1_Utilities/deviceQuery
make
./deviceQuery
```



**预期输出末尾**：

text

```
deviceQuery, CUDA Driver = CUDART, CUDA Driver Version = 12.4, CUDA Runtime Version = 12.4, NumDevs = 8
Result = PASS
```



### 3. 验证深度学习框架（以 PyTorch 为例）

bash

```
pip3 install torch torchvision torchaudio
python3 -c "import torch; print(torch.cuda.is_available())"
```



**预期输出**：`True`

------

## 📌 第六部分：A100 生产环境专项优化设置

### 1. 开启持久模式（必须）

bash

```
sudo nvidia-smi -pm 1
```



- **目的**：保持驱动常驻内存，避免任务结束时卸载驱动引发的延迟与 **Xid 43** 错误。
- **验证**：`nvidia-smi -q | grep -i persistence` 应输出 `Enabled`。

### 2. 确认计算模式为默认共享

bash

```
sudo nvidia-smi -c 0
```



- `-c 0`：允许多进程共享 GPU（生产环境默认）。
- `-c 1`：独占模式，仅用于单任务性能压测。

### 3. 查看 GPU 互联拓扑（多卡训练必查）

bash

```
nvidia-smi topo -m
```



- **目的**：确认 NVLink 是否正常启用。若预期为 `NV12` 却显示 `PHB`，说明桥接器松动或 BIOS 配置有误，需立即处理。

### 4. MIG 模式管理（多租户隔离场景）

A100 支持将一张物理卡切分为最多 7 个独立逻辑 GPU 实例。

bash

```
# 查看当前 MIG 状态
nvidia-smi mig -lgi

# 启用 MIG 模式（需重启）
sudo nvidia-smi -mig 1 -i 0

# 创建 MIG 实例（示例：切分为 3 个 20GB 实例）
sudo nvidia-smi mig -cgi 14,14,14 -i 0
```



> **注意**：开启 MIG 后，`nvidia-smi` 表格中 `MIG M.` 列将显示 `Enabled`，且 GPU 实例将以 `GI` 编号单独列出。

### 5. 开启 ECC 校验（若尚未开启）

bash

```
# 查看 ECC 状态
nvidia-smi -q -i 0 | grep -i ecc

# 若为 Disabled，可通过 nvidia-smi 开启（需重启）
sudo nvidia-smi -e 1 -i 0
```



- **生产环境强烈建议开启 ECC**，防止静默数据损坏导致训练梯度异常。

------

## 📌 第七部分：常见问题与排查

| 现象                                     | 可能原因                   | 解决方案                                                     |
| :--------------------------------------- | :------------------------- | :----------------------------------------------------------- |
| `nvcc: command not found`                | 环境变量未生效             | `source ~/.bashrc` 或重新登录                                |
| `torch.cuda.is_available()` 返回 `False` | PyTorch 与 CUDA 版本不兼容 | 从 PyTorch 官网获取匹配命令                                  |
| 多卡训练速度远低于预期                   | NVLink 未启用或 PCIe 降速  | `nvidia-smi topo -m` 检查拓扑；`lspci -vvv | grep LnkSta` 检查带宽 |
| 新 A100 无法切分 MIG                     | MIG 模式未开启             | `sudo nvidia-smi -mig 1 -i ` 并重启                          |
| `nvidia-smi` 显示 `ERR!`                 | 显存 ECC 错误（Xid 48）    | 立即 `drain` 隔离并申请 RMA 更换                             |

------

## 📌 第八部分：完整安装流程速查表

| 步骤        | 命令                                               | 说明                             |
| :---------- | :------------------------------------------------- | :------------------------------- |
| 1. 下载     | `wget `                                            | 从官网获取 `.run` 文件           |
| 2. 赋权     | `chmod +x cuda_*.run`                              | 添加执行权限                     |
| 3. 安装     | `sudo sh cuda_*.run`                               | **取消 Driver 勾选**             |
| 4. 环境变量 | `vi ~/.bashrc`                                     | 添加 `PATH` 与 `LD_LIBRARY_PATH` |
| 5. 软链接   | `sudo ln -sf /usr/local/cuda-12.4 /usr/local/cuda` | 方便版本切换                     |
| 6. 生效     | `source ~/.bashrc`                                 | 立即生效                         |
| 7. 验证     | `nvcc --version`                                   | 确认编译器可用                   |
| 8. 持久模式 | `sudo nvidia-smi -pm 1`                            | 开启驱动常驻                     |
| 9. 拓扑检查 | `nvidia-smi topo -m`                               | 确认 NVLink 正常                 |

------

> **笔记总结**：
> 本指南针对 NVIDIA A100 数据中心 GPU 的生产环境需求，提供了从 CUDA 工具包安装到 MIG 配置、NVLink 验证、ECC 开启的完整流程。遵循此文档操作，可确保 A100 集群的 CUDA 运行时环境稳定、高效且易于维护。