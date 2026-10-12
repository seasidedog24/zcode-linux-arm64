# ZCode for Linux on ARM64

[![CI](https://github.com/seasidedog24/zcode-linux-arm64/actions/workflows/build-linux-arm64.yml/badge.svg)](https://github.com/seasidedog24/zcode-linux-arm64/actions/workflows/build-linux-arm64.yml)
[![Release](https://img.shields.io/badge/release-v3.14.3--arm64-blue)](https://github.com/seasidedog24/zcode-linux-arm64/releases/tag/v3.14.3-arm64)
[![License](https://img.shields.io/badge/license-Apache--2.0-green)](LICENSE)

Unofficial, **fully reproducible** builds of [ZCode](https://github.com/zai-org/ZCode) — Z.ai's open-source agentic coding workbench — for **Linux aarch64**, produced from the **unmodified vanilla source** with a one-click GitHub Actions pipeline on free `ubuntu-24.04-arm` runners.

> ZCode (Apache-2.0, © Z.ai) officially distributes desktop installers for macOS and Windows on GitHub Releases. The public source supports `--os linux --arch arm64` end to end, but no official Linux artifacts are published on GitHub. This repo fills that gap for ARM64 Linux machines (NVIDIA DGX Spark, ARM servers, Raspberry Pi-class SBCs with enough RAM, etc.), and adds a **headless web distribution** that even the official Linux Beta does not ship.

[中文说明](#中文说明)

## Artifacts (v3.14.3)

| Artifact | Format | Size | sha256 (first 16) |
|---|---|---|---|
| `ZCode-3.14.3-arm64.AppImage` | AppImage (desktop GUI) | 182 MiB | `49befe7528b26c8f` |
| `ZCode-3.14.3-arm64.deb` | Debian/Ubuntu package | 128 MiB | `6e13c3ec0a40d5ff` |
| `ZCode-3.14.3-aarch64.rpm` | Fedora/RHEL package | 106 MiB | `eda2b1124c930b9a` |
| `ZCode-3.14.3-aarch64.pkg.tar.zst` | Arch Linux package | 107 MiB | `8310acac5b6240b0` |
| `zcode-headless-3.14.3-linux-arm64.tar.gz` | Headless CLI / Web runtime | 78 MiB | `7146c8e5127edc9c` |

Full checksums: [SHA256SUMS-verified.txt](SHA256SUMS-verified.txt) · Download: [Releases](https://github.com/seasidedog24/zcode-linux-arm64/releases)

## Verified on real hardware

Everything below was exercised end-to-end on an **NVIDIA DGX Spark (GB10 Grace-Blackwell, Ubuntu 24.04, 4K display)** — not just packaged:

- **Desktop GUI**: welcome/onboarding, Z.AI & BigModel account login, workspace management, remote-control QR pairing (colleagues join the same workspace from a phone/laptop via `zcode.z.ai/remote`), Feishu/Lark bot channel binding.
- **Headless web** (`zcode --web`): session create/resume with full persistence across restarts, streaming replies, image input via vision models, file and terminal tools driven through the WebSocket RPC protocol.
- **Parity testing**: an 8-item functional matrix against the official website's Linux Beta (v3.14.5) passed on all counts; the differences found were version lags, not defects (see "Findings" below).

## Quick start

### Desktop (AppImage, Ubuntu 24.04)

Ubuntu ≥ 23.10 restricts unprivileged user namespaces, which breaks Electron's SUID sandbox inside AppImage mounts. Extract and fix once:

```bash
chmod +x ZCode-3.14.3-arm64.AppImage
./ZCode-3.14.3-arm64.AppImage --appimage-extract
sudo chown root:root squashfs-root/chrome-sandbox
sudo chmod 4755 squashfs-root/chrome-sandbox
./squashfs-root/zcode
```

### Headless web (no GUI, remote access from any browser)

Requires Node.js 24 on `PATH` (matching upstream `mise.toml`).

```bash
tar -xzf zcode-headless-3.14.3-linux-arm64.tar.gz
node zcode/bin/zcode.mjs --web --workspace /path/to/project --host 0.0.0.0 --port 3031 --no-open
```

Listening on a non-loopback address auto-generates an access token; the console prints the ready-to-share URL.

## Reproduce the build

One click: **Actions → "Build Linux arm64" → Run workflow** (GitHub's free ARM64 runners, ~40 min), or push a `v*` tag to build and publish a release automatically. Or build locally from vanilla source:

```bash
git clone https://github.com/zai-org/ZCode && cd ZCode
pnpm bootstrap
pnpm bundle:desktop -- --os linux --arch arm64        # desktop artifacts
pnpm typecheck && pnpm build:zcode --base-url <url>   # headless runtime
```

System deps: Node 24.14.0, pnpm 10.33.2, `rpm`, `libarchive-tools`.

### Build findings (documented so you don't have to rediscover them)

1. `pnpm typecheck` must run before `build:zcode` — `packages/{shared,services,client}` have no build scripts; their `dist/` is emitted by the composite `tsc -b` pass that the TUI runtime closure collector requires.
2. On networks where `electronjs.org` is unreachable, pre-seed `~/.electron-gyp/<ver>/` with the headers tarball (extract so that `include/node/` sits directly under the version dir). Mirror `SHASUMS256.txt` files lack the headers entry, so `--dist-url` mirror overrides fail checksum validation.
3. `pnpm bootstrap` runs `git submodule update` — build from a real git checkout, not a source tarball.
4. AppImages need the SUID sandbox fix above on Ubuntu 24.04.

## How this compares

| | This repo | Official site (Linux Beta) | [ZCodium](https://github.com/ZCodium-project/ZCodium) | AUR `zcode` |
|---|---|---|---|---|
| Source | vanilla `zai-org/ZCode` v3.14.3 | official v3.14.5 (source not public) | fork with de-telemetry patches | repack of official deb |
| ARM64 desktop | ✅ all 4 package formats | ✅ 3 formats | ✅ AppImage | ✅ |
| Headless web runtime | ✅ | ❌ | ✅ (assets) | ❌ |
| Reproducible CI | ✅ GitHub Actions (arm64 runner) | ❌ | ✅ | ✅ (PKGBUILD) |

## Findings worth reporting upstream

- `readSessionMessages` / `readSessionEvents` RPC responses fail their own Zod schema validation in v3.14.3 (affects web-mode session readers).
- The official v3.14.5 Linux package's bundled CLI needs a `resources/glm/provider → resources/config/provider` symlink fix before it starts.
- The upstream repository (zai-org/ZCode) currently has PR creation closed for outside contributors (verified via both GraphQL and REST APIs); this repo exists partly because of that.

## Disclaimer

This is an unofficial community build, not affiliated with or endorsed by Z.ai. ZCode is Apache-2.0, © Z.ai; upstream third-party notices apply. No API keys, credentials, or session data are ever included in this repository or its releases.

---

## 中文说明

[ZCode](https://github.com/zai-org/ZCode) 是智谱 Z.ai 开源的 AI 编程工作台(Apache-2.0)。官方在 GitHub 只发布 macOS / Windows 安装包,本仓库用**未经改动的官方源码(v3.14.3)** 构建 Linux aarch64 全套产物,并通过 GitHub Actions 免费 arm64 runner 实现**一键可复现构建**。

- **产物**:AppImage / deb / rpm / pacman 四种桌面格式 + 无头 CLI/Web 运行包(官方 Linux Beta 没有的形态),全部附 sha256。
- **真机验证**:在 NVIDIA DGX Spark(GB10,Ubuntu 24.04)上完成 GUI 登录、二维码远控、飞书 Bot 绑定、无头 Web 会话持久化/流式/图像输入/文件与终端工具的端到端验证;与官方 Linux Beta 3.14.5 的 8 项功能对比全部通过。
- **技术要点**:`typecheck` 须在 `build:zcode` 前运行、受限网络下 electron headers 的缓存预装法、Ubuntu 24.04 的 SUID 沙箱修复等,详见英文部分 "Build findings"。
- **定位差异**:官方 Beta 源码不公开、ZCodium 是去遥测 fork、AUR 是重打包——本仓库是唯一的 vanilla 源码 + aarch64 全格式 + headless + 可复现 CI 组合。
- 本构建非官方产物,与 Z.ai 无关联;仓库与发布物中不含任何密钥或会话数据。
