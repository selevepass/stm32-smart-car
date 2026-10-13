# 基于 STM32F103 的智能循迹避障小车

STM32 单片机课程设计项目。做了一个两轮智能小车，能红外循迹、红外加超声波避障、蓝牙遥控，工作模式通过蓝牙串口指令实时切换。

用的是 ST 标准外设库（不是 HAL 库），Keil5 工程。

## 工作模式

用手机蓝牙串口助手发单个字符就能切换，不用重新烧录：

| 指令 | 模式 | 说明 |
|:---:|---|---|
| `0` | 前进 | 两轮正转 |
| `1` | 左转 | 差速左转 |
| `2` | 右转 | 差速右转 |
| `3` | 后退 | 两轮反转 |
| `4` | 停止 | 上电默认这个模式 |
| `5` | 循迹 | 4 路红外识别黑线，脱线自动纠偏 |
| `6` | 联合避障 | 红外 + 超声波一起用 |
| `7` | 超声波避障 | 只用 HC-SR04 测距 |

## 硬件

| 模块 | 型号 | 接口 | 引脚 |
|---|---|---|---|
| 主控 | STM32F103C8T6 | — | Cortex-M3，72 MHz |
| 电机驱动 | L298N / 双 H 桥 | GPIO | PB5 使能；PB6/PB7 接 M2，PB8/PB9 接 M1 |
| 循迹传感器 | 4 路红外阵列 | GPIO 输入 | PB13 / PB14 / PB15 / PA12，上拉，压线为低 |
| 避障传感器 | 2 路红外 | GPIO 输入 | PB10（左）、PA11（右），上拉，有障碍为低 |
| 超声波 | HC-SR04 | GPIO | PB11 = TRIG，PB12 = ECHO |
| 蓝牙 | HC-05 / HC-06 | USART1 | PA9 = TX，PA10 = RX，9600bps 8N1 |

电机是 **H 桥方向控制**，不是 PWM 调速。以 M1 为例：

| M1+ | M1− | 结果 |
|:---:|:---:|---|
| 1 | 0 | 正转 |
| 0 | 1 | 反转 |
| 0 | 0 | 停转 |

转向靠两侧轮子差速实现，不是靠舵机。

## 代码结构

```
Mcu_car/
├── User/                       应用层，主要看这里
│   ├── main.c                  初始化 + 主循环
│   ├── move.c / move.h         电机运动控制
│   ├── trackingTrack.c / .h    循迹，4 路红外 + 脱线判定
│   ├── InfraredSensor.c / .h   红外避障、红外与超声波联合避障
│   ├── ultrasonic.c / .h       HC-SR04 测距，TRIG/ECHO 时序
│   ├── btControl.c / .h        蓝牙指令解析，映射到电机动作
│   ├── delay.c / delay.h       微秒/毫秒延时
│   └── wait.c / wait.h         阻塞等待，配合运动时长
├── m_lib/                      底层支持库
│   ├── System/                 SysTick、USART、系统时钟
│   ├── STM32F10X_FWLIB/        ST 标准外设库（官方）
│   └── CORE/                   Cortex-M3 内核文件
├── RTE/                        Keil 运行环境配置
└── car.uvprojx                 Keil5 工程文件
```

主线很简单，`main()` 里初始化完就进主循环，四个模块轮流跑：

```c
void myInit(void) {
    Moto_Init();
    SysTick_Init(72);
    InfSensor_Init();
    Follow_InfSensor_Init();
    btControlInit();
    ultrasonic_init();
    delay_init();
    USART1_init();
    data = '4';               // 默认待机
}

int main(void) {
    myInit();
    while (1) {
        btControl();          // 响应 0–4
        InfraredSensor();     // 响应 6
        trackingTrack();      // 响应 5
        ultrasonic();         // 响应 7
    }
}
```

每个子函数开头都用 `if (data == 'x')` 判断自己要不要干活，不是自己的模式就直接返回，这样四个模块互不阻塞。`data` 这个全局变量由串口中断改写，所以发指令能即时生效。

## 怎么编译和烧录

1. Keil μVision5 打开 `Mcu_car/car.uvprojx`
2. `Options for Target → C/C++` 里确认 Include Paths 包含 `User`、`m_lib/System`、`m_lib/STM32F10X_FWLIB/inc`、`m_lib/CORE`
3. F7 编译
4. 用 ST-Link 或 J-Link 下载到 STM32F103C8T6
5. 手机连上 HC-05，发 `5` / `6` / `7` 切换模式

原理图和模块说明书在 `Doc/` 里。

## 几个注意点

- **超声波测距用的是软件延时**：TRIG 拉高 15μs 触发，然后等 ECHO 变高，在 ECHO 保持高电平期间循环计数，最后按声速换算距离（计数/2 × 3mm）。这种方式会一直占着 CPU，而且如果期间进中断，计时就不准了。
- **循迹是阈值式判定，不是 PID**：`allcheck()` 判断四路全白（完全脱线）就后退，最外侧触线就大角度转，中间两路小角度微调。优先级顺序是这个算法的关键。
- **`printf` 重定向到了串口**：`usart.c` 里实现了 `fputc()`，另外定义了 `_sys_exit()` 关掉半主机模式，不然会卡住。
- 每个功能模块都是独立的 `.c/.h` 一对，头文件只暴露初始化函数和主函数。

## 还能继续做的

1. **把电机改成 PWM 调速**：现在只有全速开关，把 TIM 的 PWM 接到使能脚就能无级调速
2. **超声波改用输入捕获**：回波宽度交给 TIM 输入捕获来测，精度更高，也不占 CPU
3. **把 `data` 的魔法字符改成状态枚举**：现在用 `'0'`–`'7'` 直接判断，可读性差，改成 `enum` + `switch` 更清楚
4. **上 FreeRTOS**：把四个模块拆成独立任务，用消息队列传蓝牙指令
5. **循迹改 PID**：用红外阵列的偏差量做闭环，比阈值式稳
6. **加 IWDG 看门狗**：防止主循环卡死
