# 交接文档

写于 2026-11-12。给「隔一段时间回来继续」或「换人接手」用。

---

## 1. 两个仓库是什么关系

| 仓库 | 可见性 | 内容 |
|---|---|---|
| `leo123d/read-os` | 私有 | 固件源码、方案、环境踩坑、UI 模拟器 |
| `leo123d/read-os-flash` | 公开 | 网页刷写页 + 合并后的固件二进制 |

**为什么要拆两个**：GitHub Pages 对私有仓库要付费套餐，免费账号只能从公开仓库发布。源码保持私有，只把「页面 + 一个 bin」公开。

刷写页地址：https://leo123d.github.io/read-os-flash/

---

## 2. 现在做到哪一步

固件只做一件事：读书。基于 MindReset 的 Read Pico 出厂演示固件（Apache-2.0），重写了 `main/` 应用层，`components/` 硬件层沿用官方实现。

已完成：

| 模块 | 状态 |
|---|---|
| 页面清单精简 | 15 个演示页 → 只留阅读页 |
| 断网 | WiFi / 传书 / 二维码组件全部移除 |
| 中文排版 | 禁则、段首缩进、行距可调、两端对齐、中西文间隙、段间不留空 |
| 阅读页 | 页眉页脚全隐、全书完、低电提示、字号 32–56 |
| 工具条 | 目录 / 字号± / 行距 / 边距 / 书签 / 书架 / U 盘 |
| 书架 | 6 本/页、细线分隔、点上下缘翻页 |
| 欢迎页 | 书名 + 上次进度 |
| 屏保 | 卡上 JPG/PNG + 解码缓存 |
| U 盘模式 | TinyUSB MSC，退出重启重扫 |
| 书签 | NVS 存章号+字节偏移，8 条上限 |
| 低电关机 | ≤5% 连测三次，先保存再断电 |

体积：**2.32 MB / 4 MB 分区，剩余 45%**。

**已验证**：能编译、能烧、能进欢迎页与正文。
**未验证**：禁则落地位置、对齐字距手感、行距松紧、屏保解码耗时、U 盘模式能否被电脑识别、低电阈值是否合理。

---

## 3. 改了固件之后，怎么更新刷写页

这是最关键的一条，别漏。

**第一步：编译**

```powershell
$env:IDF_PATH="C:\Espressif\frameworks\esp-idf-v6.1"
$env:IDF_TOOLS_PATH="C:\Espressif\espressif"
. "$env:IDF_PATH\export.ps1"
Set-Location C:\work\read-os        # 纯英文路径，不能有中文
idf.py --no-ccache build
```

**第二步：合成单文件**

```powershell
python -m esptool --chip esp32s3 merge-bin `
  -o read-os.bin --flash-mode dio --flash-freq 80m --flash-size 16MB `
  0x0 build\bootloader\bootloader.bin `
  0x8000 build\partition_table\partition-table.bin `
  0x10000 build\Read_Pico.bin
```

**第三步：替换并推送**

```powershell
Copy-Item read-os.bin C:\work\read-os-flash\firmware\read-os.bin -Force
cd C:\work\read-os-flash
git add -A; git commit -m "firmware: <改了什么>"; git push
```

Pages 约 30 秒后自动生效。**不需要手动发布**。

顺手把 `manifest.json` 里的 `version` 改一下，方便回看烧的是哪一版。

---

## 4. 环境

| 项 | 值 |
|---|---|
| ESP-IDF | v6.1，目标 esp32s3 |
| 串口 | COM3（USB JTAG/serial，VID_303A） |
| 代码路径 | 必须纯英文，中文路径会让 xtensa-gcc 读不到 `picolibc.specs` |
| 编译 | 必须 `idf.py --no-ccache build` |

四个新环境一定会踩的坑：

1. **路径含中文** → 编译直接失败，报找不到 specs
2. **在 git bash 里跑 `idf_tools.py`** → `MSys/Mingw is not supported`，一律用 PowerShell
3. **工具链下载 502** → 先设 `$env:IDF_GITHUB_ASSETS="dl.espressif.cn/github_assets"`
4. **ccache 随机卡死** → 关掉；`-DCCACHE_ENABLE=OFF` 无效，idf.py 会覆盖回 True

---

## 5. 代码在哪

```
main/apps/app_book.c        阅读页、书架、书签、工具条（约 2000 行，最重）
main/book/book_layout.c     排版引擎：断行、禁则、对齐、分页
main/book/book_marks.c      书签存储（NVS blob）
main/book/book_progress.c   阅读进度
main/screensaver.c          休眠屏保 + 解码缓存
main/usb_msc.c              U 盘模式
main/settings.c             NVS 设置：字号/行距/边距/字体/睡眠
components/read_pico/       板级：SD、显示时序、按键
components/e0470_epaper_waveform/  波形表与裁剪工具
docs/ui-mock/simulator.html 交互模拟器（改 UI 先在这里看）
```

三条设计约束，改代码前必须知道：

1. **断行与绘制共用一套判定**。`take_line()` 同时供分页、绘制、图片命中使用；任何一处单独改算法，就会出现「量的是一行、画的是另一行」。
2. **位置用字节偏移保存，不用页码**。字号/行距/边距一改，页码全废；偏移永远有效。
3. **U 盘模式不能用原来的卡对象**。`esp_vfs_fat_sdcard_unmount()` 内部会 `free(card)`，必须重新 `sdmmc_card_init()` 一张裸卡（见 `read_pico_sd_detach_for_usb()`）。

---

## 6. 待办（按优先级）

| # | 事项 | 说明 |
|---|---|---|
| 1 | 实机核对排版 | 禁则位置、字距、行距 1.7 是否偏松、上边距 64 是否合适 |
| 2 | U 盘模式实测 | 能否被电脑识别、退出后能否正常重扫、中文文件名是否正常 |
| 3 | 翻页提速 | DU 波形从 20 相裁到 16/14 相，可省 45–65 ms；改 `e0470_epaper_waveform.c` 的阈值 DU 构建 |
| 4 | 欢迎页封面 | 需要 EPUB 封面提取与图片缓存 |
| 5 | 书架搜索入口 | 代码还在（`read_pico_search` 组件），UI 上没入口 |
| 6 | 低电阈值校准 | 5% 是否真的对应快没电，需要放电实测 |

---

## 7. 常见故障

| 现象 | 原因 | 处理 |
|---|---|---|
| `Could not open COM3 ... Access is denied` | 上次烧写后 Windows 没放句柄 | 拔插 USB；或用网页刷写页 |
| 系统里完全看不到设备 | 设备在休眠（USB 断电） | 按电源键唤醒；或按住 BOOT 插线进下载模式 |
| 编译卡在某个 .c 不动，CPU 不涨 | ccache / 杀软干扰 | 关 ccache，重试；实在不行重跑一次 |
| `git push` 报连不上 github.com | 本地代理挂了 | 检查 `HTTP_PROXY`；直连 443 通时用 `--noproxy` |
| 烧完屏幕没反应 | 电池没电 / 卡没插 | 插 USB 供电再看 |

---

## 8. 硬件事实

ESP32-S3（16 MB flash / 8 MB PSRAM）、4.7" 1216×684 16 级灰阶墨水屏（SY7636A 供电）、CST836U 触控、SC7A20H 加速度计、CW32L010 电源管理、TF 卡 1-bit SDMMC。

- 面板刷新：DU 20 相 × 11.09 ms ≈ 222 ms；GC16 36 相 ≈ 400 ms
- 帧周期 11.09 ms 是厂家设计点，缩短会牺牲成像，不要拿它调速度
- 改波形表会使设备失去保修

---

## 9. 刷写方式

| 方式 | 命令 |
|---|---|
| 网页 | 打开 https://leo123d.github.io/read-os-flash/ ，Chrome/Edge，点按钮 |
| 命令行 | `idf.py -p COM3 flash` |
| 单文件 | `esptool --chip esp32s3 -p COM3 write-flash 0x0 read-os.bin` |

网页刷写和命令行互不干扰，后者失败时前者往往还能用（端口由浏览器独占打开，绕开系统句柄问题）。
