# camera_PC

`camera_PC` 是一个基于 Qt 的摄像头采集与 XDMA 联调项目，核心目标是把视频数据在 PC 侧先标准化封包，再按可配置批次稳定发送到 FPGA。

## 1. 项目思路

项目按“采集、协议、发送、控制”四层拆分，避免把业务流程和硬件细节耦在一起：

1. 采集层（`cameraprobe.*`）
- 枚举摄像头模式。
- 执行 one-shot 单帧抓拍。
- 向上层回传原始帧数据与元信息。

2. 协议层（`video_packet_batcher.*`）
- 将原始视频流切成固定 `1024B` 协议包。
- 严格写入头字段（`EB 90`、长度、路由字段）。
- 尾包不足 `1006B` 时补零。

3. 发送层（`widget.cpp` 中 XDMA 子模块）
- 将协议包流累计为“可配置批次”（默认 `1MiB`）。
- 每个批次触发一次 XDMA 单次写入（single write）。
- 不满一个批次的数据缓存等待后续补齐。

4. UI 控制层（`widget.ui` + `widget.*`）
- 按钮分组控制：采集/视频发送、XDMA/测试。
- `写入大小(KB)` 控件可直接控制视频主链路每次写入 XDMA 的数据量。
- 提供日志观察点，便于联调与定位。

这个分层的直接收益是：
- 协议规则可独立验证，不依赖硬件。
- 发送策略统一，实时流和手动发送都走同一条封包+聚合链路。
- UI 只负责编排，不直接处理底层协议细节。

## 2. 协议与批量发送规则

### 2.1 固定包格式（1024B）

每包严格为：

`EB 90 | lengthH | lengthL | dest(6B) | source(6B) | priority(2B) | payload(1006B)`

字段说明：
- `lengthH/lengthL`：当前包 `payload` 的实际长度（高/低字节）。
- `payload`：每 `1006B` 原始视频数据生成 1 包。
- 最后一包不足 `1006B` 也必须生成，剩余区域补 `0`。

### 2.2 可配置批次聚合发送

- 协议包按批次累计后发送，默认批次为 `1MiB`（`1024 * 1024B`）。
- 每个批次只进行一次 XDMA 写入（single write）。
- 未凑满一个批次的包数据保存在内部缓存，等后续视频数据补齐再发送。
- 批次大小可通过 UI 的 `写入大小(KB)` 调整（代码要求按 `1024B` 包边界对齐）。

## 3. 核心功能

1. 摄像头枚举与模式检查。
2. 单帧抓拍（优先 YUY2/YUYV，含回退策略）。
3. 单帧缓存发送：缓存帧 -> 封包 -> 批次聚合 -> XDMA。
4. 实时视频发送：预览帧 -> 封包 -> 批次聚合 -> XDMA。
5. XDMA 通道打开与 ready_state 自检。
6. XDMA 链路测试包发送（不依赖相机）。
7. 协议封包模块手动自测（纯软件，不依赖 XDMA）。

## 4. UI 按钮功能分配

### 4.1 采集与视频发送

- `列出模式`
  - 枚举相机与模式，快速检查驱动返回。
- `采一帧`
  - 采集一帧并缓存；若是 YUYV 同时导出预览 PNG。
- `发送缓存帧(封包+批量)`
  - 将最近缓存帧走协议封包与批次聚合后发送。
- `开始/停止实时视频发送(封包+批量)`
  - 开关实时发送，预览可继续显示。

### 4.2 XDMA 与测试

- `打开XDMA通道并自检`
  - 打开 `user` + `h2c_0`，执行 `ready_state`。
- `发送XDMA链路测试包`
  - 发送固定 4KB 链路测试包验证 PC -> FPGA。
- `运行协议封包自测`
  - 手动执行封包/聚合规则自测（软件侧）。

### 4.3 运行时调参控件

- `节流间隔(ms)`
  - 控制实时视频发送最小间隔。
- `写入大小(KB)`
  - 控制视频主链路每次向 XDMA 写入的数据量（通过聚合批次大小实现）。

## 5. 调用链说明

### 5.1 缓存帧发送链路

`采一帧` -> 缓存 `m_lastCapturedFramePayload` -> `sendVideoPayloadWithBatching()` -> `VideoPacketBatcher::enqueueVideoPayload()` -> 输出“当前配置大小”的批次 -> `sendXdmaPayload(..., forceSingleWrite=true)`

### 5.2 实时视频发送链路

`QVideoProbe::videoFrameProbed` -> `onPreviewFrameProbed()` -> `sendVideoPayloadWithBatching()` -> `VideoPacketBatcher::enqueueVideoPayload()` -> 输出“当前配置大小”的批次 -> `sendXdmaPayload(..., forceSingleWrite=true)`

### 5.3 XDMA 测试链路

`sendXdmaTestPacket()` -> `sendXdmaPayload(..., forceSingleWrite=false)`

说明：
- 视频主链路按“当前写入大小”强制单次写入。
- 测试包与历史兼容路径仍可走分块发送逻辑。

## 6. 使用方法

### 6.1 运行前准备

1. 安装 Qt（项目依赖：`core gui widgets multimedia multimediawidgets`）。
2. 保证摄像头设备可被系统识别。
3. 保证 XDMA 驱动与设备节点可用。
4. 保证 `driver/` 下 `XDMA_MoreB.dll/.lib` 可用。

### 6.2 推荐联调流程

1. 点击 `列出模式`，确认相机和模式信息正常。
2. 点击 `打开XDMA通道并自检`，确认通道打开成功。
3. 点击 `发送XDMA链路测试包`，确认基础链路可写。
4. 点击 `运行协议封包自测`，确认协议模块 PASS。
5. 设置 `写入大小(KB)`（默认 1024 KB，可按联调需求调整）。
6. 选择业务发送路径：
- `采一帧` -> `发送缓存帧(封包+批量)`。
- 或直接 `开始实时视频发送(封包+批量)`。

## 7. 手动自测说明

当前自测为手动触发，不会自动执行。

入口：`运行协议封包自测`（按钮）

自测覆盖点：
- 包头是否为 `EB 90`。
- `lengthH/lengthL` 是否等于实际 payload 长度。
- 尾包不足 `1006B` 时是否正确补零。
- 默认配置下，连续输入 1024 个协议包后是否刚好输出 1 个 `1MiB` 批次。
- 批次输出后缓存是否正确归零。

## 8. 日志速查

- `[OK] XDMA open complete: user + h2c_0 ready.`
  - XDMA 会话已就绪。
- `[CFG] XDMA write size set to ... KB`
  - 视频主链路批次写入大小已更新。
- `[PKG] ... raw=... packets=... emitted=... x ...KB, cached=...`
  - 封包与聚合行为观察点（包含当前批次大小）。
- `[SELFTEST] PASS ...`
  - 协议自测通过。
- `[ERROR] XDMA single-write failed ...`
  - 当前批次的单次写入失败。

## 9. 代码结构

- `widget.ui`
  - 主界面按钮与日志区域。
- `widget.h / widget.cpp`
  - UI 编排、实时回调、XDMA 通道管理、发送桥接。
- `cameraprobe.h / cameraprobe.cpp`
  - 相机模式枚举、单帧抓取、抓拍回调。
- `video_packet_batcher.h / video_packet_batcher.cpp`
  - 1024B 协议封包、可配置批次聚合、缓存与自测。
- `driver/`
  - XDMA 动态库与导入库。

## 10. 备注

- 目前 `dest/source/priority` 使用集中默认常量（见 `VideoPacketBatcher::defaultRouteFields()`）。
- `写入大小(KB)` 的本质是修改视频主链路聚合批次大小，要求按 `1024B` 包边界对齐。
- 后续如果需要切换路由字段，可扩展为 UI 配置或配置文件加载。
