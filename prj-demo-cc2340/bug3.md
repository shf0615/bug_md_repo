# NCJ29D6_app固件11.0.1不响应UCI_TEST_START_模块侧无实现_已知限制

## 问题
host 侧修复（EAGAIN 重试 + TLV 载荷）后上板验证：TEST_START 无论 TLV 还是旧 flat 载荷，模块（GID 0xE OID 0x20 归属符合 UM12161 Table 7）均无 RSP，仅回 DEVICE_STATUS_NTF(READY) 并终结 test 会话。RX 全帧注入打印实证：INIT_RSP → SESSION_STATUS_NTF → DEVICE_STATUS(READY) 后 1s 无任何帧 → 超时。同组其他专有命令（0x13 reboot reason / 0x14 temperature 等）正常。模块为 UWBMAC AiO app 固件 Foundation 20.1.1 / UCI 2.5.0 / FW 11.0.1；UM12161 2.6（2025-09）对应更新的 Foundation 线。注意：Testware 固件是另一套 GID 0x0A 命令集（UM11769），与本问题无关。

## 修复
host 侧无需再修（行为已与规格一致、失败有界有日志）。打通 test mode 需升级模块固件到含 §10.4 实现的版本——docs/NCJ29D6_releases/ 有 Foundation v21.1.1 / v22.0.0 包，可走 uwb otarun OTA 管线推送；升级影响测距等既有功能，评估后另行实施。
