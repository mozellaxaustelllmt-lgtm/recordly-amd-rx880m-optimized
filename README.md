# Recordly AMD 优化版（Ryzen AI 9 H 365 + Radeon 880M）

> **非官方修改版**。基于开源项目 [Recordly](https://github.com/webadderallorg/Recordly) v1.4.0（AGPL-3.0），
> 针对以下特定硬件做了 CPU→GPU 的负载改造。上游与本仓库无关联。

## 适配硬件

| 项目 | 本版本针对的配置 |
| --- | --- |
| CPU | **AMD Ryzen AI 9 H 365**（10 核 20 线程，Zen5 + Zen5c） |
| 核显 | **AMD Radeon 880M**（RDNA 3.5，VCN 5.x 编解码引擎） |
| NPU | AMD XDNA 2（本版本未使用） |
| 系统 | Windows 11 26H2（26200+） |
| 屏幕 | 2560×1440 @ 120Hz |
| 编码器 | ffmpeg `h264_amf`（AMD AMF / VCN 硬件编码） |

> 关键技术前提：AMD 平台**没有** Media Foundation 硬件 H.264 编码 MFT，
> 所以上游默认走的 Media Foundation / Chromium WebCodecs 编码路径在 AMD 上会静默退回软件编码，
> 白白吃掉 1 个以上 CPU 核心。本版本把录制与导出都改到 ffmpeg 的 `h264_amf` 上。

## 改动清单（相对上游 v1.4.0）

### 1. 录制：采集帧转色与编码全部搬到 GPU
- 新增 `ID3D11VideoProcessor`（`VideoProcessorBlt`）在 GPU 上完成 BGRA→NV12 与缩放
  （原来是用 CPU 标量双层循环逐像素转换，2560×1440 约 12 ms/帧）。
- 新增「ffmpeg 管道 + 硬件编码」通路：自动探测 `h264_amf → h264_nvenc → h264_qsv → h264_mf`，
  按可用性选择，编码交给显卡的 VCN 引擎。
- 采集端改为固定节拍限流（按目标帧率出帧），避免过采样。
- **非 ASCII 路径修复**：helper 的 `argv` 经 CRT 转 ANSI（中文系统为 GBK），原来按 UTF-8 解析导致
  录制到 `C:\Users\<中文名>\...` 时直接失败并退回浏览器录制；现在按 UTF-8 解析失败后回退 `CP_ACP`。
- **stdout 路径编码修复**：`Recording stopped. Output path:` 改为输出 UTF-8，
  否则 Electron 端解析出乱码路径，停止录制后 rename 报 `ENOENT`。

### 2. 导出：编码从 CPU 软编改到 VCN 硬编
- 上游在 Windows 上默认走 `breeze-stream`：渲染进程用 WebCodecs `VideoEncoder` 编码，
  ffmpeg 只做 `-c:v copy`（导出提示里显示 `h264-stream-copy`）。
  但 Chromium 在 AMD 上会「报告硬件加速、实际软件编码」——实测同一台机器
  `prefer-hardware` 37.1 fps **慢于** `prefer-software` 53.4 fps。
- 本版本在两条导出实现（`videoExporter.ts` = Breeze、`modernVideoExporter.ts` = Lightning）都加了判定：
  1. 用 `navigator.mediaCapabilities.encodingInfo().powerEfficient` 判断 WebCodecs 是否真硬件；
  2. 再用 `WEBGL_debug_renderer_info` 读 GPU 厂商做兜底（AMD 的 `powerEfficient` 会误报 true）；
  3. 判定为「非真硬件」时，改用 `rawvideo` 输入让 ffmpeg 编码，并按厂商精确要求
     `amf` / `qsv` / `nvenc`；只拿到软编器（`libx264`/`h264_mf`）就取消并回退，避免「回读 + 软编」双浪费。
- **修复主进程的编码器探测尺寸**：原来用 64×64 试编码，而 AMD `h264_amf` 在该尺寸
  `encoder->Init() failed with error 5`（128×72 同样失败，640×360 才通过），
  导致明明有硬件编码器也被判为不可用。现改为 640×360。
- 去掉了 rawvideo 通路上**重复的整帧翻转**（渲染进程已做一次行翻转，ffmpeg 又加 `vflip`）。

### 3. 其它
- 关闭自动更新（三层：环境变量 `RECORDLY_DISABLE_AUTO_UPDATES=1`、设置项、移除 `app-update.yml`），
  避免升级覆盖上面的补丁。

## 实测效果（同一台机器，2560×1440）

| 场景 | 上游原版 | 本版本 |
| --- | --- | --- |
| 录制 CPU（整屏 1440p60，浏览器兜底 VP9） | 1.48 核 | — |
| 录制 CPU（原生路径） | 1.04 核 | **0.57 核** |
| 录制 GPU 3D | 1.4% | 5.3% |
| 导出编码器 | WebCodecs 软编（`h264-stream-copy`） | **`h264_amf`（VCN）** |
| 导出吞吐 | 24.5 FPS | 23.6 FPS（瓶颈转移到 RGBA 回读） |
| 导出时 CPU 峰值 | 高（软编占满一核以上） | 明显下降 |
| 编辑器窗口最小化时 | GPU 22–25% | GPU 22–25%（上游已有的 `previewRenderLoop` 会停 ticker） |

> 说明：导出墙钟速度**没有明显提升**，因为瓶颈从「CPU 编码」转移到「每帧 14.7 MB 的 RGBA 回读 + IPC」。
> 要再提速需要去掉回读（GPU 纹理直通编码器，或 WebGL 直接输出 NV12），属于另一个量级的改造。

## 安装

1. 运行 `Recordly-Setup-*.exe`；
2. 首次启动会显示欢迎页，点右上角 **✕** 关闭即可（无需登录，界面底部写明「此版本尚不支持云端登录」）。

## 卸载 / 回滚

- 控制面板卸载，或直接删除安装目录；
- 若想回到官方原版：卸载后从上游 Releases 重新安装即可，本版本不修改系统其它位置。

## 许可与合规

- 本仓库包含对 **AGPL-3.0** 项目 Recordly 的修改，按 AGPL-3.0 分发；完整源码见本仓库 `src/`、`electron/`。
- 二进制内捆绑 **FFmpeg**（`ffmpeg-static` 构建，LGPL/GPL 视其配置而定）与 `ffprobe`，
  其许可证文本随安装包 `resources/licenses` 提供；如需对应源码请参考 FFmpeg 官方仓库。
- 上游仓库自带的「FFmpeg 源码清单合规检查」在本地打包时被跳过（仅供个人使用场景）。
  **若你要公开分发此安装包，请先自行补齐 FFmpeg 对应的源码与声明**。
- Recordly 名称与图标版权归上游作者所有，本非官方版本仅用于个人使用与学习。

## 已知限制

- 仅针对 AMD 平台做了主要优化；NVIDIA/Intel 上会走各自对应的硬件编码器（未做充分实机验证）。
- 录制超过 N 分钟的稳定性、窗口/区域录制模式未做回归测试。
- NPU（XDNA 2）未使用：字幕（Whisper/Parakeet）目前仍在 CPU 上推理，可作后续优化方向。
