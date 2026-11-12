# 小纸 Pico · 网页刷写

浏览器直接烧写固件，不需要装 ESP-IDF。

**打开：** https://leo123d.github.io/read-os-flash/

## 怎么用

1. 用 Chrome 或 Edge 桌面版打开上面的链接（Safari、Firefox 没有串口 API）
2. USB-C 数据线连设备
3. 点「连接设备并刷写」，选 `USB JTAG/serial debug unit`
4. 等进度条跑完，设备自动重启

刷写会覆盖固件，TF 卡里的书不受影响。

## 仓库里有什么

```
index.html          页面
manifest.json       ESP Web Tools 清单（ESP32-S3，偏移 0）
firmware/read-os.bin  合并后的固件（bootloader + 分区表 + 应用）
esp-web-tools/      ESP Web Tools 10.4.0，内置不依赖 CDN
```

## 固件怎么来的

```bash
python -m esptool --chip esp32s3 merge-bin \
  -o read-os.bin --flash-mode dio --flash-freq 80m --flash-size 16MB \
  0x0 build/bootloader/bootloader.bin \
  0x8000 build/partition_table/partition-table.bin \
  0x10000 build/Read_Pico.bin
```

源码：https://github.com/leo123d/read-os （私有）

## 交接

改动、构建、发布流程与待办都在 [HANDOFF.md](HANDOFF.md)。

## 许可

页面代码随本仓库；ESP Web Tools 为 Apache-2.0；固件基于 MindReset 的 Read Pico 演示固件（Apache-2.0）。
