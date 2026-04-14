# 移动硬盘安装 Rust 环境指南

> **文档定位**：环境配置专题
> 
> **目标读者**：系统磁盘空间紧张，希望将 Rust 工具链和依赖包缓存完全转移到外接移动硬盘的开发者。
> 
> **适用系统**：macOS / Linux（Windows 原理相同，但路径和环境变量配置方式略有差异）

---

## 1. 背景与痛点

Rust 是一门强类型的编译型语言，其工具链（`rustup`、`rustc`、`cargo`）和依赖包缓存（crates.io registry）随着开发时间的推移，会占用极大的磁盘空间。

默认情况下，Rust 会在你的用户目录下创建两个隐藏文件夹：
- `~/.rustup`：存放工具链（编译器、标准库等），通常占用 1-3 GB。
- `~/.cargo`：存放全局可执行文件（如 `cargo-fmt`）和下载的第三方依赖包缓存，随着项目增多，可能膨胀到 10 GB 以上。

如果你的 Mac/PC 只有 256GB 的内置存储，将这两个目录迁移到移动硬盘是释放空间的最佳选择。

---

## 2. 核心原理：RUSTUP_HOME 与 CARGO_HOME

Rust 的安装器和包管理器允许通过环境变量重写默认路径：

1. **`RUSTUP_HOME`**：定义工具链的安装位置（替代 `~/.rustup`）。
2. **`CARGO_HOME`**：定义包管理器、全局可执行文件和缓存的位置（替代 `~/.cargo`）。

只要我们在安装和日常使用前，确保这两个环境变量指向移动硬盘，Rust 就会完全在移动硬盘上运行，不会占用内置磁盘空间。

---

## 3. 操作步骤（以 macOS/Linux 为例）

### 步骤 1：准备移动硬盘目录

假设你的移动硬盘挂载点为 `/Volumes/MySSD`。我们先在上面创建 Rust 的专属目录：

```bash
# 1. 进入移动硬盘
cd /Volumes/MySSD

# 2. 创建 Rust 专属目录（名字随意，这里以 RustEnv 为例）
mkdir -p RustEnv/.rustup
mkdir -p RustEnv/.cargo
```

> **注意**：建议移动硬盘使用 APFS (macOS) 或 ext4 (Linux) 等支持软链接和权限管理的文件系统。如果是 exFAT，虽然可以用，但在某些极端权限场景下可能会遇到问题。

### 步骤 2：配置环境变量

为了让安装器和未来的终端会话都能识别新路径，需要将环境变量写入你的 Shell 配置文件（通常是 `~/.zshrc` 或 `~/.bashrc`）。

打开终端，执行以下命令将配置追加到你的 rc 文件中：

```bash
# 假设你使用的是 zsh（macOS 默认）
cat << 'EOF' >> ~/.zshrc

# ======= Rust on External Drive =======
# 移动硬盘的挂载路径，请根据实际情况修改
export EXTERNAL_DRIVE="/Volumes/MySSD/RustEnv"

# 重定向 rustup 和 cargo 的默认目录
export RUSTUP_HOME="$EXTERNAL_DRIVE/.rustup"
export CARGO_HOME="$EXTERNAL_DRIVE/.cargo"

# 将 cargo 的 bin 目录加入系统 PATH，以便全局命令可直接执行
export PATH="$CARGO_HOME/bin:$PATH"
# ======================================
EOF
```

保存后，**必须重载配置**使当前终端生效：

```bash
source ~/.zshrc
```

可以通过 `echo $CARGO_HOME` 验证是否输出 `/Volumes/MySSD/RustEnv/.cargo`。

### 步骤 3：执行 Rust 安装脚本

在环境变量生效的终端中，运行官方的安装脚本：

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

在安装向导出现时，仔细观察输出的路径提示：

```text
Current installation options:

   default host triple: aarch64-apple-darwin
     default toolchain: stable (default)
               profile: default
  modify PATH variable: yes

# 这里应该显示你移动硬盘的路径，而不是 /Users/xxx/...
```

如果路径正确，直接按 `1` (Proceed with standard installation) 继续安装。

安装过程中的所有下载和解压都会直接写入移动硬盘。

### 步骤 4：验证安装

安装完成后，执行：

```bash
rustc --version
cargo --version
```

验证缓存路径：

```bash
# 查看 rustup 工具链的真实物理路径
rustup show active-toolchain
```
输出应该类似于 `/Volumes/MySSD/RustEnv/.rustup/toolchains/stable-aarch64-apple-darwin`。

---

## 4. 迁移已有项目和 Target 目录（进阶）

仅仅把 `~/.cargo` 移走是不够的。Rust 编译每个项目时，会在项目目录下生成一个 `target/` 文件夹，存放编译中间产物。对于中大型项目，一个 `target/` 文件夹轻易就能达到几 GB。

如果你的代码克隆在内置硬盘，编译产物依然会吃掉内置硬盘空间。

**解决方案：重定向 `CARGO_TARGET_DIR`**

你可以强制 Cargo 把所有项目的编译产物统一放到移动硬盘上的一个特定目录。

在移动硬盘上创建一个编译缓存目录：

```bash
mkdir -p /Volumes/MySSD/RustEnv/cargo-target
```

然后再次修改你的 `~/.zshrc`，增加一行：

```bash
export CARGO_TARGET_DIR="/Volumes/MySSD/RustEnv/cargo-target"
```
重载 `~/.zshrc`。

**效果**：
你在内置硬盘任何地方执行 `cargo build`，生成的二进制和中间缓存都会被扔到移动硬盘的 `cargo-target` 目录里。你的项目目录将永远保持干净，只有源码。

---

## 5. 日常使用的注意事项

1. **先插硬盘，再开终端**
   如果你没插硬盘就打开了终端，虽然变量加载了，但路径不存在，执行 `cargo` 会报错。
   如果没插硬盘时执行了 `rustup`，它可能会尝试在你内置硬盘的当前路径创建文件夹（如果权限允许的话），所以请养成“插盘 -> 开发”的习惯。

2. **如何清理移动硬盘上的空间？**
   即使在移动硬盘上，空间也不是无限的。你可以使用 `cargo-cache` 工具来清理。
   ```bash
   cargo install cargo-cache
   cargo cache -a  # 清理所有无用缓存
   ```

3. **如果我想临时在没有移动硬盘时开发怎么办？**
   由于你把环境变量写死在了 `~/.zshrc`，拔掉硬盘后 Rust 就无法使用了。
   如果你有这种混合需求，建议不要把 `export` 写死在全局，而是写一个脚本 `enable_rust.sh` 放在本地，只有插上硬盘需要开发时，执行 `source enable_rust.sh` 临时激活环境变量。

---

## 6. 从内置硬盘迁移到移动硬盘（已有环境）

如果你之前已经安装在了内置硬盘（`~/.rustup` 和 `~/.cargo` 已经存在），想迁移到移动硬盘，不需要卸载重装。

1. **移动文件**：
   ```bash
   mv ~/.rustup /Volumes/MySSD/RustEnv/
   mv ~/.cargo /Volumes/MySSD/RustEnv/
   ```

2. **配置环境变量**（同步骤 2）：
   在 `~/.zshrc` 中设置 `RUSTUP_HOME` 和 `CARGO_HOME`，并更新 `PATH`。

3. **重载配置并验证**：
   ```bash
   source ~/.zshrc
   cargo --version
   ```
   迁移即告完成，内置硬盘的空间已被释放。
