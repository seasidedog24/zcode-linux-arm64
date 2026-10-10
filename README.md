# ZCode Linux arm64 (aarch64) Builds

> **English summary:** This repository provides fully reproducible Linux arm64 (aarch64) builds of the **vanilla, unmodified** [ZCode](https://github.com/zai-org/ZCode) open-source editor (v3.14.3), including all four desktop package formats (AppImage, .deb, .rpm, .pacman / pkg.tar.zst) plus the headless CLI tarball — a combination not offered by the official download site. It also ships a GitHub Actions workflow that rebuilds everything from source on the free `ubuntu-24.04-arm` runner, and a build guide covering China-network mirror workarounds and Ubuntu 24.04 pitfalls (SUID sandbox, AppImage extraction). This is an **unofficial community project**; all credit and copyright belong to Z.AI, released under Apache-2.0.

---

## 项目定位

- **vanilla 源码，零改动构建**：本仓库所有产物均直接从 [zai-org/ZCode](https://github.com/zai-org/ZCode) 官方开源源码 `v3.14.3` tag 构建，未修改任何业务代码（构建网络适配见下文「中国网络环境」节，均不影响产物行为）。
- **补齐 arm64 格式空缺**：官方下载页目前只提供 Linux x64 与 arm64 Beta 的部分格式；本仓库提供 **aarch64 全部四种桌面格式 + headless tarball** 共 5 个产物。
- **CI 可复现**：附带的 GitHub Actions workflow 在免费 `ubuntu-24.04-arm` runner 上从源码完整重建全部产物，任何人都可以 fork 后自行验证或构建其他版本。

## 产物清单

见 [Releases](https://github.com/seasidedog24/zcode-linux-arm64/releases)。

| 产物 | 格式 | 适用系统 |
| --- | --- | --- |
| `ZCode-3.14.3-arm64.AppImage` | AppImage | 大多数发行版（需 libfuse2，或解包运行） |
| `ZCode-3.14.3-arm64.deb` | deb | Debian / Ubuntu 24.04+ (arm64) |
| `ZCode-3.14.3-aarch64.rpm` | rpm | RHEL 8+ / Rocky / Fedora (aarch64) |
| `ZCode-3.14.3-aarch64.pkg.tar.zst` | pacman | Arch Linux ARM / Manjaro ARM |
| `zcode-headless-3.14.3-linux-arm64.tar.gz` | tarball | headless / 服务器场景（无 GUI 依赖） |

### SHA-256

<!-- SHA256-TABLE -->

| 文件 | SHA-256 |
| --- | --- |
| `ZCode-3.14.3-arm64.AppImage` | `49befe7528b26c8f5e8a414781e5976c2e892fcd4e2a0d96144f0a39f80b3136` |
| `ZCode-3.14.3-arm64.deb` | `6e13c3ec0a40d5ff0d918094674c5027ffb645ae499265157ef27caed29c225d` |
| `ZCode-3.14.3-aarch64.rpm` | `eda2b1124c930b9a24c18879c77f9eb5e14e71a59c0573de1c7091e49e982399` |
| `ZCode-3.14.3-aarch64.pkg.tar.zst` | `8310acac5b6240b0f7106c36f7362e4b10697189faacbb60cede9fd033093d7d` |
| `zcode-headless-3.14.3-linux-arm64.tar.gz` | `7146c8e5127edc9c980cfa29911b618ad127870219a3d33a28724174b17044f6` |

Release notes 中亦有构建环境说明。

## 构建步骤（手动）

构建机：Linux arm64（或 qumu/容器内 arm64），Ubuntu 24.04 实测通过。

系统依赖：

```bash
sudo apt-get update
sudo apt-get install -y rpm libarchive-tools git
# 仅运行 AppImage 需要：
sudo apt-get install -y libfuse2
```

- Node.js **24.14.0**
- pnpm **10.33.2**（源码 `package.json` 的 `packageManager` 字段锁定）

```bash
git clone --branch v3.14.3 https://github.com/zai-org/ZCode.git
cd ZCode
pnpm bootstrap
pnpm bundle:desktop -- --os linux --arch arm64
pnpm build:zcode --base-url https://example.com/zcode/deps/zcode/
```

产物位置：

- 桌面四种格式：`packages/desktop/dist/`
- headless tarball：`dist/zcode/releases/<version>/zcode-<version>.tar.gz`（同目录有 `sha256.txt`）

## CI 构建（推荐）

`.github/workflows/build-linux-arm64.yml`：

- 触发：推送 `v*` tag，或手动 `workflow_dispatch`
- Runner：GitHub 免费 `ubuntu-24.04-arm`
- 步骤：checkout v3.14.3 源码 → 装依赖 → `pnpm bootstrap` → `pnpm bundle:desktop -- --os linux --arch arm64` → `pnpm build:zcode` → 上传 artifact；tag 推送时自动 `gh release create` 挂全部产物

注意：`pnpm bootstrap` 内部会执行 `git submodule update`，因此**必须使用真实 git checkout**，官方 release tarball 会导致该步骤失败。

## 中国网络环境

在境内网络构建时，两处网络适配实测有效（均不影响产物本身，非必须）：

1. **Electron 二进制下载**：`packages/desktop/scripts/node-pty-rebuild.mjs` 中 electron-rebuild 需要下载 Electron 41.0.3 的 headers。npmmirror 镜像的 `SHASUMS256.txt` 缺 headers 条目，导致校验值拿到 `undefined` 而失败。两种解法（实测用后者）：
   - 给 electron-rebuild 加参数 `--dist-url https://npmmirror.com/mirrors/electron/`；
   - 或预先放置官方 headers 缓存 `~/.electron-gyp/41.0.3/`，跳过下载。
2. **npm registry**：项目根 `.npmrc` 的 registry 换成 `https://registry.npmmirror.com`，仅为绕开境外限速。

GitHub Actions 的 runner 网络良好，CI 中不需要以上任何适配。

## Ubuntu 24.04 已知坑

### SUID sandbox

Ubuntu 24.04 对非特权 user namespace 有 AppArmor 限制，Electron 的 chrome-sandbox 如果没有 SUID 位会启动失败，报错类似：

```
The SUID sandbox helper binary was found, but is not configured correctly.
```

两种解法：

```bash
# 解法一：给 chrome-sandbox 加 SUID（需要 root，deb/rpm/pacman 安装后路径）
sudo chown root:root /opt/ZCode/chrome-sandbox
sudo chmod 4755 /opt/ZCode/chrome-sandbox

# 解法二：临时关闭 sandbox 启动（仅自用/受控环境）
zcode --no-sandbox
```

### AppImage 运行

AppImage 依赖 FUSE：

```bash
sudo apt-get install -y libfuse2
./ZCode-3.14.3-arm64.AppImage
```

没有 FUSE（容器 / 最小系统）时解包运行：

```bash
./ZCode-3.14.3-arm64.AppImage --appimage-extract
./squashfs-root/AppRun
```

## 免责声明

- 本仓库是**非官方社区项目**，与 Z.ai 无隶属关系。
- ZCode 的全部版权归 **Z.AI Co., Ltd** 所有，源码与产物遵循 **Apache-2.0** 许可证（见 [LICENSE](LICENSE)）。
- 产物从官方 v3.14.3 源码零改动构建，但不提供任何担保（as-is）；生产使用请自行验证。
