# CAN1启动Hardfault-ecual_flexcan硬编码kCLOCK_Can0

## 问题描述
现象: kw47 (KW47B42ZB7) 烧录后 app_can_task 启动即崩, 串口日志 'CAN1: state 5, set mode 1' 后立即 Hardfault, 固件陷入 崩溃->复位->崩溃 死循环(约0.7s一轮, 10秒复位14次)。Hardfault 寄存器: PC=0x0005fc94(flexcan_enable, DK-Platform/ecual/common/drivers/ecual_flexcan.c:597), LR=0x0006445f(can_set_mode, DK-Platform/ecual/arm/kw45/ecual_flexcan.c:94), CFSR=0x00008200(BFARVALID|PRECISEERR), BFAR=0x4004F000=CAN1_BASE。根因: can_set_mode() 的 CAN_MODE_START 分支中 CLOCK_EnableClock(kCLOCK_Can0) 硬编码了 CAN0 时钟门控, 忽略板级配置的 config->clk_id(kCLOCK_Can1), CAN1 外设时钟未开, flexcan_init() 首次写 CAN1->MCR 即触发精确总线错误。CAN0 能正常启动纯属巧合(硬编码恰好正确)。定位方法: CFSR/BFAR 判定精确总线错误+外设地址, addr2line 解析 PC/LR, 对照 bsp/CAN.c 的 clk_id 配置。

## 修复方法
DK-Platform/ecual/arm/kw45/ecual_flexcan.c:91 将 CLOCK_EnableClock(kCLOCK_Can0) 改为 CLOCK_EnableClock(config->clk_id)。修改后 docker-build.sh 重编并烧录 kw47_default_all.hex, 串口确认 CAN1 正常进入 ERROR_ACTIVE(VEH: canstack ops bound / PduR / Nm 版本日志正常打出), Hardfault 消失, 系统稳定运行不再复位循环。注意: 此文件为 DK-Platform 共享代码, kw45 系列其他板卡同路径生效; 单 CAN 实例板卡行为不变。
