# uwb_task发送-EAGAIN被判死_多阶段请求撞模块NTF必现静默失败_已修复

## 问题
uwb test start 上板必现静默失败（shell 无任何输出返回 FAILED）。根因链：SESSION_INIT 后模块立刻发 SESSION_STATUS_NTF（nINT 拉低）→ ecual_uci_write 发送前置检查见 nINT 低立即返回 -EAGAIN（ecual_uci.c:148，帧未上线）→ drive_phase 把任何 send<0 当请求失败 inst_complete（无重试、无日志）。注入证据：encode TEST_RUN OK → send ret=-11(-EAGAIN) → 1ms 后 NTF 才被任务收走。H1 的 fake 传输无此拒绝行为，单测天然看不见。

## 修复
新增 INST_SEND_PARK 状态：drive_phase 对 send==-EAGAIN 暂挂请求；inst_try_start 开头重试同相位并跨重试保留原 deadline（同 CRC resend 习惯），永久拒绝按原预算有界超时且有日志。H1 新增 send_eagain_retry_succeeds / send_eagain_permanent_times_out 两用例（fake 加 fail_send 旋钮复刻 nINT-low 拒绝语义）。uwb_host commit 1134fc7（2026-09-16）。
