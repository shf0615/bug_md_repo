# 冷上电首次SPI读CRC_fail(recv_err_-77)_尝试加ready握手导致死机已回退

## 问题描述
现象: 冷上电/复位启动后 ~0.46s RTT 日志报 'CRC fail calculated_crc 0x1dae, pkt_crc 0x9898' + 'inst0 recv err: -77'，模块冷启动的第一包 DEVICE_STATUS NTF 被丢弃；之后通信完全正常(uwb temp/info 均可往返)，热复位(uwb reset)不复现。定位: DK-Platform ecual/common/drivers/ecual_uci.c 中 ecual_uci_write 有 MISO-as-ready 握手(切GPIO等wait_slave_ready再时钟)，ecual_uci_read 完全没有——冷上电从机FIFO未就绪时主机时钟读出垃圾导致CRC失败。影响: 仅启动日志一行错误+丢首个NTF，功能自恢复，属轻微问题。

## 修复方法
已尝试修复并回退：在 ecual_uci_read 中镜像写路径加 wait_slave_ready 握手 → 系统开机即冻死(shell/RTT全无响应，uwb高优先级任务疑似饿死所有任务，PC卡在__udivmoddi4(64位除法,R5=0xF4240=1e6除数)，与 udelay/US_TO_JIFFIES/gettimeofday 时间换算特征吻合)；改为有界迭代循环版本仍冻死(疑烧录未生效或另有根因)。已完整回退 ecual_uci.c 至原版并验证板子恢复正常(shell活、uwb temp 29C)。下次再修的建议: (1)先用/gdb单步第一次读路径确认卡点 (2)勿用udelay/gettimeofday做早启动等待(该平台jiffies/时间源在此场景表现异常)，改用硬件定时器或纯CPU空转计数 (3)注意此板JLink烧录偶发假成功，验证以RTT行为为准
