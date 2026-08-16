---
title: STM32学习笔记（自用精简版）
published: 2026-07-11
pinned: true
description: 暑假准备数学建模笔记分享
image: ./images/cover2.avif
tags:
  - learning
  - note
category: learning
---
# 点灯

## 按键控制
将按键开关固定在面包板上，使用按键开关控制STM32F103C8T6最小系统板上的 PC13 指示灯，使得按下按钮时，指示灯亮，松开按钮时，指示灯灭（暂不按键消抖的问题）
>按键：上拉模式，平时为高电平，按下后为低电平
>灯：平时为高电平，低电平点亮

```c
while (1)
  {
    /* USER CODE END WHILE */
	if (HAL_GPIO_ReadPin(GPIOA,GPIO_PIN_9) ==GPIO_PIN_RESET) {
		HAL_GPIO_WritePin (GPIOC,GPIO_PIN_13,GPIO_PIN_RESET);
	}
	else
	{
		HAL_GPIO_WritePin(GPIOC,GPIO_PIN_13,GPIO_PIN_SET);
	}
    /* USER CODE BEGIN 3 */
  }
```

## 摇杆控制
使用双轴按键摇杆传感器Z轴控制STM32F103C8T6最小系统板上的PC13指示灯，使得按下摇杆时，指示灯状态改变，即由亮变暗、由暗变亮（需考虑按键消抖）。要求通过外 部中断来完成这一控制逻辑。（与前面题目要求不同，按下之后会自动回去，所以需要中断函数）
>Z作为按键，默认为高电平，按下时变为低电平
>每次按下修改状态，所以是TogglePin
>考虑消抖,在回调函数里先延时20ms，待信号 稳定后再读取按键电平；若此时按键确实处于按下状态，再翻转PC13电平。通过这种方式， 我们能有效区分按键是被按下还是释放，只在按下时进行状态切换

```C
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_PIN) {
	if (GPIO_PIN == GPIO_PIN_9) {
		HAL_Delay(10);
		if (HAL_GPIO_ReadPin(GPIOA,GPIO_PIN_9) == RESET) {
			HAL_GPIO_TogglePin(GPIOC,GPIO_PIN_13);
		}
	}
}
```

# TIM
- 预分频的系数是𝑃𝑆𝐶+1， 也就是把输入时钟频率除以𝑃𝑆𝐶+1，得到一个较低的频率。这个分频后的时钟送入计数器 CNT，作为CNT的计数速度。CNT就是一个“计数器”，根据定时器计数模式进行向上或向 下计数，然后产生溢出事件。重复计数器RCR可设置重复次数，达到RCR+1次溢出后触发 一次update 事件。
- 定时器中断就是利用update事件来触发的。这样我们就可以“自动”地每隔一段时间执 行一次代码
- 定时器update的触发周期：$T_{out} = \frac {(ARR+1)(PSC+1)(RCR+1)} {T_{clk}}$ 
- 函数
```C
//启用定时器并使其能中断
HAL_StatusTypeDef HAL_TIM_Base_Start_IT(TIM_HandleTypeDef *htim)
//停用并禁用中断
HAL_StatusTypeDef HAL_TIM_Base_Stop_IT(TIM_HandleTypeDef *htim)
//中断回调函数
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim) 
{ 
	if (htim == &htim1)
	 { 
		 //TODO 
	 } 
	 if (htim == &htim2) 
	 { 
		 //TODO } 
	 }
```
- 输出
- PWM 占空比 模拟模拟信号，假设时钟源频率是8MHz，设置预分频器PSC的值为799，自动重装寄存器ARR的值为 9，则计数器的溢出频率为1KHz，因此方波信号的周期为1ms。。若将CCR的值设置为4，则 可以得到周期为1ms，占空比为50%的方波信号（阶梯式图形）
```c
//启动指定通道PWM输出
HAL_StatusTypeDef HAL_TIM_PWM_Start(TIM_HandleTypeDef *htim, uint32_t Channel）
//关闭指定通道PWM输出
HAL_StatusTypeDef HAL_TIM_PWM_Stop(TIM_HandleTypeDef *htim, uint32_t Channel）
//启动指定通道互补PWM输出
HAL_StatusTypeDef HAL_TIMEx_PWMN_Start(TIM_HandleTypeDef *htim, uint32_t Channel)
//compare 设置的是CCR HANDLE (&htim1)
#define __HAL_TIM_SET_COMPARE(HANDLE, CHANNEL, COMPARE)
//AutoReload 设置的是ARR
#define __HAL_TIM_SET_AUTORELOAD(HANDLE, AUTORELOAD)
//预分频器值 Prescalar PSC
#define __HAL_TIM_SET_PRESCALER(HANDLE, PRESC)
```
## MyDelay
自制延迟函数MyDelay()，复现点灯实验
>配置出1ms的定时器中断（PSC=71，ARR=999，RCR=0），在定时器中断回调函数中对 currentMiliSeconds 自增，以维护“当前系统时间" 

```c
//系统时间经常需要修改
volatile uint32_t currentMiliSeconds = 0;

//中断
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTyoeDef *htim) {
	if (htim == &htim1) {
		currentMiliSeconds++;
	}
}

void MyDelay(uint32_t Delay) {
	uint32_t expireTime = currentMiliSeconds + Delay;
	while (currentMiliSeconds < expireTime);
}
```
## 摇杆控制灯
使用双轴按键摇杆传感器Z轴控制STM32F103C8T6最小系统板上的PC13指示灯， 使得按下摇杆时，指示灯状态改变，即由亮变暗、由暗变亮（需考虑按键消抖）。要求通过定 时器中断来完成这一控制逻辑
>前文中用delay消抖会发生延时
   用定时器每10秒update，则每10s检查一次按键电平
> PSC = 71 ARR = 9999 RCR = 0

```c
GPIO_PinState lastState, nowState;

void HAL_TIM_PeriodElapsedCallback(TIM_HandleTyoeDef *htim) {
	if (htim == &htim1) {
		nowState = HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_8);
		
		if (lastState == GPIO_PIN_SET && nowState == GPIO_PIN_RESET) {
			HEL_GPIO_TogglePin(GPIOC,GPIO_PIN_13);
		}
		lastState = nowState;
	}
}

int main(void) {
	HAL_TIM_Base_IT(&htim1);
	while(1) {
	
	}
}
```

## 呼吸灯 
呼吸灯的亮度变化通常呈现类似于正弦波的波形。请控制互补的双LED灯实现呼吸 灯效果，需通过PWM交替调节两颗LED的亮度，同时加入限流电阻保护电路。
- PWM信号调节LED亮度
- 亮度变化，用占空比表示$duty = 0.5sin(2pi* t) + 0.5 = \frac{CCR}{ARR+1}$ 
```C
int main(void) {
	HAL_TIM_PWM_Start(&htim1,TIM_CHANNEL_1);
	HAL_TIMEx_PWMN_Start(&htim1,TIM_CHANNEL_1);
	while(1) {
		//获取当前时间，毫秒转化为秒
		float t = HAL_GetTick() * 0.001；
		float duty = 0.5 * sin(2*3.14*t) + 0.5;
		TIM1->CCR1 = duty * (TIM1->ARR + 1);
	}
}
```
# ADC
模拟信号->数字信号
单片机常见的输入电压范围是0∼3.3V。如果用4位ADC，就相当于只能用0000b~1111b 共16个数，去表示0∼3.3V的范围，这样一份电压大约是0.22V
```c
//启动ADC校准过程的函数。它接受要校准的ADC的指针hadc。该函数只需在ADC初 始化完成后、开始第一次转换之前调用一次即可，不需要在每次转换前都调用
HAL_StatusTypeDef HAL_ADCEx_Calibration_Start(ADC_HandleTypeDef* hadc)
//ADC启动函数
HAL_StatusTypeDef HAL_ADC_Start(ADC_HandleTypeDef* hadc)
//中断模式ADC启动函数
HAL_StatusTypeDef HAL_ADC_Start_IT(ADC_HandleTypeDef* hadc)
//ADC停止函数
HAL_StatusTypeDef HAL_ADC_Stop(ADC_HandleTypeDef* hadc)
//中断模式ADC停止函数
HAL_StatusTypeDef HAL_ADC_Stop_IT(ADC_HandleTypeDef* hadc)
//阻塞模式等待ADC转换完成函数,通常可以直接视作“等待转换完成”的调用
HAL_StatusTypeDef HAL_ADC_PollForConversion(ADC_HandleTypeDef* hadc, uint32_t Timeout)
//获取ADC转换结果函数
uint32_t HAL_ADC_GetValue(ADC_HandleTypeDef* hadc)
//ADC中断回调函数
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef* hadc) {
	if (hadc == &hadc1) {
	//todo
	}
	if (hadc == &hadc2) {
	//todo
	}
}
```
在由摇杆的X轴和Y轴构成的平面上，判断当前位置为原点或向上、向下、向左、 向右四个方向的移动。在重定向printf后，每100ms输出一次位置判断结果
```c
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim) {
	if (htim == &htim1) {
		HAL_ADC_Start(&hadc1);
		HAL_ADC_PollForConversion(&hadc1,100);
		uint16_t adc_value0 = HAL_ADC_GetValue(&hadc1);
		
		HAL_ADC_Start(&hadc1);
		HAL_ADC_PollForConversion(&hadc1,100);
		uint16_t adc_value1 = HAL_ADC_GetValue(&hadc1);
		
		float x_voltage = adc_value0 / 4096.0f * 3.3f;
		float x_voltage = adc_value1 / 4096.0f * 3.3f;
		
		if (x_voltage < 1.0f) {
			printf("left\r\n");
		}
		else if (x_voltage > 2.5f) {
			printf("right\r\n");
		}
		else if (y_voltage < 1.0f) {
			printf("down\r\n");
		}
		else if (y_voltage > 2.5f) {
			printf("up\r\n");
		}
		else {
			printf("center\r\n");
		}
	}
}

int main(void) {
	HAL_ADCEx_Calibration_Start(&hadc1);
	HAL_TIM_Base_IT(&htim1);
	while(1) {
	 //主循环
	}
}
```
# UART通信
- 点对点通信 Rx 接受端 Tx 发送端 GND 
- 异步通信:发送端啥时候想发啥时候发，不需要等某一方的时钟信号来进行同步接受次 序后再发
```c
uint8_t RxData[10];
uint8_t TxData[10] = "hello";

//阻塞式
//发一条然后收一次，每个任务不超过1000ms，未果就继续往下进行
while(1) {
	HAL_UART_Transmit(&huart1,TxData,5,1000);
	HAL_UART_Receive(&huart1,RxData,5,1000);
	if (RxData[0] == '1') {
		HAL_GPIO_TogglePin(GPIOC,GPIO_PIN_13);
		RxData[0] = 0;
	}
}

//中断式
int main(void) {
	HAL_UART_Receive_IT(&huart1,RxData,5);
	while(1) {
		HAL_UART_Transmit_IT(&huart1,TxData,5);
		HAL_Delay(1000);
	}
}

void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart) {
	if (hurat == &huart1) {
		HAL_UART_Receive_IT(&huart1,RxData,5);
		if (RxData[0] == '1') 
		{
			HAL_GPIO_TogglePin(GPIOC,GPIO_PIN_13);
			RxData[0] = 0;
		}
	}
}

//阻塞式 
HAL_UART_Transmit(&huart1, TxData, 5, 1000); 
HAL_UART_Receive(&huart1, RxData, 5, 1000); 

//中断式（使能中断） 
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart) 
HAL_UART_Transmit_IT(&huart1, TxData, 5); 
HAL_UART_Receive_IT(&huart1, RxData, 5); 

//空闲中断（使能中断） 
void HAL_UARTEx_RxEventCallback(UART_HandleTypeDef *huart, uint16_t Size){} •HAL_UART_Transmit_IT(&huart1, TxData, 5); 
HAL_UARTEx_ReceiveToIdle_IT(&huart1, RxData, 5);  

//DMA中断（配置DMA且使能中断） 
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart){} 
HAL_UART_Transmit_DMA(&huart1, TxData, 5); • HAL_UART_Receive_DMA(&huart1, RxData, 5); 

//DMA空闲中断（配置DMA且使能中断） 
void HAL_UARTEx_RxEventCallback(UART_HandleTypeDef *huart, uint16_t Size){} HAL_UART_Transmit_DMA(&huart1, TxData, 5); 
HAL_UARTEx_ReceiveToIdle_DMA(&huart1, RxData, 5);
```

# DMA
一种能够在不占用CPU时间的情况下，直接将数据从外设（如串口、ADC）传输到内存，或从内存传输到外设的技术
DMA模式：由DMA控制器直接负责数据搬运，CPU几乎不参与，效率高
- 正常模式：适合单次发送，`HAL_ UART_Transmit_DMA` `HAL_ADC_Start_DMA`
- 循环模式：传完会自动从头再来，适合连续不断的数据流，`HAL_ADC_Start_DMA`
```c
//HAL_UART_Transmit_DMA存在设计缺陷，因此在使用时需要在CubeMX上将对应串口的全局中断打开
//DMA模式串口发送函数
HAL_StatusTypeDef HAL_UART_Transmit_DMA(UART_HandleTypeDef *huart, uint8_t *pData, uint16_t Size)
//DMA模式ADC启动函数
HAL_StatusTypeDef HAL_ADC_Start_DMA(ADC_HandleTypeDef* hadc, uint32_t* pData, uint32_t Length)
```
使用DMA采集摇杆的X轴和Y轴的ADC数值（范围0～4095），并每隔100ms通过串口DMA模式将采集结果发送到上位机。上位机可以使用SerialPlot或逐飞助手等工具对数据进行波形显示，从而直观观察摇杆的运动轨迹。
>ADC配置为多通道扫描模式，关闭连续和间断模式，DMA使用正常模式
>定时器中断一次ADC转换，采样后进入ADC的转换完成回调函数
```c
uint8_t buf[12];
uint16_t ADC_value[2];

void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim) {
	if (htim == &htim1) {
		HAL_ADC_Start_DMA(&hadc1,(uint32_t *)ADC_Value,2);  
	}
}

void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef* htim) {
	if (hadc  == &hadc1) {
		uint16_t len = sprintf(buf, "%d %d\r\n",ADC_Value[0],ADC_Value[1]);
		HAL_UART_Transmit_DMA(&HUART1,BUF,len);
	}
}

int main(void) {
	HAL_ADCEx_Calibration_Start(&hadc1);
	HAL_TIM_Base_Start_IF(&htim1);
	while(1) {
	}
}
```
# 项目一
通过双轴按键摇杆传感器控制无源蜂鸣器的音乐循环播放，实现相应功能：
1. X轴控制音乐速率：摇杆的X轴位置决定音乐播放的速率，当摇杆拨至上方、中央、下方时，分别为2倍速、原速和0.5倍速。速率的变化根据摇杆的当前位置实时调整
2. Y轴控制音调：摇杆的Y轴用于改变音调，当拨至上方、中央、下方时，音调分别为高 音、原音和低音，根据当前摇杆位置即时调整
3. Z轴控制音乐速率：摇杆Z轴用于切换音乐的播放状态。当按下并松开摇杆Z轴时，音 乐在暂停和播放之间切换
4. 严格的播放时间控制：每个音符的播放时间需要精确控制。例如，一个音符需要播放1 秒，如果在播放到0.2秒时暂停，那么当恢复播放时，应该从剩余的0.8秒继续播放，而不是重新播放该音符
```c

```
<script src="https://giscus.app/client.js"
        data-repo="ice-creamy405/blog"
        data-repo-id="R_kgDOTsZHBg"
        data-category="General"
        data-category-id="DIC_kwDOTsZHBs4DC7cK"
        data-mapping="title"
        data-strict="0"
        data-reactions-enabled="1"
        data-emit-metadata="1"
        data-input-position="bottom"
        data-theme="preferred_color_scheme"
        data-lang="zh-CN"
        data-loading="lazy"
        crossorigin="anonymous"
        async>
</script>