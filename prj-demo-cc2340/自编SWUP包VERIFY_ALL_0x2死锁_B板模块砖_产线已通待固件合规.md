# 自编SWUP包VERIFY_ALL_0x2死锁_B板模块砖_产线已通待固件合规

## 问题（2026-09-16，B 板 601012352）
自编模块固件（ncj29d6-spi6 app，含 TEST_SESSION）SWUP 包推送：66/66 段全部传输 ACK，VERIFY_ALL 稳定 0x2（STATUS_FAILED）。APP-only/factory 变体/修剪 sectorTable/hostifBoot 复原均无效。第一轮传输**最后一段恒 -71**（疑 AppInfo/保护页），第二轮重传"成功"后 verify 仍挂。之后官方 v22 包同样 0x2（模块 app 区已被自编数据污染，官方包不覆盖污染页）→ flash 脏 + verify 死锁 + DEACTIVATE 被拒（transfer 态）+ 断电不解（激活/标志持久）。**模块深砖，SPI6 无法恢复**（SWD 未引出）。

## 已打通的产线（仓库内修复，固件合规后可直接复用）
- 构建：phscaUwb.c local_radar.h include 提守卫、bsp/LocalRadar.c 全量桩（20 钩子 no-op）、post_build python→python3、ImageIntegrity.py 去 crcmod（自实现 CRC-32/MPEG-2 已对拍标准向量 0x0376E6E7 ✓）
- 打包：pack.sh 可在 WSL 宿主跑（Generator.exe chmod +x 后 interop 直调）；APP-only/factory 变体生成法（app.yaml 去 hostif 组件/裁 sectorTable）
- cc2340 分区：FLASH_PART_PKG 264K→192K（304KB 容自编包）+ pack.sh offset 0x30000
- OTA 传输链路本身健康（自编包 66 段全 ACK ~1.8s）

## 教训
自编包 verify 失败**没有退路**（官方包 verify 失败可整包重推恢复，自编包失败即污染 flash）——推自编包前必须在能擦除的环境（SWD 可达）预演。下一步：NXP SWUP Recovery Tools（需引出 SWD）或换模块；向 NXP 确认自编 app hex 进包的 VERIFY 要求（疑 scatter hex 间隙填充/CRC 域与模块校验规则不符——传输全过但读回不匹配）。
