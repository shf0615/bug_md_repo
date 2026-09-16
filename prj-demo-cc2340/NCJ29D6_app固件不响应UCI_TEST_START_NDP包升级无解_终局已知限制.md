# NCJ29D6_app固件不响应UCI_TEST_START_NDP包升级无解_终局已知限制

## 问题
host 侧修复（EAGAIN 重试 + TLV 载荷，1134fc7）后，TEST_START（GID 0xE OID 0x20，TLV/flat 均符合 Table 7 归属）在任何可得固件上均无 RSP：模块仅回 DEVICE_STATUS_NTF(READY) 并终结 test 会话。同组专有命令（0x13/0x14 等）正常，TEST_STOP 正常。

## 修复（终局结论 2026-09-16）
host 侧无需再修。**升级 NDP 包无解**：B 板已升 v22（SBE 13.0.0/DSP 14.3.0，OTA 七判据通过）实测 TEST_START 仍无应答——NDP 包不含 vendor app（FW/Foundation，见 OTA 手册 §5.3），命令处理器在 vendor app 内。需向 NXP 索取含 UM12161 §10.4 test mode 实现的 app 固件变体。Testware（GID 0x0A/phsca，UM11769）是另一套命令集，非替代路径。
