# comfyui-cpu-guide
# ComfyUI 无显卡（CPU）部署完全指南

> 8GB 内存、无独立显卡的笔记本实测记录。本文不是官方文档的翻译，而是 2026 年 11 月在一台 i5-8250U / 8GB 内存 / 集成显卡的笔记本上，从零装到出第一张图的完整过程，包括所有踩过的坑。
>
> 适合人群：电脑里没有 NVIDIA 显卡、被网上"CUDA 教程"劝退过的人。

---

## 第 0 步：先判断你的机器该走哪条路

打开 PowerShell（按 `Win + X`，选「终端」），输入：

```powershell
Get-CimInstance Win32_VideoController | Select-Object Name, AdapterRAM
[math]::Round((Get-CimInstance Win32_ComputerSystem).TotalPhysicalMemory/1GB, 1)
```

对照结论：

| 你的情况 | 建议 |
|---|---|
| 有 NVIDIA 显卡（名字里有 RTX/GTX） | 走官方 CUDA 路线，本文不适用 |
| 只有 Intel/AMD 集成显卡，内存 ≥ 8GB | 本文就是为你写的 |
| 内存 < 8GB | 建议直接用云端（Google Colab 免费版），别在本机折腾 |

**重要认知**：CPU 模式下 ComfyUI 界面操作完全流畅（拖节点、改参数都不吃性能），只有最后的采样出图慢。512×512 分辨率下 SD1.5 大约 0.05-0.1 it/s，12 步约 2-4 分钟一张。把它当学习工具，别当生产力工具。

---

## 第 1 步：下载 ComfyUI 本体

国内直连 GitHub 大概率失败（连接被重置），用加速前缀：

```powershell
cd F:\
git clone https://ghfast.top/https://github.com/comfyanonymous/ComfyUI.git
```

实测 `ghfast.top` 可用；`ghproxy.net` 当时已失效。如果某个加速站挂了，搜"github 加速"换一个前缀即可，格式都是 `https://加速站/https://github.com/原始地址`。

也可以直接下载 zip 解压，效果一样（但以后更新不方便）。

---

## 第 2 步：Python 环境（新手最容易翻车的地方）

### 坑 1：`git` 命令找不到

如果 `git --version` 报"无法将 git 项识别为 cmdlet"，说明 git 不在系统 PATH。临时解决方案（只对当前窗口有效）：

```powershell
$env:Path += ";你的PortableGit路径\cmd"
```

长期方案：去 git-scm.com 下载 Git for Windows 正式版安装。

### 坑 2：`python` 指向的版本太老

很多老教程让装虚拟环境后直接敲 `python`，但你的系统 PATH 里可能是 years 前装的旧版本。**先确认版本**：

```powershell
python --version
```

ComfyUI 需要 **Python 3.10 - 3.13**，推荐 3.12。如果版本不对，用 `py -0p` 列出所有已安装版本，选一个 3.10+ 的：

```powershell
py -0p
py -3.12 -m venv venv
```

如果机器上没有任何新版本，去 python.org 装 3.12（安装时勾选 Add to PATH）。

### 坑 3：PowerShell 激活虚拟环境报错

PowerShell 默认禁止执行 `.ps1` 脚本，`.\venv\Scripts\Activate.ps1` 会报红字。**绕过激活**，直接用完整路径调用：

```powershell
.\venv\Scripts\python.exe -m pip install xxx
```

这样完全不需要激活，也不碰执行策略。

---

## 第 3 步：安装依赖（pip 源的选择有讲究）

### 实测各镜像源速度（同一个包）

| 源 | 实测速度 |
|---|---|
| 腾讯云 `mirrors.cloud.tencent.com/pypi` | **17.1 MB/s** |
| 清华 `pypi.tuna.tsinghua.edu.cn` | 8.3 MB/s（但缺部分新包） |
| 阿里云 `mirrors.aliyun.com/pypi` | 338 KB/s |
| 官方 pypi.org | 6.8 秒才打开索引页 |

清华源**不一定有** ComfyUI 依赖的最新版本（当时缺 `comfyui-workflow-templates-media-assets-02`），腾讯云和阿里云有。**结论：优先腾讯云。**

```powershell
cd F:\ComfyUI
.\venv\Scripts\python.exe -m pip install --progress-bar off -i https://mirrors.cloud.tencent.com/pypi/simple -r requirements.txt
```

注意：Windows 上从 PyPI 直接装的 torch 就是 CPU 版（约 250MB），**不要**跟着网上教程加 `--index-url .../whl/cu121`，那是给 N 卡的，下下来 2.5GB 还跑不起来。

### 坑 4：`[WinError 5] 拒绝访问`

批量安装时 pip 可能在重命名 `.dist-info\RECORD` 文件时报"拒绝访问"，通常是杀毒软件实时扫描锁住了文件。三个应对办法：

1. **重跑同一条命令**，已装好的包会跳过，往往第二次就过
2. 还不行就**逐个安装**，失败不影响其他包：

```powershell
grep -vE '^\s*#|^\s*$' requirements.txt | While-Read pkg {
  .\venv\Scripts\python.exe -m pip install --no-deps -q -i https://mirrors.cloud.tencent.com/pypi/simple $pkg
}
```

3. 注意 `--no-deps` 会漏掉传递依赖（如 `aiohappyeyeballs`、`annotated_types`、`huggingface_hub`），缺哪个报 `ModuleNotFoundError: No module named 'xxx'`，就把 xxx 补进安装列表再跑一次。

---

## 第 4 步：下载模型（镜像速度差距是 100 倍）

### 实测下载速度（SD1.5 模型，4.27GB）

| 源 | 实测速度 |
|---|---|
| **ModelScope（魔搭）API 直链** | **15.4 MB/s，约 5 分钟** |
| hf-mirror.com | 584 KB/s，要 2 小时 |
| huggingface.co | 国内无法访问 |

ModelScope 直链格式：

```
https://modelscope.cn/api/v1/models/{组织}/{模型名}/repo?Revision=master&FilePath={文件名}
```

SD1.5 示例：

```powershell
cd F:\ComfyUI\models\checkpoints
curl.exe -L -C - --retry 5 -o v1-5-pruned-emaonly.safetensors "https://modelscope.cn/api/v1/models/AI-ModelScope/stable-diffusion-v1-5/repo?Revision=master&FilePath=v1-5-pruned-emaonly.safetensors"
```

**通用规则**：任何 HuggingFace 链接，把域名 `huggingface.co` 换成 `hf-mirror.com` 即可国内下载。想更快的，去 modelscope.cn 搜同款模型。

另外建议在启动脚本里设置环境变量，让 ComfyUI 内部下载也走镜像：

```bat
set HF_ENDPOINT=https://hf-mirror.com
```

### 模型体积与内存的关系（8GB 机器必看）

| 模型 | 体积 | 8GB 内存能否扛住 |
|---|---|---|
| SD1.5 pruned（fp32） | 4.27 GB | 勉强，见第 6 步优化 |
| sd_turbo | 5.2 GB | 危险，容易爆 |
| SDXL / Flux | 6.5 - 23 GB | 不可能 |

---

## 第 5 步：一键启动脚本

在 `F:\ComfyUI\` 下新建 `start_comfyui.bat`：

```bat
@echo off
chcp 65001 >nul
title ComfyUI (CPU mode)
cd /d "F:\ComfyUI"

set HF_ENDPOINT=https://hf-mirror.com

venv\Scripts\python.exe main.py --cpu --use-split-cross-attention

pause
```

双击运行，看到 `To see the GUI go to: http://127.0.0.1:8188` 后，浏览器打开该地址。

以后 ComfyUI-Manager 装插件缺依赖时，同样用腾讯云源补装。

---

## 第 6 步：8GB 内存的生存法则（核心章节）

### 血泪教训：直接跑一张图，系统假死

SD1.5 的 fp32 权重本身 4.27GB，推理峰值内存 6-7GB，8GB 物理内存的机器一动就耗尽，系统疯狂读写页面文件导致整机假死，只能强制关机。

### 必做三件事

**1. 加大虚拟内存**（治本）

`Win + R` → `sysdm.cpl` → 高级 → 性能设置 → 高级 → 虚拟内存「更改」→ 取消自动管理 → 选个剩余空间大的盘（如 F 盘）→ 自定义大小：初始 `8192`，最大 `16384` → 设置 → 重启。

有了足够的页面文件，内存吃紧时系统只会变慢，不会假死。

**2. 启动参数加 `--use-split-cross-attention`**

把注意力计算拆成小块，内存峰值明显下降，速度略降。

**3. 纪律性操作**

- 分辨率锁死 **512×512**，不要往上调
- 出图时关掉浏览器多余标签、微信
- 点了 Run 之后别动鼠标，等进度
- **卡住先等 10 分钟**——那是在读写页面文件，不是死机，强制关机会前功尽弃

### 合理的参数

| 参数 | 值 | 说明 |
|---|---|---|
| steps | 12（想快）/ 20（追求质量） | SD1.5 标准 20 步 |
| cfg | 7.0 | |
| sampler / scheduler | euler / normal | |
| 分辨率 | 512×512 | 极限就是极限 |

---

## 第 7 步：界面操作要点

1. 首次打开是空白画布：左上角菜单 → `Workflows` → 加载默认文生图工作流，或直接把 `.json` 工作流文件**拖进画布**
2. `Load Checkpoint` 节点里选你下载的模型
3. 提示词必须**英文**，写在连到 `positive` 端口的 CLIP Text Encode 节点里
4. 点右上角 `Run`
5. 出图在 `SaveImage` 节点预览，文件在 `ComfyUI\output\`
6. **同一个 seed + 同一段提示词 = 同一张图**，想换效果必须改 seed

---

## 常见问题速查表

| 症状 | 原因 | 解法 |
|---|---|---|
| `git 无法识别` | git 不在 PATH | 加 PATH 或装正式版 |
| `py -3.11 无法创建进程` | 那个版本路径已失效 | 用 `py -0p` 找真实可用的版本 |
| pip 装到一半卡死不动 | 镜像源限速/挂起 | 换腾讯云源，`Ctrl+C` 后重跑 |
| `No matching distribution` | 清华源缺新包 | 换腾讯云源 |
| `[WinError 5] 拒绝访问` | 杀软锁文件 | 重跑 / 逐个装（见坑 4） |
| `ModuleNotFoundError` | `--no-deps` 漏了传递依赖 | 缺啥补啥 |
| HuggingFace 下载不动 | 国内被墙 | 域名换 `hf-mirror.com` 或用 ModelScope |
| hf-mirror 也只有几百 KB/s | CDN 限速 | 换 ModelScope API 直链（15MB/s） |
| 出图时系统假死 | 物理内存耗尽 | 加虚拟内存 + split attention + 512 分辨率 |
| 界面提示缺三个模型 | 点开的是 Flux/SDXL 模板 | 别下，用默认 SD1.5 工作流 |
| 界面能开但点了 Run 没反应 | 正在加载 4.27GB 模型 | 等几十秒，正常 |

---

## 硬件路线总结

最后给个诚实的大实话：

- 本机 CPU 版：**适合学习**节点连线、工作流逻辑，出图当玩具
- Google Colab 免费 T4：出图快 10 倍以上，流程完全一样，缺点是有时长限制
- AutoDL 等云 GPU 租用：约 1-2 元/小时，想认真出图（尤其 SDXL/Flux）就走这条

---

*实测环境：Windows 10/11 · Python 3.12.14 · PyTorch 2.15.0+cpu · ComfyUI（2026-11 版）· SD1.5*
