# CPD烧all.bin"不跑"两层根因——J-Flash未写入+单条loadbin大镜像编程中断

## 现象

CPD 板烧 `Source/CPD/.build/cal1106/image/cal1106_default_all.bin` 后断电重上"固件不跑"，每次必现。两套烧录路径（用户 J-Flash GUI、JLink Commander 单条 loadbin）都翻车，极易误判为分区重排（2026-10-28 SBL 0x30040000→0x30020000、APP 0x30076000→0x30050000）引入的 bug。

## 根因（两层独立故障叠加）

**第一层：J-Flash GUI 烧录未写入芯片。** 板上 flash 完整保留 10-21 旧布局旧镜像（mem32 对账：旧槽 0x30040000/0x30076000=96F3B83D 旧数据，新槽 0x30020000/0x30050000=FFFFFFFF 空）。旧 APP 自身在 libuwb UWB 初始化卡死（PC 恒 0x10075B58 RAM 代码，ra=PhyDrv_FineSyncXcorrSizeGet），串口停在 Reset status 后静默——"没烧进去"被伪装成"烧了新版但卡死"。

**第二层：all.bin 单条 loadbin 编程中断（V7.96b + cJTAG2000 + V12 探针实测）。** loadbin 702KB 连续镜像（含大片 0xFF gap）显示 Programming Done 但尾部未落盘：Erase 覆盖全区、Programming 在 APP 中段截断（0x30021000/0x30036000/0x30051000 读回与文件一致，0x30080000/0x300A0000/0x300AB628 恒 FFFFFFFF）。伴随症状：Program 仅 4.7s（三段式等量数据 8.8s）、Verify 0.000s 未执行、"Timeout while preparing target, core does not stop" → 会话级联报错。烧后固件完全不跑。

## 诊断方法（关键动作）

1. **串口 banner 的 Build 日期 + PBL 打印的槽布局**——一眼分辨板上跑的是哪次构建哪套分区，比盲猜镜像内容快得多；
2. **halt 后 mem32 抽查槽头+槽中+槽尾**——只看槽头会被"烧了一半"骗过（本次槽头全对、尾部全空）；注意批量读有孤立字节漂移伪影（+0x10/+0x40，如 D738 读成 D778），以多采样点+文件比对为准；
3. all.bin 布局离线可先核实：pack.sh 从 linker.ld 运行期解析槽地址，镜像本身正确。

## 修复

CPD 链形态一律**三段式**烧录（`boards/cal1106/tools/burn_new.jlink`，每段 <400KB，2026-10-29 两次实测稳定）：

```
loadbin out/flash.bin 0x30000000          # CPVT+PBL 整包
loadfile cal1106_sbl_default.signed.hex   # hex 自带槽地址
loadfile cal1106_app_default.signed.hex
w4 0xF0300058 0xAA55 / w4 0xF0300054 0x55AA   # 复位对（最终复位仍以断电重上为准）
```

或经 embedded CLI 指定版本/探针：`embedded flash --script burn_new.jlink --jlink-version V796b --serial 601012352`。

验证判据：断电重上 → 串口 pbl banner Build 当日 → 新槽布局（SBL0@0x20000/SBL1@0x38000/APP@0x50000）→ APP validated → ccc ranging 周期测量输出。

## 经验

- **all.bin（fw0+gap+sbl+gap+app 连续 ~700KB）单条 loadbin 在本环境不可用**——中断点不固定在块边界，槽头正常也掩护不了尾部缺失；若坚持单镜像需分块 loadbin 或查新版 JLink（V856a identify 失败不可用，未再展开）。
- .bin 无地址信息且 J-Flash GUI 是事故多发区（未配 device/flash loader 也"显示成功"）；烧后必须 mem32 读回对账。
- 分区重排后旧槽残留旧数据：上电表现怪异先读新旧槽基址各一个字，分辨板上是哪套布局。
- `tools/burn_chain2000.jlink` 读回验证地址仍是旧布局（0x30040000/0x30076000），勿再用。
