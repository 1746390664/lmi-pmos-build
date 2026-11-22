# 用 GitHub Actions 云端编译 lmi 的 postmarketOS 镜像

本机不需要 Linux、不需要 WSL、不用装 40GB 的编译环境。
GitHub 免费跑一台 Ubuntu 24.04 机器，约 60–100 分钟出镜像，直接下载。

---

## 一、建仓库（1 分钟）

推荐 **public** 仓库：公开仓库的 Actions 免费且不限时长；私有仓库会扣每月 2000 分钟额度，
一次编译约 90 分钟。

打开 https://github.com/new ，填：

- Repository name：`lmi-pmos-build`
- 选择 **Public**
- 不要勾选 README / .gitignore / license
- Create repository

## 二、把本目录推上去

在本目录（`D:\刷机工具包\github-build`）打开 Git Bash：

```bash
cd /d/刷机工具包/github-build
git init -b main
git add .
git commit -m "build lmi postmarketOS image"
git remote add origin https://github.com/<你的用户名>/lmi-pmos-build.git
git push -u origin main
```

推送时会让你输入 GitHub 用户名和 **Personal Access Token**（不是密码）：
GitHub → 头像 → Settings → Developer settings → Personal access tokens →
Tokens (classic) → Generate new token，勾 `repo` 即可。

> 如果 push 报 `CRYPT_E_NO_REVOCATION_CHECK` 或证书错误，是本机代理的证书链不被 git 信任，
> 用这一次性的免校验推送即可（仅影响这次连接）：
>
> ```bash
> git -c http.sslVerify=false push -u origin main
> ```

> 不想用命令行：在仓库页面点 **Add file → Create new file**，
> 文件名填 `.github/workflows/build-lmi.yml`，把本目录同名文件的内容整段粘进去，Commit 即可。

## 三、开始编译

工作流配置了「push 到 main 自动跑」，所以推上去就已经开始了。
想手动再跑一次：仓库页面 → **Actions** → 左侧 `Build lmi postmarketOS image` →
右侧 **Run workflow** → 选桌面（`phosh` / `plasma-mobile`）→ Run。

如果 Actions 提示未启用：**Settings → Actions → General → Allow all actions** 保存。

## 四、下载产物

运行结束后（绿勾）：

**Actions → 点进那次运行 → 页面底部 Artifacts → `xiaomi-lmi-pmos`** 下载。

压缩包里三样：

| 文件 | 用途 |
|---|---|
| `boot.img` | 刷入 boot 分区 |
| `xiaomi-lmi.img.xz` | 刷入 userdata 分区（需先解压） |
| `SHA256SUMS.txt` | 校验用 |

解压 `.xz`（Windows 用 7-Zip：右键 → 7-Zip → 提取到当前文件夹），
把 `boot.img`、`xiaomi-lmi.img` 放进 `D:\刷机工具包\lmi-pmos\`，然后：

```bat
cd /d D:\刷机工具包
flash-pmos.bat check
flash-pmos.bat flash
```

---

## 说明与排查

**为什么不用 `washyda/lmi-nightly-builds` 直接 fork？**
那个仓库的 workflow 走 `--envkernel` 路径，依赖 `washyda/linux` 的 `nikroks/alioth` 分支，
从 2026-02 起连续失败（内核分支已不在）。本工作流改成直接按 APKBUILD 拉
`yuweiyuan8/Linux-redmi-k30pro-6.19` 的源码包编译，并内置了仓库改名后 404 的自动回退。

**编译在跑什么**

1. 装 pmbootstrap，拉 `yuweiyuan8/pmaports` 的 `nikroks/alioth` 分支（含 lmi 设备包 / 固件包）
2. 交叉编译 `linux-postmarketos-qcom-sm8250-lmi` 6.19.7
3. 编译 `device-xiaomi-lmi`
4. `pmbootstrap install` 生成 rootfs（不加密，密码 `147147`）
5. `pmbootstrap export` → 压缩 → 上传

**常见失败**

| 现象 | 原因 / 处理 |
|---|---|
| `HTTP 404` 拉内核源码 | 工作流已自动回退到新仓库名；若仍失败，看日志里 `declared source` 那行，手动改 APKBUILD + `pmbootstrap checksum` |
| `No space left on device` | 工作目录已放在 `/mnt`（大盘）；若仍爆盘，把 `xz -T0 -1` 改成 `-T0 -9e` 前先删 `out/xiaomi-lmi.img` |
| 超过 350 分钟 | 极少见；把 `timeout-minutes` 提到 355（GitHub 硬上限 6 小时） |
| Artifact 下载很慢 | 镜像约 1–1.5GB，正常；下载后可先核对 `SHA256SUMS.txt` |
| 提示需要登录才能下载 Artifact | Artifact 只对仓库所有者/协作者开放，用自己的账号即可 |

**产物时效**：Artifacts 保留 30 天（`retention-days: 30`），要长期留存请下载到本地。

**改动 UI**：仓库里改 workflow 的 default，或手动 Run 时选 `plasma-mobile`。
`phosh` 更轻、触屏体验稳；`plasma-mobile` 功能多但更吃 GPU 固件。
