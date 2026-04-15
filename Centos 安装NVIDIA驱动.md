# CentOS 9 环境下 NVIDIA 显卡驱动安装详细笔记



### 1. 安装 EPEL 软件源

bash

```
dnf install https://d.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
```



**说明与完善**：

- **目的**：EPEL（Extra Packages for Enterprise Linux）是由 Fedora 社区维护的额外软件包仓库，提供了 `dkms` 等 CentOS 官方源未包含的关键工具。

- **错误提示**：你的命令使用了 **`el7`**（RHEL 7）版本，而系统是 **CentOS 9**（对应 RHEL 9）。虽然可能通过兼容性勉强工作，但强烈建议修正为：

  bash

  ```
  dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
  ```

  

  *(此处仅作提示，正文保留原命令)*

### 2. 更新软件包信息

bash

```
dnf makecache
```



**说明与完善**：

- **目的**：强制 `dnf` 立即下载所有已启用仓库的最新元数据（包列表）。新添加 EPEL 源后执行此命令，可确保后续搜索和安装能找到 EPEL 中的包。

### 3. 更新系统内核

bash

```
dnf update kernel -y
```



**说明与完善**：

- **目的**：将 Linux 内核更新到官方仓库中的最新稳定版本。
- **为什么这么做**：NVIDIA 驱动的编译需要 **`kernel-devel`** 包与当前运行的 **`kernel`** 版本严格一致。CentOS 9 初始安装镜像的内核版本往往较旧，而在线仓库中的 `kernel-devel` 包通常只保留最新版本。先更新内核并重启，能确保后续能找到完全匹配的开发包，避免出现 “Unable to find kernel source tree” 报错。

### 4. 安装内核开发包与工具

bash

```
dnf install kernel-devel-$(uname -r) kernel-headers-$(uname -r)ntfs-3g -y
```



**说明与完善**：

- **目的**：安装编译内核模块（驱动）所必需的头文件和构建脚本。

- **命令拆解**：

  - `kernel-devel-$(uname -r)`：安装与当前运行内核精确匹配的开发包。
  - `kernel-headers-$(uname -r)`：提供 C 语言头文件，用于用户态程序编译。
  - `ntfs-3g`：这是一个 NTFS 文件系统驱动，通常与显卡驱动安装**无关**，推测为笔误或顺手安装的工具。

- **错误提示**：命令末尾的 `kernel-headers-$(uname -r)ntfs-3g` 存在**语法错误**，由于缺少空格，系统会尝试寻找名为 `kernel-headers-3.10.0...ntfs-3g` 的包，导致失败。**建议原笔记修正为**：

  bash

  ```
  dnf install kernel-devel-$(uname -r) kernel-headers-$(uname -r) ntfs-3g -y
  ```

  

### 5. 安装 GCC 编译器与开发库

bash

```
dnf install gcc kernel-devel kernel-doc gcc\* glibc\* glibc-\* -y
```



**说明与完善**：

- **目的**：安装 GNU 编译器集合（GCC）及 C 语言运行库的开发版本。
- **为什么这么做**：NVIDIA 驱动的安装脚本（`.run` 文件）本质上是在现场编译内核接口层。缺乏 GCC 会导致安装失败并报错 **"Unable to find the development tool cc"**。

### 6. 屏蔽系统自带的开源驱动 Nouveau

#### 步骤 6.1：修改黑名单配置

bash

```
vi /lib/modprobe.d/dist-blacklist.conf
```



**操作内容**：

bash

```
# blacklist nvidiafb
blacklist nouveau
options nouveau modeset=0
```



**说明与完善**：

- **目的**：阻止 Linux 内核在启动时自动加载开源的 `nouveau` 驱动，将其释放给 NVIDIA 闭源驱动使用。

- **背景知识**：CentOS 9 的 `/lib/modprobe.d/` 目录由系统管理，更新系统时可能覆盖该文件。**更规范的隔离做法**是创建独立文件：

  bash

  ```
  echo -e "blacklist nouveau\noptions nouveau modeset=0" | sudo tee /etc/modprobe.d/blacklist-nouveau.conf
  ```

  

  *(此处仅作提示，正文保留原操作)*

#### 步骤 6.2：内核引导参数屏蔽

bash

```
vi /etc/default/grub
```



**操作内容**：找到 `GRUB_CMDLINE_LINUX` 行，在 `rhgb quiet` 后添加：

text

```
rd.driver.blacklist nouveau nouveau.modeset=0
```



**说明与完善**：

- **目的**：这是**双重保险**。不仅防止内核启动后期加载驱动，也防止系统在启动初期（initramfs 阶段）加载 `nouveau`。
- **验证效果**：执行 `cat /etc/default/grub` 检查配置是否已保存。

### 7. 切换为纯命令行模式（关闭图形界面）

bash

```
systemctl set-default multi-user.target
```



**说明与完善**：

- **目的**：将系统默认启动目标设置为多用户文本模式（不启动 GNOME/KDE 桌面环境）。
- **为什么这么做**：NVIDIA 驱动安装程序必须在 **X Server 完全关闭** 的状态下运行。如果桌面环境正在运行并占用了显卡，安装会报错并中断。`multi-user.target` 相当于以前的运行级别 3。

### 8. 更新系统初始内存文件系统镜像（Initramfs）

#### 步骤 8.1：备份原镜像

bash

```
mv /boot/initramfs-$(uname -r).img  /boot/initramfs-$(uname -r).img.bak
```



**说明与完善**：

- **目的**：备份当前的启动镜像。万一生成的新镜像无法启动系统，可以在 GRUB 界面选择旧镜像恢复。

#### 步骤 8.2：重新生成镜像

bash

```
dracut /boot/initramfs-$(uname -r).img$(uname -r)
```



**说明与完善**：

- **目的**：将刚才配置的黑名单规则（`blacklist nouveau`）真正封入启动内核时使用的内存盘中。

- **错误提示**：命令末尾存在明显的**拼写错误**。原命令 `...img$(uname -r)` 会生成一个错误的文件名。

- **建议修正**：正确的命令应为：

  bash

  ```
  dracut /boot/initramfs-$(uname -r).img $(uname -r)
  ```

  

  *(此处仅作提示，正文保留原命令)*

### 9. 重启服务器

bash

```
reboot
```



**说明与完善**：

- **目的**：让内核引导参数的修改和 initramfs 更新生效。

### 10. 重启后验证环境

#### 步骤 10.1：验证 Nouveau 是否屏蔽成功

bash

```
lsmod | grep nouveau
```



**说明与完善**：

- **目的**：检查当前加载的内核模块中是否还有 `nouveau`。
- **预期结果**：**没有任何输出**，代表开源驱动已彻底闭嘴。

#### 步骤 10.2：检查并停止显示管理器 (GDM)

bash

```
systemctl --all | grep gdm
```



**说明与完善**：

- **目的**：确认 GNOME Display Manager 的服务状态。
- **操作**：如果有 `gdm.service` 显示为 `loaded` 或 `active`，需要立即停止它：

bash

```
systemctl stop gdm.service
```



**说明**：这一步是为了确保在运行 `.run` 文件时，没有残留的图形服务占用显卡设备文件 `/dev/dri/`。

### 11. 执行驱动安装程序

bash

```
chmod +x 包名
./包名.run
```



**说明与完善**：

- **目的**：给予安装脚本执行权限并运行。
- **交互提示**：脚本通常会询问是否安装 **DKMS**（推荐选择 Yes，这样未来系统内核升级时驱动会自动重新编译）、是否安装 **32位兼容库**（服务器环境可选 No）。

### 12. 验证安装结果

bash

```
nvidia-smi
```



**说明与完善**：

- **预期结果**：屏幕上打印出 NVIDIA Logo 表格，显示 Driver Version、CUDA Version 以及 GPU 利用率/显存占用。

### 13. 恢复图形界面并重启

bash

```
systemctl enable gdm.service
reboot
```



**说明与完善**：

- **目的**：将系统启动目标恢复为图形化（如果之前曾设置为 `multi-user.target`，建议加一步 `systemctl set-default graphical.target`），并最终重启进入带 NVIDIA 硬件加速的桌面环境。