# CPD J-Flash烧all.bin"不跑"——烧录实际未生效+板上旧镜像UWB初始化卡死

## 现象

CPD 板用 J-Flash GUI 烧 `Source/CPD/.build/cal1106/image/cal1106_default_all.bin`（基址 0x30000000），J-Flash 显示烧录完成，但断电重上后"固件不跑"，每次必现。发生在 2026-10-28 分区重排（SBL 0x30040000→0x30020000、APP 0x30076000→0x30050000）之后，极易误判为分区重排引入 bug。

## 诊断（2026-10-29）

1. 离线核实 all.bin 本身正确：pack.sh 从 linker.ld 运行期解析槽地址，镜像内 SBL@0x30020000/APP@0x30050000 各带 mcuboot 签名头（0x30B）+0x400 向量表，未跨坏区 0x300F3000、未碰 NV。
2. 串口 COM15 抓到启动日志：PBL 打印的是 **Build 2026-10-21 11:44:20**、槽布局 **SBL0@0x40000/SBL1@0x5b000/APP@0x76000（旧布局）**——板上跑的是 10-21 的旧镜像，不是当天 16:49 构建的 all.bin。
3. JLink halt 后 mem32 读回对账（权威判读，勿信批量 verify）：新槽 0x30020000/0x30050000 = `FFFFFFFF`（空），旧槽 0x30040000/0x30076000 = `96F3B83D`（mcuboot magic，旧数据）——**J-Flash 那次烧录根本没写进芯片**。
4. 旧 APP 本身有独立 bug：启动后卡死在 libuwb UWB 初始化（PC 恒为 0x10075B58 RAM 代码，ra=PhyDrv_FineSyncXcorrSizeGet@0x300828a6，三次采样纹丝不动），串口停在 "Reset status: 00000008" 后静默——这造成"不跑"的表象。

## 根因

- 主因：J-Flash 烧录未生效（GUI 黑盒，疑似 device/flash loader 未配对，烧录"成功"但芯片 flash 无任何变化）。
- 次因：板上残留的 2026-10-21 旧 APP 在 UWB 初始化卡死 + 串口无输出，让"没烧进去"伪装成"烧了新版但卡死"。

## 修复

用验证过的 JLink 三段式脚本（`boards/cal1106/tools/burn_new.jlink`，V7.96b + 601012352 + cJTAG 2000kHz + 前置四行）：

```
loadbin out/flash.bin 0x30000000          # CPVT+PBL
loadfile cal1106_sbl_default.signed.hex   # hex 自带槽地址
loadfile cal1106_app_default.signed.hex
w4 0xF0300058 0x0000AA55 / w4 0xF0300054 0x000055AA   # 复位对
```

烧后读回 0x30020000/0x30050000 = 96F3B83D ✓，断电重上：pbl Build 2026-10-29 → 新槽布局打印 → APP validated → CAN 栈 + UWB session 建立（status=0）→ ccc ranging 周期测量持续运行，问题消失。

## 经验

- "烧完不跑"先看串口 banner 的 **Build 日期**和 **PBL 打印的槽布局**——一眼分辨板上跑的是哪次构建、哪套分区，比盲猜镜像内容快得多。
- .bin 无地址信息，J-Flash GUI 烧 bin 是事故多发区（基址错/device 未配/未真写都可能显示成功）；cal110x 一律用工程脚本烧 + halt 后 mem32 读回槽头对账。
- 分区重排后旧槽残留旧数据：新链上电若表现怪异，先 mem32 新旧槽基址各读一个字分辨板上到底是哪套布局。
- `tools/burn_chain2000.jlink` 的读回验证地址仍是旧布局（0x30040000/0x30076000），待更新；判活/烧录一律以 burn_new.jlink（新布局）为准。
