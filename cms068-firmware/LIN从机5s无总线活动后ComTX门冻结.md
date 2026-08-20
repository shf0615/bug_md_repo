# LIN从机5s无总线活动后ComTX门冻结

## 问题描述
台架(无LIN master)上电约8s后,cms_info_set的所有状态变化不再进LIN帧0x1F(ModSts/VoltgTooHi/环境光byte6全冻结;tst lin tx get读LinIf层LinIfFixedFrameSdu恒为旧值)。真车master持续轮询不触发,台架专属假死,自动化测试用例成片FAIL。根因:LinSM_WakeUp_Confirmation启动GoToSleepTimer(LinSMSleepTime=5000ms),唯一重置点LinSM_EventIndication(总线活动);无master轮询时LinSM_TimerTick判过期→LinSM_RequestComMode(NO_COMMUNICATION)→LinSM_GotoSleep_Confirmation→Com_IpduGroupStop→groupActiveStatus=0→Com_MainFunctionTx的IS_START门全跳过。注意:入睡后喂狗救不回(EventIndication只重置timer不重开Com门)。受控实验:t<8s帧随状态活,t>=8s全冻结。

## 修复方法
tst lin keepalive <on|off>(CMS-Public cms_test/cms_test_commu.c,仅CONFIG_CMS_TEST_SIM编入):on时立即LinSM_RequestComMode(0,FULL)(救已睡通道,走唤醒链重开Com门)+LinSM_EventIndication(0)(立即喂狗),PERIODIC_TASK注册4s周期续喂(<5s窗口);off/复位恢复真实总线驱动。测试主机(CMS-TEST)建连按profile lin_keepalive自动开;将来台架配LIN主机硬件改false即回归真实睡眠/唤醒(可反向测试睡眠策略)。验证:真机5/5 PASS-AUTO(过压VoltgTooHi/ModSts切回/扫档/长按不响应x2)。
