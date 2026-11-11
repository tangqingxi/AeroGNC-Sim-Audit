# 统一推力接口与 PX4 对照

DEV `3acba790adcaad8215eccb69f495b5f89d8a435b`；冻结架构基线不变。

| 指定 PX4 df-main 源码位置 | 观察到的数据流 | 本项目映射 |
|---|---|---|
| [mc_att_control_main.cpp:374](https://github.com/mengchaoheng/DuctedFanUAV-Autopilot/blob/2eb33667575619b02a2d9b52032df2fac7fb5291/src/modules/mc_att_control/mc_att_control_main.cpp#L374) | 姿态 setpoint 的 thrust_body 随 rates setpoint 传递 | Attitude P 原样传 thrust_n |
| [MulticopterRateControl.cpp:383](https://github.com/mengchaoheng/DuctedFanUAV-Autopilot/blob/2eb33667575619b02a2d9b52032df2fac7fb5291/src/modules/mc_rate_control/MulticopterRateControl.cpp#L383) | Rate 读取 rates setpoint 中的推力 | Rate 读取 rate reference.thrust_n |
| [MulticopterRateControl.cpp:477](https://github.com/mengchaoheng/DuctedFanUAV-Autopilot/blob/2eb33667575619b02a2d9b52032df2fac7fb5291/src/modules/mc_rate_control/MulticopterRateControl.cpp#L477) | 发布独立 thrust/torque，二者 timestamp_sample 来自同一角速度采样 | 两个控制 Topic 同一 t 发布 |
| [ControlAllocator.cpp:638](https://github.com/mengchaoheng/DuctedFanUAV-Autopilot/blob/2eb33667575619b02a2d9b52032df2fac7fb5291/src/modules/control_allocator/ControlAllocator.cpp#L638) | 独立订阅 torque/thrust | Allocation 不读取 Guidance 或姿态参考 |

此处只对照数据流；PX4 侧的归一化向量、可选电池缩放与分配策略未搬入本项目。
本项目总推力为 N 标量、FRD -Z，六舵面 SI PINV 与电机独立求解保持。
没有引入七维统一分配或新推力控制器，没有修改 PID/增益/动力学/执行器/Simulation/Bus 实现。
源码逐字比较与保护路径检查见 evidence/source-scope.json；PX4 文件 SHA 与行号见 evidence/px4-reference.json。
