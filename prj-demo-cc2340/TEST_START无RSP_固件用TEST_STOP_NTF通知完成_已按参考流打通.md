# TEST_START无RSP_固件用TEST_STOP_NTF通知完成_已按参考流打通

## 问题
`uwb test start` 上板必现失败。三层根因（逐层剥开）：① 引擎对发送 -EAGAIN 判死（已修 1134fc7）；② TEST_START 载荷 flat 布局不符规格 + 缺官方参考流的 SESSION_SET_APP_CONFIG 前置相位（已修）；③ **真机固件（FW 11.0.1）对 TEST_START 不回 RSP**——不是没实现 test（vendor app 源码 phscaTest.c 有完整实现），而是跑完测试用 TEST_STOP_NTF(GID 0xE/OID 0x26 带 session handle) 通知完成（新版固件/CAS 源码为 0x21 无载荷且回 RSP），host 等 RSP 必超时。

## 修复（uwb_host 04baf29）
三相位复刻 NXP test_session.py 参考流（INIT→SET_APP_CONFIG 8TLV→TEST_START 9参数TLV）+ 引擎"完成 NTF"语义（请求声明 wait+ntf_match，WAIT_RSP 态匹配 NTF 到达即成功）。B 板实测：`uwb test start` 全链路 ~400ms `test finished` 成功返回。关键方法：CAS Examples zip 里有 NXP 官方主机参考脚本（test_session.py，命令流黄金标准）+ 模块 vendor app 源码（phsca*，可直接读命令处理器的行为），排障时先翻这两个。
