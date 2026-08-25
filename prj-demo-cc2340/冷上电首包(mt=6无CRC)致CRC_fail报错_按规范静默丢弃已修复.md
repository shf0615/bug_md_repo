# 冷上电首包(mt=6无CRC)致CRC_fail报错_按规范静默丢弃已修复

## 问题描述
现象: 模块冷上电后首个 SPI 包触发 'CRC fail (calc=0x1dae, pkt=0x9898)' + 'inst0 recv err: -77'，首包被丢。定位(增强日志解码): 该包 mt=0x6(RFU) gid=0 oid=2 len=2 payload=02 f0，是格式完好的模块上电产物但无 CRC(尾随 0x9898 为 FIFO 填充)；UM12161 §2.3.1 规定 RFU-MT 包应静默丢弃。热复位/运行中重启时模块发的是正常 mt=3 DEVICE_STATUS(CRC 有效)。注: MISO-ready 信号只管写方向(读期间恒高)，与读无关。

## 修复方法
DK-Platform ecual/common/drivers/ecual_uci.c ecual_uci_read: CRC 失败且 MT∈[4,7](RFU) 时按规范静默丢弃，LOG-INF 一行 'boot packet discarded'，返回 -EAGAIN；其余 CRC 失败仍报 -EBADMSG(保留增强日志)。DK-Platform components/uwb_host/src/uwb_task.c inst_handle_rx: -EAGAIN 静默返回不报错。验证: mw/JLink 直写 GPIO DIO12(0x4002310C, 低有效) 做模块断电循环，mt=6 产物被静默丢弃、无 recv err、mt=3 DEVICE_STATUS 正常分发、uwb temp 正常。教训: (1)早启动路径勿用 udelay/gettimeofday 做等待(会死循环饿死任务) (2)MISO-ready 不是读门控 (3)此板 JLink 烧录偶发假成功，验证以 RTT 行为为准
