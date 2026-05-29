# Mini Cheetah SPI-to-CAN Bridge Firmware

这是 MIT Mini Cheetah 四足机器人腿部 SPI 转 CAN 模块的固件源码。模块运行在 mbed/STM32 环境中，上位机通过 SPI 下发两条腿的期望关节控制量，模块把这些控制量打包为 CAN 指令发送给电机，并把电机状态通过 SPI 返回给上位机。

当前工程主要文件：

- `main.cpp`：主逻辑，包含 SPI 从机收发、CAN 打包发送、CAN 回复解析、使能/失能、急停、软限位。
- `leg_message.h`：定义 SPI 上行/下行数据结构，以及关节状态/控制结构。
- `math_ops.cpp` / `math_ops.h`：浮点限幅、浮点与整数编码转换等工具函数。
- `mbed-dev.lib`：mbed 依赖引用。

## 硬件和通信拓扑

模块连接两路 CAN，总线速率均为 1 Mbps：

- `can1(PB_12, PB_13, 1000000)`：一条腿。
- `can2(PB_8, PB_9, 1000000)`：另一条腿。

每条腿 3 个电机，模块向电机下发命令时使用 CAN 标准帧 ID：

- `0x1`：abad 电机。
- `0x2`：hip 电机。
- `0x3`：knee 电机。

两条 CAN 总线上的电机命令 ID 相同，因为它们处于不同物理总线。代码中对应关系为：

- `a1_can.id = 0x1`，`h1_can.id = 0x2`，`k1_can.id = 0x3`。
- `a2_can.id = 0x1`，`h2_can.id = 0x2`，`k2_can.id = 0x3`。

电机回复帧的 CAN ID 是 `0x0`。模块用 CAN filter 接收 `CAN_ID=0x0`，然后从回复数据的 `data[0]` 中读取具体电机编号：

- `data[0] == 1`：abad 状态。
- `data[0] == 2`：hip 状态。
- `data[0] == 3`：knee 状态。

因此，`CAN ID=0` 是回复 ID，不是下发命令 ID。

## SPI 数据结构

SPI 接收上位机命令结构 `spi_command_t`，共 132 字节，即 66 个 16-bit word：

- `q_des_abad[2]`、`q_des_hip[2]`、`q_des_knee[2]`：两条腿三个关节的位置期望。
- `qd_des_abad[2]`、`qd_des_hip[2]`、`qd_des_knee[2]`：速度期望。
- `kp_abad[2]`、`kp_hip[2]`、`kp_knee[2]`：位置增益。
- `kd_abad[2]`、`kd_hip[2]`、`kd_knee[2]`：速度增益。
- `tau_abad_ff[2]`、`tau_hip_ff[2]`、`tau_knee_ff[2]`：前馈力矩。
- `flags[2]`：控制标志位。当前实际使用 `flags[0] bit0` 作为使能位。
- `checksum`：XOR 校验。

SPI 返回上位机状态结构 `spi_data_t`，共 60 字节，即 30 个 16-bit word：

- `q_abad[2]`、`q_hip[2]`、`q_knee[2]`：两条腿三个关节的位置反馈。
- `qd_abad[2]`、`qd_hip[2]`、`qd_knee[2]`：速度反馈。
- `flags[2]`：模块状态、软限位、急停、校验异常等反馈标志。
- `checksum`：XOR 校验。

`rx_buff` 和 `tx_buff` 都定义为 66 个 `uint16_t`。实际返回数据只填 `DATA_LEN=30` 个 word，其余不会作为有效 `spi_data_t` 使用。

## CAN 普通控制帧格式

普通控制帧由 `pack_cmd()` 生成，长度 8 字节：

- 16 bit 位置命令，范围 `[-12.5, 12.5]`。
- 12 bit 速度命令，范围 `[-65, 65]`。
- 12 bit `kp`，范围 `[0, 500]`。
- 12 bit `kd`，范围 `[0, 5]`。
- 12 bit 前馈力矩，范围 `[-18, 18]`。

打包前会用 `fminf/fmaxf` 限幅。浮点转整数使用 `float_to_uint()`，本质是线性映射并强制转换为整数。

8 字节布局：

```text
data[0] = p_int[15:8]
data[1] = p_int[7:0]
data[2] = v_int[11:4]
data[3] = v_int[3:0] + kp_int[11:8]
data[4] = kp_int[7:0]
data[5] = kd_int[11:4]
data[6] = kd_int[3:0] + t_int[11:8]
data[7] = t_int[7:0]
```

电机回复由 `unpack_reply()` 解析。回复数据中包含电机编号、位置、速度、电流/力矩估计，解析后写入 `l1_state` 或 `l2_state`。

## 上电初始化流程

`main()` 启动后先等待 1 秒，然后完成串口、急停输入、CAN filter、全局数据清零和 CAN 消息 ID 初始化。

重要顺序：

1. `pc.baud(921600)` 设置串口。
2. `pc.attach(&serial_isr)` 注册串口命令中断。
3. `estop.mode(PullUp)` 设置急停输入上拉。
4. `can1.filter(...)` 和 `can2.filter(...)` 只接收 `CAN ID=0` 的电机回复。
5. `memset(&tx_buff, 0, ...)`、`memset(&spi_data, 0, ...)`、`memset(&spi_command, 0, ...)` 清零。
6. 设置 6 个发送 CANMessage 的长度和 ID。
7. 调用 `pack_cmd()` 生成 6 个初始普通控制帧。
8. 调用 `WriteAll()` 发送。
9. 等待 SPI CS 不为低电平后调用 `init_spi()`，注册 `cs.fall(&spi_isr)`。
10. 进入 `while(1)`，不断读取 CAN 回复。

上电后即使上位机还没开机，代码也会在初始化阶段执行一次 `WriteAll()`。如果 CAN 分析仪只接在一条 CAN 总线上，会看到 3 帧：

```text
ID 0x01: 7F FF 7F F0 00 00 07 FF
ID 0x02: 7F FF 7F F0 00 00 07 FF
ID 0x03: 7F FF 7F F0 00 00 07 FF
```

这不是使能帧，而是所有控制量为 0 时打包出来的普通控制帧：

- `p_des = 0`
- `v_des = 0`
- `kp = 0`
- `kd = 0`
- `t_ff = 0`

代码实际上会向 `can1` 和 `can2` 都发，所以总线合计是 6 帧；单条腿总线上看到 3 帧。

## SPI 从机逻辑

`init_spi()` 创建 `SPISlave(PA_7, PA_6, PA_5, PA_4)`：

- SPI 数据宽度：16 bit。
- SPI mode：0。
- 频率：12 MHz。
- 默认回复：`0x0`。
- 片选下降沿触发 `spi_isr()`。

`main()` 中有一个保护逻辑：如果上电时 CS 正好为低，会等待 CS 回到高电平再初始化 SPI。注释说明原因是 SPI 在 CS 被拉低时启用会工作异常。

`spi_isr()` 的主要逻辑：

1. 先把 `tx_buff[0]` 写入 `SPI1->DR`，准备回传上一周期的状态。
2. 当 CS 保持低电平时，循环读取 SPI 收到的 16-bit word。
3. 每收到一个 word 就写入 `rx_buff[bytecount]`。
4. 如果还有待发送 word，就把对应 `tx_buff[bytecount]` 写入 SPI 数据寄存器。
5. CS 拉高后，计算接收 buffer 的 XOR checksum。
6. 把 `rx_buff` 复制进 `spi_command`。
7. 如果校验不匹配，将 `spi_data.flags[1]` 设为 `0xdead`。
8. 调用 `control()` 更新控制状态。
9. 无条件调用 `PackAll()` 和 `WriteAll()`，把当前控制命令发到 CAN。

需要注意：checksum 不匹配时，代码只是设置了返回给上位机的错误标志，并没有丢弃这帧 SPI 命令，也没有阻止 `control()` 和 `WriteAll()` 执行。因此错误 SPI 帧仍可能影响 CAN 输出。

## 使能和失能逻辑

当前实际使用的使能位是：

```cpp
spi_command.flags[0] & 0x1
```

代码里定义的 `ENABLE_CMD = 0xFFFF` 和 `DISABLE_CMD = 0x1F1F` 没有被使用。

模块内部用全局变量 `enabled` 记录本地认为的电机模式状态。初始值为 0。

### 使能触发条件

当上位机下发的 `flags[0] bit0` 为 1，且本地 `enabled == 0` 时触发：

```cpp
if (((spi_command.flags[0] & 0x1) == 1) && (enabled == 0))
```

触发后：

1. 立即把 `enabled` 设置为 1。
2. 给 6 个电机分别写入 Enter Motor Mode 特殊帧。
3. 直接调用 `can.write()` 发送这些特殊帧。
4. 打印 `e`。
5. 从 `control()` 返回。

Enter Motor Mode 数据内容：

```text
FF FF FF FF FF FF FF FC
```

使能分支中的发送顺序是：

```text
can1 ID 0x1
can2 ID 0x1
can1 ID 0x3
can2 ID 0x3
can1 ID 0x2
can2 ID 0x2
```

即每条腿内的顺序是 `1 -> 3 -> 2`，不是 `1 -> 2 -> 3`。

### 失能触发条件

当上位机下发的 `flags[0] bit0` 为 0，且本地 `enabled == 1` 时触发：

```cpp
else if (((spi_command.flags[0] & 0x1) == 0) && (enabled == 1))
```

触发后：

1. 立即把 `enabled` 设置为 0。
2. 给 6 个电机分别写入 Exit Motor Mode 特殊帧。
3. 直接调用 `can.write()` 发送这些特殊帧。
4. 打印 `x`。
5. 从 `control()` 返回。

Exit Motor Mode 数据内容：

```text
FF FF FF FF FF FF FF FD
```

失能分支中的发送顺序是：

```text
can1 ID 0x1
can2 ID 0x1
can1 ID 0x2
can2 ID 0x2
can1 ID 0x3
can2 ID 0x3
```

即每条腿内的顺序是 `1 -> 2 -> 3`。

### 使能/失能发送次数

从 SPI 控制路径看，每次状态跳变只发送一次特殊模式帧：

- 未使能到使能：每个电机 1 帧 Enter Motor Mode，总计 6 帧。
- 已使能到失能：每个电机 1 帧 Exit Motor Mode，总计 6 帧。

如果上位机一直保持 `flags[0] bit0 = 1`，模块不会重复发送使能帧；如果一直保持 0，也不会重复发送失能帧。只有本地 `enabled` 状态和上位机使能位发生边沿关系时才发送。

但 `spi_isr()` 在 `control()` 返回后仍然会无条件执行：

```cpp
PackAll();
WriteAll();
```

所以一次使能或失能跳变时，模式特殊帧之后会立刻再发一轮普通控制帧。那一轮不是使能/失能重发，而是普通 `p/v/kp/kd/t_ff` 控制命令。

### 不会基于电机回复确认模式状态

电机回复帧 `CAN ID=0` 只被解析为位置、速度和电流/力矩反馈。代码没有根据电机回复判断 Enter/Exit Motor Mode 是否成功。

因此，一旦调用 `can.write()`，代码就认为模式状态已经改变。即使 CAN 帧没有真正发出去、电机没有收到、或电机没有进入模式，`enabled` 也已经被改掉。

## 急停逻辑

急停输入是 `estop(PB_15)`，并配置为 `PullUp`。在 `control()` 中：

- `estop == 0`：认为急停触发。
- `estop != 0`：正常。

急停触发时：

1. 清零 `l1_control` 和 `l2_control`。
2. 设置 `spi_data.flags[0] = 0xdead`。
3. 设置 `spi_data.flags[1] = 0xdead`。
4. 点亮 LED。
5. 后续 `PackAll()` 会把清零后的控制量打包为普通控制帧。

急停不会主动发送 Exit Motor Mode 特殊帧，也不会把 `enabled` 改成 0。也就是说，急停路径更像是把普通控制命令清零并向上位机报告错误，而不是执行电机失能流程。

## 正常控制逻辑

当没有发生使能/失能边沿，且急停未触发时，`control()` 会：

1. 把 `l1_state`、`l2_state` 中的电机反馈填入 `spi_data`，准备下次 SPI 回给上位机。
2. 清零 `l1_control`、`l2_control`。
3. 从 `spi_command` 拷贝两条腿三关节的 `q_des`、`qd_des`、`kp`、`kd`、`tau_ff`。
4. 清零 `spi_data.flags[0]` 和 `spi_data.flags[1]`。
5. 执行 abad 和 hip 的软限位保护。
6. 计算 `spi_data.checksum`。
7. 把 `spi_data` 写入 `tx_buff`，供下一次 SPI 事务返回。

正常控制路径中，knee 的软限位检查被注释掉了：

```cpp
// spi_data.flags[0] |= (softstop_joint(l1_state.k, &l1_control.k, K_LIM_P, K_LIM_N))<<2;
// spi_data.flags[1] |= (softstop_joint(l2_state.k, &l2_control.k, K_LIM_P, K_LIM_N))<<2;
```

因此当前只有 abad 和 hip 启用了软限位反馈/保护。

## 软限位逻辑

`softstop_joint()` 接收当前关节状态、控制命令和正负限位。

如果位置超过正限位：

- `v_des = 0`
- `kp = 0`
- `kd = KD_SOFTSTOP`
- `t_ff += KP_SOFTSTOP * (limit_p - state.p)`
- 返回 1

如果位置低于负限位：

- `v_des = 0`
- `kp = 0`
- `kd = KD_SOFTSTOP`
- `t_ff += KP_SOFTSTOP * (limit_n - state.p)`
- 返回 1

如果没有越界，返回 0。

返回值会被 OR 到 `spi_data.flags` 中，用于通知上位机哪个关节触发软限位。

当前限位常量：

- abad：`[-1.5, 1.5]`
- hip：`[-5.0, 5.0]`
- knee：`[7.7, 0.2]` 相关检查当前注释掉。

## CAN 发送逻辑

`PackAll()` 把当前 `l1_control` 和 `l2_control` 打包进 6 个全局 CANMessage：

- `a1_can`
- `a2_can`
- `h1_can`
- `h2_can`
- `k1_can`
- `k2_can`

`WriteAll()` 负责发送：

```text
can1 a1
can2 a2
can1 h1
can2 h2
can1 k1
can2 k2
```

每帧之间 `wait(.00002)`，即约 20 us。

所有 `can.write(...)` 的返回值都被忽略。mbed 的 `CAN::write()` 成功通常返回 1，失败返回 0，但本代码没有检查返回值，没有错误统计，没有软件重发，也没有发送完成确认。

硬件 CAN 控制器可能会对已经进入发送邮箱的帧做总线层自动重发；但如果 `can.write()` 因邮箱满等原因失败，软件不会补发，这帧就被静默丢弃。

代码中没有软件发送队列，所以不存在按上位机历史命令严格 FIFO 发送的机制。每次 SPI 到来时，新的控制量覆盖全局 CANMessage；能写进硬件邮箱的帧就进入硬件发送流程，写不进去就丢失。

## 上位机离线时的行为

如果上位机完全不产生 SPI CS 下降沿和 SPI 时钟，模块不会周期性发送普通控制命令。代码里存在 `sendCMD()`，但没有启用：

```cpp
// loop.attach(&sendCMD, .001);
```

因此，除初始化阶段外，正常情况下 CAN 发送由 SPI 事务触发。

上位机离线时仍会出现一次上电初始化发送，因为 `main()` 中在启用 SPI 前就执行了：

```cpp
pack_cmd(...);
WriteAll();
```

如果约 30 秒后又看到同样 3 帧，源码中没有 30 秒定时器。更可能的原因是：

- MCU 复位后重新执行初始化，再发一次初始普通控制帧。
- SPI CS 脚产生了误下降沿，触发 `spi_isr()`，中断末尾的 `WriteAll()` 又发了一轮普通控制帧。

一个值得注意的硬件/代码点是：`estop` 设置了 `PullUp`，但 `cs(PA_4)` 没有显式设置上下拉。如果 UP Board 未上电时 CS 线悬空或状态不稳定，可能误触发 `cs.fall(&spi_isr)`。

区分方式：

- 如果 30 秒时串口重新打印 `SPIne` 和 `done`，说明 MCU 复位了。
- 如果没有重新打印启动信息，但出现 CAN 三帧，说明更可能是 CS 误触发。

## 串口命令逻辑

`serial_isr()` 处理串口输入，主要用于调试：

- `Esc`：调用 `ExitMotorMode()` 填充 6 个消息，设置 `enabled = 0`，最后 `WriteAll()`。
- `m`：调用 `EnterMotorMode()` 填充 6 个消息，等待 0.5 秒，设置 `enabled = 1`，最后 `WriteAll()`。
- `s`：设置 `is_standing = 1`，但相关 stand 逻辑没有实际展开。
- `z`：调用 `Zero()`。

`Zero()` 会把目标消息设为：

```text
FF FF FF FF FF FF FF FE
```

不过 `Zero()` 内部会调用 `WriteAll()`，而 `serial_isr()` 在 switch 之后也会再调用一次 `WriteAll()`。因此串口 `z` 路径可能产生多轮发送。

串口路径和 SPI 上位机路径是两套入口。实际机器人运行时通常关注 SPI 路径。

## CAN 接收逻辑

代码定义了 `rxISR1()` 和 `rxISR2()`，但在 `main()` 中 attach 已经注释掉：

```cpp
// can1.attach(&rxISR1);
// can2.attach(&rxISR2);
```

当前实际接收是在主循环里轮询：

```cpp
while (1) {
    counter++;
    can2.read(rxMsg2);
    unpack_reply(rxMsg2, &l2_state);
    can1.read(rxMsg1);
    unpack_reply(rxMsg1, &l1_state);
    wait_us(10);
}
```

这里没有检查 `can.read()` 是否成功。如果没有新消息，`rxMsg1/rxMsg2` 可能保留旧值或未更新，但仍会被 `unpack_reply()` 解析。因此状态反馈有可能重复使用旧帧。

## 校验逻辑

`xor_checksum()` 对 `uint32_t` 数组做 XOR。

SPI 接收命令时：

```cpp
uint32_t calc_checksum = xor_checksum((uint32_t*)rx_buff, 32);
```

`spi_command_t` 是 132 字节，共 33 个 `uint32_t`。这里对前 32 个 `uint32_t` 做 XOR，刚好排除最后的 `checksum` 字段。

SPI 返回数据时：

```cpp
spi_data.checksum = xor_checksum((uint32_t*)&spi_data, 14);
```

`spi_data_t` 是 60 字节，共 15 个 `uint32_t`。这里对前 14 个 `uint32_t` 做 XOR，排除最后的 `checksum` 字段。

设计意图是合理的：校验不包含 checksum 自身。但当前接收校验失败时并不阻止控制命令执行。

## 已知风险和行为特征

- 上电会主动发一轮普通控制帧，即使上位机还没开机。
- 使能/失能只由 `flags[0] bit0` 的边沿触发，不会周期性重发。
- 使能/失能没有等待或检查电机确认。
- CAN 写失败不会软件重发。
- 没有软件 CAN 发送队列，也没有严格 FIFO 保证。
- SPI checksum 错误只设置返回 flag，不阻止 CAN 输出。
- 急停不会发送 Exit Motor Mode，只会清零普通控制命令并报告 `0xdead`。
- `cs(PA_4)` 未显式设置上下拉，上位机未上电时可能因 CS 不稳定触发 SPI 中断。
- CAN 接收轮询不检查 `read()` 返回值，可能重复解析旧消息。
- `sendCMD()` 和 CAN 接收中断函数存在但未启用，实际 CAN 发送主要由初始化和 SPI 中断触发。

## 当前 GitHub 仓库

本工程已推送到公开仓库：

```text
https://github.com/Ijxuan/mini-cheetah-spi-can
```

本地远程仓库使用 GitHub SSH 443 端口：

```text
ssh://git@ssh.github.com:443/Ijxuan/mini-cheetah-spi-can.git
```
