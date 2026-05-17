# 在 ARM64 银河麒麟上构建 DeepStudent

适用于 **ARM64（aarch64，飞腾 / 鲲鹏）银河麒麟 V10 桌面版**。DeepStudent 是 Tauri 应用，Linux 包依赖 WebKitGTK 等原生库，**无法跨平台交叉编译**，必须在麒麟机器上原生构建。

## 前提

- ARM64 银河麒麟 V10 桌面版（`uname -m` 应为 `aarch64`）
- 建议 **8GB+ 内存**（主程序 crate 优化编译很吃内存）、**~20GB 空闲磁盘**、可联网
- 先确认环境：

  ```bash
  uname -m
  cat /etc/os-release
  cat /etc/kylin-version 2>/dev/null
  ```

## 1. 系统依赖（apt）

```bash
sudo apt update
sudo apt install -y build-essential curl wget file git pkg-config \
  libssl-dev libgtk-3-dev libwebkit2gtk-4.1-dev \
  libayatana-appindicator3-dev librsvg2-dev libxdo-dev \
  protobuf-compiler libprotobuf-dev patchelf
```

> ⚠️ **最大风险点：`libwebkit2gtk-4.1-dev`**
> Tauri 2 强制要求 webkit2gtk **4.1**。
> - 先查可用性：`apt-cache policy libwebkit2gtk-4.1-dev`
> - 麒麟 V10 **SP1**（Ubuntu 20.04 基底）通常只有 **4.0** → Tauri 2 无法构建。
> - 较新的麒麟（SP3 / 2403+）基底较新，可能带 4.1。
> - 若只有 4.0：需升级麒麟版本。从源码编译 webkit2gtk-4.1 极其麻烦，不推荐。
> - **这一步能否装上，决定整件事是否可行——请先确认。**

## 2. Rust 工具链（aarch64 原生）

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source "$HOME/.cargo/env"
rustc --version   # 应显示 aarch64
```

## 3. Node.js 20（麒麟自带版本可能过旧）

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc
nvm install 20
node -v
```

## 4. 获取代码

```bash
git clone https://github.com/superfm831010/deep-student.git
cd deep-student
```

## 5. 下载 ARM64 pdfium 动态库

`scripts/download-pdfium.sh` 已支持 `linux-arm64` 并会自动识别 aarch64：

```bash
bash scripts/download-pdfium.sh
# 产物：src-tauri/resources/pdfium/libpdfium.so
```

## 6. 构建

```bash
npm install
npm run tauri build -- --bundles deb,appimage
```

在 ARM 机器上 host target 自动为 `aarch64-unknown-linux-gnu`，无需 `--target`。
（也可用 `bash scripts/build_linux_all.sh`，但它会同时尝试 x86_64 目标——在 ARM 机器上那个会失败，可忽略。）

## 7. 产物与安装

```
src-tauri/target/release/bundle/deb/*.deb
src-tauri/target/release/bundle/appimage/*.AppImage
```

```bash
sudo dpkg -i src-tauri/target/release/bundle/deb/*.deb
sudo apt -f install    # 补齐缺失的运行时依赖
```

## 注意事项

- 首次编译约 1288 个 crate，ARM 机器上约需 **1–2 小时**（视 CPU 而定）。
- `lance` 向量库的构建脚本需要 `protoc`（步骤 1 的 `protobuf-compiler` 已包含）。若仍报找不到：`export PROTOC=$(which protoc)` 后重试。
- 构建末尾若报 `TAURI_SIGNING_PRIVATE_KEY` 错误，可忽略——那是应用内自动更新的签名步骤，安装包此时已生成。
- 内存不足会导致主程序 crate 编译时 OOM；可临时加 swap 缓解。
- webkit2gtk 4.1 的可用性是最大不确定因素，未在真实麒麟环境验证。
