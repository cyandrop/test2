# 电控组第二次培训作业

- 姓名：吴琰栋
- 学号：20253000548

---

## 一、任务目标

使用 **CLion + STM32CubeMX** 完成三个 LED 的流水灯控制：三个 LED 依次点亮，每个保持 500ms 后切换到下一个，循环往复。

在完成基础功能的前提下，进一步对比并实现了两种时间控制方式：

- **方案一**：使用 `HAL_Delay()` 的**阻塞式延时**
- **方案二**：使用 `HAL_GetTick()` 时间戳的**非阻塞延时**

## 二、软硬件环境

| 项目    | 说明                                   |
| ----- | ------------------------------------ |
| 主控芯片  | STM32F103C8T6（64KB Flash / 20KB RAM） |
| 调试器   | ST-Link V2（SWD 两线）                   |
| 配置工具  | STM32CubeMX                          |
| 开发环境  | CLion + CMake + arm-none-eabi-gcc    |
| 烧录调试  | OpenOCD 0.12                         |
| HAL 库 | STM32F1xx HAL Driver                 |

## 三、CubeMX 引脚配置

三个 LED 均配置为**推挽输出**，并通过 **User Label** 分别命名为 LED0、LED1、LED2：

| 引脚  | 信号类型        | GPIO Mode        | User Label |
| --- | ----------- | ---------------- | ---------- |
| PA1 | GPIO_Output | Output Push Pull | LED0       |
| PA2 | GPIO_Output | Output Push Pull | LED1       |
| PA3 | GPIO_Output | Output Push Pull | LED2       |

设置 User Label 后，CubeMX 会自动在 `Core/Inc/main.h` 中生成宏定义：

```c
#define LED0_Pin         GPIO_PIN_1
#define LED0_GPIO_Port   GPIOA
#define LED1_Pin         GPIO_PIN_2
#define LED1_GPIO_Port   GPIOA
#define LED2_Pin         GPIO_PIN_3
#define LED2_GPIO_Port   GPIOA
```

代码中全部通过上述宏操作引脚，不再直接书写 `GPIOA`、`GPIO_PIN_1` 这类裸参数，可读性和可维护性更好。

## 四、方案一：HAL_Delay 阻塞式延时

### 4.1 实现代码

最直观的写法，顺序执行：点亮一个灯 → 等待 500ms → 熄灭 → 点亮下一个。

```c
while (1)
{
    HAL_GPIO_WritePin(LED0_GPIO_Port, LED0_Pin|LED1_Pin|LED2_Pin, GPIO_PIN_RESET);
    HAL_GPIO_WritePin(LED0_GPIO_Port, LED0_Pin, GPIO_PIN_SET);
    HAL_Delay(500);

    HAL_GPIO_WritePin(LED0_GPIO_Port, LED0_Pin|LED1_Pin|LED2_Pin, GPIO_PIN_RESET);
    HAL_GPIO_WritePin(LED1_GPIO_Port, LED1_Pin, GPIO_PIN_SET);
    HAL_Delay(500);

    HAL_GPIO_WritePin(LED0_GPIO_Port, LED0_Pin|LED1_Pin|LED2_Pin, GPIO_PIN_RESET);
    HAL_GPIO_WritePin(LED2_GPIO_Port, LED2_Pin, GPIO_PIN_SET);
    HAL_Delay(500);
}
```

三个灯位于同一端口（GPIOA），因此“全部熄灭”可以用一次位或操作完成，只需写一次寄存器，属于原子操作。

### 4.2 存在的问题

`HAL_Delay()` 的内部实现（`stm32f1xx_hal.c`）是一个空转等待的循环：

```c
__weak void HAL_Delay(uint32_t Delay)
{
  uint32_t tickstart = HAL_GetTick();
  ...
  while ((HAL_GetTick() - tickstart) < wait)
  {
  }
}
```

在这 500ms 内，CPU 一直在 `while` 里空转，**无法执行任何其他逻辑**。一旦需要同时处理按键扫描、串口收发、传感器读取等任务，多个 `HAL_Delay()` 会互相阻塞，只能靠“拆分成更短的延时 + 全局状态机”勉强应付，代码会迅速变得混乱。

## 五、方案二：HAL_GetTick 非阻塞延时（本工程采用）

### 5.1 核心思路

不“等待”，而是“检查”：**记录上一次动作发生的时刻，每轮循环只判断当前时刻与上次时刻的差值是否达到 500ms**。条件成立才执行动作，不成立就立刻返回，主循环继续空转——而空转只需要几微秒。

```c
/* USER CODE BEGIN 2 */
uint8_t  step      = 0;   /* 当前点亮第几个灯：0→LED0，1→LED1，2→LED2 */
uint32_t last_time = 0;   /* 上一次换灯的时刻（ms） */
/* USER CODE END 2 */

while (1)
{
    if (HAL_GetTick() - last_time >= 500)
    {
        last_time += 500;

        /* 三个灯先全部熄灭 */
        HAL_GPIO_WritePin(LED0_GPIO_Port, LED0_Pin|LED1_Pin|LED2_Pin, GPIO_PIN_RESET);

        /* 只点亮当前这一个 */
        switch (step)
        {
            case 0: HAL_GPIO_WritePin(LED0_GPIO_Port, LED0_Pin, GPIO_PIN_SET); break;
            case 1: HAL_GPIO_WritePin(LED1_GPIO_Port, LED1_Pin, GPIO_PIN_SET); break;
            case 2: HAL_GPIO_WritePin(LED2_GPIO_Port, LED2_Pin, GPIO_PIN_SET); break;
        }

        step++;
        step = step % 3;      /* 0→1→2→0 循环 */
    }
}
```

### 5.2 关键细节一：为什么写 `last_time += 500`

`last_time` 记录的不是“上一次实际执行完的时刻”，而是“上一次应当执行的那个基准时刻”。

如果写成 `last_time = HAL_GetTick()`，那么从“判断成立”到“执行完毕”所消耗的几毫秒会被累加进下一次的基准，**误差逐轮累积**，运行时间越长，周期偏移越明显。

写成 `last_time += 500`，基准时刻始终落在 500 的整数倍上（0、500、1000……），误差不会累积，每轮自动对齐。

### 5.3 关键细节二：为什么用减法而不是加法

```c
if (HAL_GetTick() - last_time >= 500)   // 正确
if (HAL_GetTick() >= last_time + 500)   // 存在溢出隐患
```

`HAL_GetTick()` 返回 `uint32_t`，最大值约 42.9 亿毫秒，即 **49.7 天**后回绕归零。

- 减法写法：C 语言中无符号数相减会自动按 2³² 取模，差值在回绕点依然正确；
- 加法写法：`last_time + 500` 自身在回绕时也会溢出，导致条件判断失效，可能出现“灯卡住不动”。

比较使用 `>=` 而非 `==`，则可以容忍执行延迟（例如变成 501ms），并在落后时逐轮补齐，不会因为错过精确值而永久失效。

### 5.4 非阻塞带来的好处

改造后，主循环在两次换灯之间的约 500ms 内是完全空闲的，因此可以在同一个 `while (1)` 中直接并行安排其他不同周期的任务，互不干扰：

```c
/* 例如：每 10ms 扫描一次按键，每 1000ms 上报一次数据 */
static uint32_t last_key = 0, last_log = 0;

if (HAL_GetTick() - last_key >= 10)   { last_key += 10;   /* 按键扫描 */ }
if (HAL_GetTick() - last_log >= 1000) { last_log += 1000; /* 串口上报 */ }
```

这正是“非阻塞”的实际意义：**时间由软件自己管理，CPU 不再被某个延时独占。**

## 六、两种方案对比

| 对比项      | 方案一：`HAL_Delay` | 方案二：`HAL_GetTick`  |
| -------- | --------------- | ------------------ |
| 代码复杂度    | 最简单，顺序执行        | 稍复杂，需要一个基准变量和状态变量  |
| CPU 占用   | 100%（空转等待）      | 接近 0（每轮几微秒）        |
| 能否并行其他任务 | 不能              | 可以，任意添加不同周期任务      |
| 时间精度     | 受其他任务影响大        | 由硬件 SysTick 保证，1ms |
| 累积误差     | 无（每次重新计时）       | 无（`+=` 写法自动对齐）     |
| 适用场景     | 单任务、对实时性无要求     | 多任务、需要及时响应         |

结论：**方案一适合快速验证，方案二才是实际工程中的常规写法。**

## 七、工程结构

```
led_0/
├── Core/
│   ├── Inc/
│   │   ├── main.h              # 引脚宏定义（LED0_Pin / LED0_GPIO_Port 等）
│   │   ├── gpio.h
│   │   └── stm32f1xx_hal_conf.h
│   └── Src/
│       ├── main.c              # 主程序，流水灯逻辑位于 USER CODE BEGIN WHILE 区域
│       ├── gpio.c              # GPIO 初始化
│       ├── stm32f1xx_it.c      # 中断服务函数
│       └── system_stm32f1xx.c
├── Drivers/                    # HAL 库与 CMSIS
├── led_0.ioc                   # CubeMX 工程文件
├── STM32F103C8TX_FLASH.ld      # 链接脚本
├── CMakeLists.txt
└── README.md
```

## 八、编译与烧录

1. 用 CLion 打开本工程目录，等待 CMake 加载完成
2. `Ctrl + F9` 编译，控制台出现 `[Finished]` 即为通过
3. 连接 ST-Link（SWDIO→PA13、SWCLK→PA14、GND、3.3V）
4. `Shift + F9` 下载并进入调试，程序开始运行

编译占用情况：

```
RAM:   2656 B / 20 KB   (12.97%)
FLASH: 4880 B / 64 KB   ( 7.45%)
```

## 九、运行效果

三个 LED 按 LED0 → LED1 → LED2 的顺序依次点亮，每个保持 500ms 后熄灭并切换到下一个，循环往复形成流水灯效果。

由于采用非阻塞延时，主循环在等待期间保持空闲，后续可在此基础上直接扩展按键、串口等功能，而不需要改动流水灯部分的逻辑。

演示录屏见仓库内视频文件。
