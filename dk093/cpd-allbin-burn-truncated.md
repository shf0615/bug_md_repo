# CPD烧all.bin"不跑"两层根因——烧录未写入+V7.96b大镜像编程中断（定稿：V8.16单条烧）

## 现象

CPD 板烧 `Source/CPD/.build/cal1106/image/cal1106_default_all.bin` 后断电重上"固件不跑"，每次必现。两套烧录路径（用户 J-Flash GUI、JLink V7.96b 单条 loadbin）都翻车，极易误判为分区重排（2026-10-28 SBL 0x30040000→0x30020000、APP 0x30076000→0x30050000）引入的 bug。

## 根因（两层独立故障叠加）

**第一层：J-Flash GUI 烧录未写入芯片。** 板上 flash 完整保留 10-21 旧布局旧镜像（mem32 对账：旧槽 0x30040000/0x30076000=96F3B83D 旧数据，新槽 0x30020000/0x30050000=FFFFFFFF 空）。旧 APP 自身在 libuwb UWB 初始化卡死（PC 恒 0x10075B58 RAM 代码，ra=PhyDrv_FineSyncXcorrSizeGet），串口停在 Reset status 后静默——"没烧进去"被伪装成"烧了新版但卡死"。

**第二层：V7.96b 对 ~700KB 连续大镜像编程中断。** `loadbin all.bin 0x30000000` 显示 Programming Done 但尾部未落盘：Erase 覆盖全区、Programming 在 APP 中段截断（0x30021000/0x30036000/0x30051000 读回与文件一致，0x30080000/0x300A0000/0x300AB628 恒 FFFFFFFF）。伴随症状：Program 仅 4.7s（等量三段式 8.8s）、Verify 0.000s 未执行、"Timeout while preparing target, core does not stop"。两次实测复现。

## 修复（定稿）

**CPD 日常烧录 = 单条 all.bin + JLink V8.16**（2026-10-29 全链路验证）：

```bash
cd Source/CPD/boards/cal1106/tools
embedded flash --script burn_allbin.jlink --jlink-version V816 --serial 601012352
```

V8.16：identify 正常、Program 8.4s、Verify 9.3s 完整执行、尾部三点读回与文件逐字一致、断电重上 pbl banner（当日 Build）→ 新槽布局 → validated → ccc ranging 周期测量。
应急回退：V7.96b 分件三段式（`loadbin out/flash.bin 0x30000000` + 两条 `loadfile *.signed.hex`，每段 <400KB 稳定）。V8.56a identify 失败不可用。

## 诊断方法（关键动作）

1. **串口 banner 的 Build 日期 + PBL 打印的槽布局**——一眼分辨板上跑的是哪次构建哪套分区；
2. **halt 后 mem32 抽查槽头+槽中+槽尾三点**——只看槽头会被"烧了一半"骗过；批量读有孤立字节漂移伪影（+0x10/+0x40，如 D738 读成 D778），以多采样点+文件比对为准；
3. all.bin 布局离线可先核实：pack.sh 从 linker.ld 运行期解析槽地址，镜像本身正确。

## 经验

- JLink 版本对 cal110x 分工（V12 探针实测）：**V8.16=烧录主力（all.bin 大镜像必用）**；V7.96b=小镜像/identify 兜底（>500KB 连续 bin 截断）；V8.56a identify 失败勿用。
- `embedded flash` CLI 本质是 JLink Commander 包装，`--jlink-version`/`--serial` 与 `--script` 兼容即可用；纯 .bin 直烧 CLI 无地址参数，地址必须写在脚本里。
- .bin 无地址信息且 J-Flash GUI 是事故多发区（未配 device/flash loader 也"显示成功"）；烧后必须 mem32 读回对账。
- 分区重排后旧槽残留旧数据：上电表现怪异先读新旧槽基址各一个字，分辨板上是哪套布局。
