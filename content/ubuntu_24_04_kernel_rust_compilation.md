+++
title= "在Ubuntu 24.04 LTS上编译支持Rust的Linux内核"
description= "详细指南：如何在Ubuntu 24.04 LTS上安装Rust工具链并编译支持Rust的Linux内核"
date= 2025-07-13
draft = false
[taxonomies]
categories = ["Rust For Linux"]
tags = ["rust-for-linux", "内核开发", "Ubuntu 24.04 LTS"]
+++

本文档详细介绍了如何在Ubuntu 24.04 LTS上安装Rust工具链并编译支持Rust的Linux内核。

<!--more-->

## 作者环境

### 主机环境（Mac）
- 芯片：Apple M4 Pro（ARM 架构，Apple Silicon）
- 内存：24 GB
- 启动磁盘：Macintosh HD
- 操作系统：macOS Sequoia 15.5

### 虚拟机环境

#### 虚拟化平台
- 平台：UTM
- 官方下载地址：[https://mac.getutm.app/](https://mac.getutm.app/)

#### 客户机系统
- 系统：Ubuntu 24.04 LTS（ARM64 架构）
- 官方镜像下载：[https://cdimage.ubuntu.com/releases/24.04/release/](https://cdimage.ubuntu.com/releases/24.04/release/)

## 下载Linux内核源码

### 1. 通过GitHub克隆Rust-For-Linux源码

你可以直接从GitHub获取Rust for Linux的最新开发源码：

```bash

git clone https://github.com/Rust-for-Linux/linux.git
cd linux
```

> 官方仓库地址：[https://github.com/Rust-for-Linux/linux](https://github.com/Rust-for-Linux/linux)

## 工具链说明

本次实验所用工具链为 kernel.org 官方提供的 matching LLVM + Rust 预编译包，适用于 ARM64 架构，具体版本为：

- llvm-18.1.8-rust-1.81.0-aarch64.tar.gz

下载地址：[https://kernel.org/pub/tools/llvm/rust/](https://kernel.org/pub/tools/llvm/rust/)

下载并解压：

```bash

wget https://mirrors.edge.kernel.org/pub/tools/llvm/rust/files/llvm-18.1.8-rust-1.81.0-aarch64.tar.gz
mkdir -p $HOME/llvm-rust
tar -xf llvm-18.1.8-rust-1.81.0-aarch64.tar.gz -C $HOME/llvm-rust --strip-components=1
cat << 'EOF' >> ~/.bashrc
# Rust for Linux toolchain
llvm_prefix=$HOME/llvm-rust
export PATH=$llvm_prefix/bin:$PATH
export LIBCLANG_PATH=$llvm_prefix/lib/libclang.so
export RUST_LIB_SRC=$llvm_prefix/lib/rustlib/src/rust/library
EOF
source ~/.bashrc
```

进入源码根目录，安装 bindgen：

```bash

sudo apt update
sudo apt install build-essential libncurses-dev libssl-dev bc flex bison libelf-dev
cargo install --locked --root $llvm_prefix --version $(scripts/min-tool-version.sh bindgen) bindgen-cli
```

验证工具链版本：

```bash

clang --version
cargo --version
rustc --version
bindgen --version
echo $RUST_LIB_SRC
```

示例输出：

```bash

ClangBuiltLinux clang version 18.1.8 (https://github.com/llvm/llvm-project.git 3b5b5c1ec4a3095ab096dd780e84d7ab81f3d7ff)
Target: aarch64-unknown-linux-gnu
Thread model: posix
InstalledDir: /home/ben/llvm-rust/bin
cargo 1.81.0 (2dbb1af80 2024-08-20)
rustc 1.81.0 (eeb90cda1 2024-09-04)
bindgen 0.65.1
/home/ben/llvm-rust/lib/rustlib/src/rust/library
```

> 详细说明和最新下载包请参考：[https://kernel.org/pub/tools/llvm/rust/](https://kernel.org/pub/tools/llvm/rust/)

## 检查 Rust 工具链可用性

在 Rust-For-Linux 源码根目录下，执行：

```bash

make LLVM=1 rustavailable
```

该命令会触发 Kconfig 的检测逻辑，判断当前环境下 Rust 工具链是否满足内核编译需求，并输出详细的检测结果。

示例输出：

```
Rust is available!
```

如果有报错，根据具体报错信息排查及解决。

## 生成内核配置文件

在确认 Rust 工具链可用后，接下来需要生成内核配置文件。执行以下命令：

```bash

make LLVM=1 defconfig
```

该命令会：
- 使用 LLVM 工具链（而不是 GCC）
- 生成默认的内核配置文件 `.config`
- 启用基本的 Rust 支持选项

执行完成后，会在源码根目录生成 `.config` 文件，这是内核编译的配置文件。

## 配置内核选项

生成默认配置文件后，你可能需要根据具体需求调整内核选项。使用以下命令启动交互式配置界面：

```bash

make LLVM=1 menuconfig
```
### 重要的 Rust 相关配置

在配置界面中，确保以下 Rust 相关选项已启用：

1. **Rust support** (`CONFIG_RUST=y`)
   - 位置：`General setup` → `Rust support`
   - 这是启用 Rust 支持的核心选项

2. **Rust samples** (`CONFIG_SAMPLES_RUST=y`)
   - 位置：`Kernel hacking` → `Sample kernel code` → `Rust samples`
   - 用于编译 Rust 示例代码

### 保存配置

完成配置后：
- 按 `S` 保存配置到 `.config` 文件
- 按 `Q` 退出配置界面

## 编译内核

配置完成后，开始编译内核。使用以下命令：

```bash

make LLVM=1 -j$(nproc)
```

### 命令参数说明

- `LLVM=1`：使用 LLVM 工具链进行编译
- `-j$(nproc)`：使用所有可用的 CPU 核心进行并行编译，`$(nproc)` 会自动检测 CPU 核心数

### 编译过程

编译过程可能需要较长时间（通常 30 分钟到 2 小时，取决于硬件配置），期间会显示：

1. **配置检查**：验证工具链和配置
2. **Rust 代码编译**：编译 Rust 编写的内核代码
3. **C 代码编译**：编译传统的 C 语言内核代码
4. **链接阶段**：将所有目标文件链接成最终的内核镜像

### 编译输出

编译成功后，会在以下位置生成关键文件：

- **内核镜像**：`arch/arm64/boot/Image`（ARM64 架构）
- **内核模块**：`*.ko` 文件在相应目录中
- **编译日志**：详细的编译信息会显示在终端

### 常见问题处理

如果编译过程中出现错误：

1. **内存不足**：减少并行编译数量，使用 `-j2` 或 `-j4`
2. **依赖缺失**：根据错误信息安装缺失的开发包
3. **配置冲突**：重新运行 `make LLVM=1 menuconfig` 调整配置

> **提示**：编译过程中请保持网络连接，因为可能需要下载额外的依赖包。

## 验证编译结果

编译完成后，验证内核镜像是否成功生成：

```bash

ls -alF arch/arm64/boot/Image
```

### 预期输出

如果编译成功，应该看到类似以下输出：

```bash

-rwxrwxr-x 1 ben ben 40417792 Jul 13 13:09 arch/arm64/boot/Image*
```

### 其他重要文件

编译成功后，还可以检查以下文件：

```bash

# 检查编译日志
ls -la .config

# 检查内核版本信息
cat include/config/kernel.release
```

> **成功标志**：如果能看到 `arch/arm64/boot/Image` 文件且大小正常（通常 30-50 MB），说明内核编译成功。
