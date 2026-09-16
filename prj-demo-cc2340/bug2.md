# TEST_START载荷flat布局不符UM12161规格_已改TLV

## 问题
encode_test_start TEST_RUN 原为 flat 布局（handle(4)+channel(1)+mode(1)+num_packets(2)+t_gap(2)，len=10），与 UM12161 2.6 §10.4.2.1 Table 85/87 的 TLV 参数表不符（规格：参数个数 + ID/Len/Value，MODE=0x00、DELAY=0x01(µs,1000-65535)、FRAME_TYPE=0x02、PSDU=0x03、TIME_OUT=0x04、EVENT_COUNTER_MAX=0x05(4B)；无 handle 字段，channel 非该命令参数，属会话 app config 域）。

## 修复
改 TLV：参数个数=3，MODE(1B)+DELAY(2B,µs,范围钳位)+EVENT_COUNTER_MAX(4B)；channel 不编码（留字段注释说明）。黄金向量 test_mode_wire_golden 期望字节同步换规格 TLV。uwb_host commit 1134fc7（2026-09-16）。
