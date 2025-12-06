# SBF 子系统说明（PPP‑B2b）

本目录实现 Septentrio SBF 流的基础解析，以及对北斗 PPP‑B2b 广播消息的提取与纠错映射，供实时精密定位使用。

## 主要组件

- `SBFDecoder`：轻量 SBF 帧解析器，仅做同步、长度与类型提取，并把 4242（BDSRawB2b）块交给 B2b 解码。
- `PPPB2bDecoder`：B2b 负载处理核心，完成导航比特解码、消息结构解析、轨道/钟差缓冲与转换、结果发出。
- `SBFcoDecoder`：LDPC 纠错器，用于对 B2b 导航比特进行纠错（BCNV3，GF(2⁶) 扩展最小和算法）。
- 其他：`rtklib.h` 及相关类型，承载 RTCM/SSR 映射所需基础结构。

## 数据流与职责

- 输入：持续的 SBF 字节流，其中每帧以 `0x24 0x40` 同步开头，头部包含长度与类型。
- `SBFDecoder`：
  - 同步检测与帧切分；校验 CRC16‑CCITT；记录类型列表。
  - 对每帧调用 `PPPB2bDecoder::input(b,len)`；当类型为 4242 时进入 B2b 解码流程。
- `PPPB2bDecoder`：
  - 解析 B2b 头（TOW、WNc、SVID 等），提取 31×4 字节导航比特；
  - 通过 `SBFcoDecoder::decode_LDPC_navbitsRaw()` 纠错得到净荷；
  - 使用 `b2b_parsecorr()` 填充内部 `ppp_ssr_orbit/clock/mask` 结构；
  - 将轨道（RAC）与钟差（C0）映射为 RTCM3 风格的 `t_orbCorr/t_clkCorr` 列表，并按历元缓冲与发出。

## 关键函数

- `SBFDecoder::Decode(buffer, len, errmsg)`（`SBF/SBFDecoder.cpp:124`）：
  - 同步、取帧、CRC 校验、类型记录；把帧传入 B2b 解码。
- `PPPB2bDecoder::input(sbf_block, len)`（`SBF/PPPB2bDecoder.cpp:226`）：
  - 识别 4242 并调用 `decode_b2b_payload()`。
- `PPPB2bDecoder::decode_b2b_payload(payload, payload_len)`（`SBF/PPPB2bDecoder.cpp:240`）：
  - 解析头域与导航比特；调用 `SBFcoDecoder::decode_LDPC_navbitsRaw()`；构造 `Message_header`；执行 `b2b_parsecorr()`；在成功时调用 `emitCorrections()`。
- `PPPB2bDecoder::emitCorrections(p_sbas)`（`SBF/PPPB2bDecoder.cpp:811`）：
  - 根据消息类型缓冲并转换轨道/钟差，触发 `newOrbCorrections/newClkCorrections` 信号或发送到 `ClockOrbit`。
- `SBFcoDecoder::decode_LDPC_navbitsRaw(navBits)`（`SBF/SBFcoDecoder.h:11`, `SBF/SBFcoDecoder.cpp:215`）：
  - 基于 GF(64) 的扩展最小和迭代译码，输出纠错后的字节序列与错误计数。

## 类型与映射

- `pppdata`（`SBF/PPPB2bDecoder.h:148`）：承载 B2b 消息解析结果，含 `mestype/SSR/BDSweek/sow` 等。
- `ppp_ssr_orbit`（`SBF/PPPB2bDecoder.h:174`）：轨道改正（RAC、URA、IODN 等）。
- `ppp_ssr_clock`（`SBF/PPPB2bDecoder.h:184`）：钟差改正（C0、IODP 等）。
- `t_orbCorr/t_clkCorr`：项目内 RTCM3 风格改正类型，由 `PPPB2bDecoder` 生成与发出。

## 使用建议

- 输入 SBF 流（含 4242），构造 `SBFDecoder(staID)` 连续喂入字节缓冲。
- 通过日志或信号观察 B2b 轨道/钟差输出；必要时绑定到后续 PPP 处理链。
- 若只需 B2b 纠错净荷，可直接调用 `SBFcoDecoder::decode_LDPC_navbitsRaw()`。

## 注意事项

- CRC 校验失败帧会被忽略；导航比特前缀异常（如以 `EC0FC` 开始）也会跳过。
- 纠错参数（迭代次数、EMS 阈值）在 `SBFcoDecoder.cpp` 中设定，如需性能/鲁棒性权衡可调整。
- 周周跳/历元一致性由 `b2b_parsecorr()` 中的时间一致性检查处理，实时流通常以 SBF 的 WNc/TOW 为准。

