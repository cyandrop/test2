# 电控组第二次培训作业

- 姓名：青
- 学号：请填写你的学号

---

## 一、任务目标

使用 **CLion + STM32CubeMX** 完成三个 LED 的流水灯控制：三个 LED 依次点亮、每次保持 500ms、循环往复。

在此基础之上，程序采用了**非阻塞延时**的方式实现时间控制，全程不使用 `HAL_Delay()`。

## 二、软硬件环境

| 项目 | 说明 |
| --- | --- |
| 主控芯片 | STM32F103C8T6（64KB Flash / 20KB RAM） |
| 调试器 | ST-Link V2（SWD 两线） |
| 配置工具 | STM32CubeMX |
| 开发环境 | CLion + CMake + arm-none-eabi-gcc |
| 烧录调试 | OpenOCD 0.12 |
| HAL 库 | STM32F1xx HAL Driver |

## 三、CubeMX 引脚配置

三个 LED 均配置为**推挽输出**，并通过 **User Label** 命名为 LED0、LED1、LED2：

| 引脚 | 信号类型 | GPIO Mode | User Label |
| --- | --- | --- | --- |
| PA1 | GPIO_Output | Output Push Pull | LED0 |
| PA2 | GPIO_Output | Output Push Pull | LED1 |
| PA3 | GPIO_Output | Output Push Pull | LED2 |

配置 User Label 后，CubeMX 会自动在 `Core/Inc/main.h` 中生成宏定义：

```c
#define LED0_Pin         GPIO_PIN_1
#define LED0_GPIO_Port   GPIOA
#define LED1_Pin         GPIO_PIN_2
#define LED1_GPIO_Port   GPIOA
#define LED2_Pin         GPIO_PIN_3
#define LED2_GPIO_Port   GPIOA
```

代码中全部通过上述宏操作引脚，不再直接书写 `GPIOA`、`GPIO_PIN_1` 这类裸参数。

## 四、程序实现思路

### 4.1 为什么不用 HAL_Delay

`HAL_Delay(500)` 的内部实现是一个空转等待的循环：

```c
while ((HAL_GetTick() - tickstart) < 500) { }
```

在这 500ms 内 CPU 一直在原地打转，无法执行任何其他逻辑（例如按键扫描、串口收发）。一旦后续要加入其他功能，两个 `HAL_Delay` 之间就会互相打架。

### 4.2 非阻塞延时的实现

核心思路：**记录上一次动作发生的时刻，每轮循环只判断"当前时刻 − 上次时刻"是否已达到 500ms**。

```c
uint32_t last_time = 0;
uint8_t  step      = 0;

while (1)
{
    if (HAL_GetTick() - last_time >= 500)
    {
        last_time += 500;

        HAL_GPIO_WritePin(LED0_GPIO_Port, LED0_Pin, GPIO_PIN_RESET);
        HAL_GPIO_WritePin(LED1_GPIO_Port, LED1_Pin, GPIO_PIN_RESET);
        HAL_GPIO_WritePin(LED2_GPIO_Port, LED2_Pin, GPIO_PIN_RESET);

        switch (step)
        {
            case 0:  HAL_GPIO_WritePin(LED0_GPIO_Port, LED0_Pin, GPIO_PIN_SET); break;
            case 1:  HAL_GPIO_WritePin(LED1_GPIO_Port, LED1_Pin, GPIO_PIN_SET); break;
            default: HAL_GPIO_WritePin(LED2_GPIO_Port, LED2_Pin, GPIO_PIN_SET); break;
        }

        step = (uint8_t)((step + 1) % 3);
    }
}
```

### 4.3 两个关键细节

**一、为什么写 `last_time += 500`，而不是 `last_time = HAL_GetTick()`**

`+= 500` 让基准时刻始终落在 500 的整数倍上（0、500、1000……），实际执行耗掉的几毫秒不会累积成误差；而每次直接赋值为当前时刻，就会把执行耗时一次次累加进去，长时间运行后会明显变慢。

**二、为什么用减法比较，而不是加法**

```c
if (HAL_GetTick() - last_time >= 500)   // 正确
if (HAL_GetTick() >= last_time + 500)   // 存在溢出隐患
```

`HAL_GetTick()` 返回 `uint32_t`，约 49.7 天回绕一次。无符号数相减会自动按 2³² 取模，差值始终正确；而加法写法在回绕点会算错。

### 4.4 非阻塞带来的好处

主循环中除判断语句外全部空闲，因此可以在同一个 `while (1)` 内并行安排多个不同周期的任务，互不干扰。本工程后续可在此基础上直接追加：

```c
/* 例如：每 10ms 扫描一次按键、每 1000ms 上报一次数据 */
if (HAL_GetTick() - last_key >= 10)   { last_key += 10;   /* 按键扫描 */ }
if (HAL_GetTick() - last_log >= 1000) { last_log += 1000; /* 串口上报 */ }
```

## 五、工程结构

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

## 六、编译与烧录

1. 用 CLion 打开本工程目录，等待 CMake 加载完成
2. `Ctrl + F9` 编译，控制台出现 `[Finished]` 即为通过
3. 连接 ST-Link（SWDIO→PA13、SWCLK→PA14、GND、3.3V）
4. `Shift + F9` 下载并进入调试，程序开始运行

## 七、运行效果

三个 LED 按 LED0 → LED1 → LED2 的顺序依次点亮，每个保持 500ms 后熄灭并切换到下一个，循环往复形成流水灯效果。

演示录屏见仓库内视频文件。
