# UCI_test_mode协议链路通_RF执行被固件立即终止_需合规app固件

## 问题
uwb test mode 验证分两层结论（2026-09-16，B 板实测，host 实现 04baf29/e228e90）：
1. **协议链路全通**：INIT(0xD0)→SET_APP_CONFIG(官方8TLV,RSP OK)→TEST_START(官方9参数)→TEST_STOP_NTF(0xE/0x26带handle)完成，shell 正常返回，H1 黄金向量锚官方参考字节。
2. **RF 执行未发生**：TX 100帧×20ms 应~2s，实测 TEST_START 后 ~9ms 即完成通知；LOOPBACK 同样立即终止且无 TEST_LOOPBACK_NTF；无 LOG_NTF；TIME_OUT TLV 被忽略（固件默认 ~100ms 窗）。真实固件（FW 11.0.1）对 TEST_START 的实际行为是立即终止测试会话。
另：A 板（941000024）模块个体异常三连证——SWUP 不驱动 RDY、test 会话 SET_APP_CONFIG 无 RSP（B 板同版本秒回）、外观版本号全同。

## 修复
host 侧已完成且合规（换合规固件即用）。要真正跑测试需 NXP 提供与 UM12161 2.6/CAS v14.3 行为一致的 app 固件（现固件 TEST_START 立即终止）；A 板模块建议换新。方法沉淀：CAS Examples zip 有官方参考脚本+模块 app 源码（路径为反斜杠）；源码版本与实机有差异，机制看源码、行为以实测为准。
