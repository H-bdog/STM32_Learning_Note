## 5-1中断系统：

中断就可以不用if

###  一）NVIC分类（响应优先级/抢占优先级）

响应：等上一个完了再插队

抢占(中断嵌套）

### 二）优先级分组

- 共5组优先级，每组之间的差别在于抢占优先级与响应优先级的个数不同
- 优先顺序（数值小的优先）

### 三）EXTI外部中断

- 触发方式：电平信号中断：上升、下降、双边、或者软件
- 支持所有的GPIO口，但数值相同的Pin不能同时触发（PinA1、PinB1）
- 触发的响应方式：
    1. 中断响应：触发中断函数
    2. 事件响应：不触发函数，而是执行某个命令

### 四）中断的设置

1. 开启时钟RCC：

    ```C
    //开启GPIO,AFIO的RCC时钟（EXTI无时钟，NVIC位于内核不需配置）
    {
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB,ENABLE);//GPIO
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_AFIO,ENABLE);//AFIO
    }
    //标准格式：
    /*
    RCC_APB2PeriphClockCmd(__,__)
    */
    ```

    2. 初始化GPIOx：

        ```C
        
        {
        GPIO_InitTypeDef GPIO_InitStructure;//定义GPIO寄存器结构体
        	GPIO_InitStructure.GPIO_Mode=GPIO_Mode_IPU;//定义引脚IO模式
        	GPIO_InitStructure.GPIO_Pin=GPIO_Pin_14;//选择引脚
        	GPIO_InitStructure.GPIO_Speed=GPIO_Speed_50MHz;//传输速度
        	GPIO_Init(GPIOB,&GPIO_InitStructure);//初始化
        }
        ```

        3. 让AFIO选择接入引脚：

            ```C
            {
            GPIO_EXTILineConfig(GPIO_PortSourceGPIOB,GPIO_PinSource14);
            }
            ```

        4. EXIT选择AFIO通道：

            ```c
            EXTI_InitStructure.EXTI_Line=EXTI_Line14;//线路选择
            EXTI_InitStructure.EXTI_LineCmd=ENABLE;//开启与关闭
            EXTI_InitStructure.EXTI_Mode=EXTI_Mode_Interrupt;//EXTI模式选择（事件）
            EXTI_InitStructure.EXTI_Trigger=EXTI_Trigger_Falling;//触发方式（下降沿）
            ```

        5. NVIC定义优先级

            ```c
             NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);//分组方式（Tips：每个芯片只能有一种分组模式）	
            	NVIC_InitTypeDef  NVIC_InitStructure;//设置NVIC结构体
            	//以下为结构体的定义
            NVIC_InitStructure.NVIC_IRQChannel=EXTI15_10_IRQn;//通道选择
            	NVIC_InitStructure.NVIC_IRQChannelCmd=ENABLE;
            	NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority=1;//响应优先级代号
            	NVIC_InitStructure.NVIC_IRQChannelSubPriority=1;//抢占优先级代号
            	NVIC_Init(&NVIC_InitStructure);//初始化
            ```

        6. 修改中断函数：

        在启动文件中找到中断函数（以IRQHandler结尾）

```c
void EXTI15_10_IRQHandler(void){
    if(EXTI_GetITStatus(EXTI_Line14==SET)){
        ……/*
        解释一下：
        因为EXTI15_10_IRQHandler这个函数包含了从10-15的各个通道，要先判断是否是通道14触发中断，再接着执行操作
        而EXTI_GetITStatus（）是用来获取通道状态的，可能为SET或RESET
        */
        EXTI_ClearITpendingBit(EXTI_Line14)//清除中断标志位，否则程序会一直卡在这里
    }
    
}
```

- 注意：前五步写在一个初始化函数中void Interrupt_Init(void)，然后在头文件中声明

    而第六步只是重新定义一下早已写入库函数中的中断函数





----



## 5-2旋转编码器计次

### 一）程序部分代码

#### 硬件配置函数（Encoder.h）

```c

int16_t Encoder_Count;

void Encoder_Init(void)
{

	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB,ENABLE);
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_AFIO,ENABLE);//第一步
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode=GPIO_Mode_IPU;
	GPIO_InitStructure.GPIO_Pin=GPIO_Pin_0 | GPIO_Pin_1;
	GPIO_InitStructure.GPIO_Speed=GPIO_Speed_50MHz;
	GPIO_Init(GPIOB,&GPIO_InitStructure);//第二步
	
	GPIO_EXTILineConfig(GPIO_PortSourceGPIOB,GPIO_PinSource0);//第三步
	GPIO_EXTILineConfig(GPIO_PortSourceGPIOB,GPIO_PinSource1);
	
	EXTI_InitTypeDef EXTI_InitStructure;
	EXTI_InitStructure.EXTI_Line=EXTI_Line0 | EXTI_Line1;
	EXTI_InitStructure.EXTI_LineCmd=ENABLE;
	EXTI_InitStructure.EXTI_Mode=EXTI_Mode_Interrupt;
	EXTI_InitStructure.EXTI_Trigger=EXTI_Trigger_Falling;
	EXTI_Init(&EXTI_InitStructure);//第四步
	
	 NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);//分组方式（Tips：每个芯片只能有一种分组模式）
	
	NVIC_InitTypeDef NVIC_InitStructure;
	NVIC_InitStructure.NVIC_IRQChannel=EXTI0_IRQn;//通道选择
	NVIC_InitStructure.NVIC_IRQChannelCmd=ENABLE;
	NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority=1;//代号
	NVIC_InitStructure.NVIC_IRQChannelSubPriority=1;//代号
	NVIC_Init(&NVIC_InitStructure);//第五步
	
	NVIC_InitStructure.NVIC_IRQChannel=EXTI1_IRQn;//通道选择
	NVIC_InitStructure.NVIC_IRQChannelCmd=ENABLE;
	NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority=1;//代号
	NVIC_InitStructure.NVIC_IRQChannelSubPriority=2;//代号
	NVIC_Init(&NVIC_InitStructure);//第五步

};
void EXTI0_IRQHandler(void)
{
	if (EXTI_GetITStatus(EXTI_Line0) == SET)//（严谨写法）判断程序是否真的进入了中断函数
	{
		if(GPIO_ReadInputDataBit(GPIOB,GPIO_Pin_1)==0)
		{
			Encoder_Count++;
		};
			
		EXTI_ClearITPendingBit(EXTI_Line0);//清除断点
	};	
	
};

void EXTI1_IRQHandler(void)
{
	if (EXTI_GetITStatus(EXTI_Line1) == SET)//（严谨写法）判断程序是否真的进入了中断函数
	{
			if(GPIO_ReadInputDataBit(GPIOB,GPIO_Pin_0)==0)
		{
			Encoder_Count--;
		};
		
		EXTI_ClearITPendingBit(EXTI_Line1);//清除断点
	};	
};

int16_t Encoder_Count_Get(void)
{
	int16_t Temp;
	Temp=Encoder_Count;
	Encoder_Count=0;
	return Temp;
};


```

#### 主函数（main.c）

```C
#include "stm32f10x.h"   
#include "Delay.h"
#include "OLED.h"
#include "CountSensor.h"
#include "Encoder.h"
// Device header

int16_t Num;
int main(void)
{
	Encoder_Init();
	OLED_Init();
	OLED_ShowString(1,3,"HZY love ZY");
	OLED_ShowString(2,1,"Count:");
	while(1)
	{
	Num+=Encoder_Count_Get();
	OLED_ShowSignedNum(2,7,Num,6);
		}
	}
```

### 二）思想

#### 1. 对于旋转编码器原理：

​					旋转时，A或B产生下降沿（故应该使用下降沿触发）的同时，B或A为低电平。为保证旋转结束再触发中断函数，应该使用if嵌套语句

```c
	if (EXTI_GetITStatus(EXTI_Line1) == SET)
	{
			if(GPIO_ReadInputDataBit(GPIOB,GPIO_Pin_0)==0)//if嵌套
		{
			Encoder_Count--;
		};
		
		EXTI_ClearITPendingBit(EXTI_Line1);
	};	
```

#### 2.计次：

​		由于旋转编码器不稳定的机械结构，如果直接以中断函数的Encoder_Count作为计次Count，数据会跳动。因此运用一种思想——在Get函数获取的同时，清零Encoder_Count，并在主函数定义新变量Num，将中断函数的Encoder_Count视为对Num的改变。就会使得数据稳定。

```C
/*清零*/
int16_t Encoder_Count_Get(void)
{
	int16_t Temp;
	Temp=Encoder_Count;
	Encoder_Count=0;
	return Temp;
};
/*修改Num*/
Num+=Encoder_Count_Get();
OLED_ShowSignedNum(2,7,Num,6);
```
---
## 6-1定时器计时中断

### 一）基本结构：

1. 时钟模式（内部时钟、外部时钟）
2. 时基单元（ARR,PSC——控制时间频率）
3. TIM计时器的中断输出控制Config，选择打开哪种中端配置（一般都是用TIM_IT_Update，但我试了其他几个，比如COM1,COM2都可以用，只是Trigger好像用不了）
4. 对应的NVIC（TIM2对应的中断函数为TIM2_IRQHandler）

```C
/*初始化TIM中断的方法*/
void Timer_Init(void){
	RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM2,ENABLE);
	
	TIM_InternalClockConfig(TIM2);//TIM2 use Internal timer
	
	TIM_TimeBaseInitTypeDef TIM_TimeBaseInitStructue;
	TIM_TimeBaseInitStructue.TIM_ClockDivision=TIM_CKD_DIV1;
	TIM_TimeBaseInitStructue.TIM_CounterMode=TIM_CounterMode_Up;;
	TIM_TimeBaseInitStructue.TIM_Period=10000 -1;
	TIM_TimeBaseInitStructue.TIM_Prescaler=7200 -1;// these 2 value is for setting time rate, there is a function to calcuate.
	TIM_TimeBaseInitStructue.TIM_RepetitionCounter=0;
	TIM_TimeBaseInit(TIM2,&TIM_TimeBaseInitStructue);
	
	TIM_ITConfig(TIM2,TIM_IT_CC1|TIM_IT_Trigger|TIM_IT_CC4,ENABLE);//TIM中断配置
	/*开始配置计时器的中断函数*/
	NVIC_InitTypeDef NVIC_InitSructure;
	NVIC_InitSructure.NVIC_IRQChannel= TIM2_IRQn;//找到TIM2专用的中断通道，并填进去
	NVIC_InitSructure.NVIC_IRQChannelCmd=ENABLE;
	NVIC_InitSructure.NVIC_IRQChannelPreemptionPriority=2;
	NVIC_InitSructure.NVIC_IRQChannelSubPriority=1;
	NVIC_Init(&NVIC_InitSructure);
	TIM_Cmd(TIM2,ENABLE);
}
/*设置中断函数的方法*/
void TIM2_IRQHandler(void)//配置TIM的中断函数
{
	if(TIM_GetITStatus(TIM2,TIM_IT_CC1)==SET)	
	{
		Num++;
		TIM_ClearITPendingBit(TIM2,TIM_IT_CC1);//清除标志位——这个是必须的，否则将卡在中断这里
	}
}
/*
解决中断函数中变量无法跨文件使用的方法：
1. 使用extern+变量名		能告诉编译器去主动寻找其他文件中的该变量
2. 直接在需要调用中断函数的文件（如main）中定义中断函数
*/
```



## 9-1串口通信

#### 一）串口参数

1. 波特率（发送速度）
2. 起始位：低电平
3. 停止位：高电平
4. 数据位：低位数据先行
5. 校验位：无校验、奇校验（奇数个1）、偶校验（偶数个1）

#### 二）printf重定向

```C
/*此时在.c文件中*/

//在该.c文件头部
#include <stdio.h>

……
    
int fputc(int ch,FILE*f)//fputc是pringf的底层函数，因此通过更改fputc可以达到更改printf的作用
{
	Serial_SendByte(ch);
	return ch;
}

/*此时在主函数中*/
//方法一：利用printf一个个输出
printf("Num=%d",666);
//方法二：利用sprint实现字符串的整体输出
char String[100];
sprintf(String,"Num=%d",666);
Serial_SendString(String);
//方法三对sprint的封装操作：
 		/*此时在.c文件中*/
#include <stdarg.h>
void Serial_Printf(char *format,...)//...是你将要输出的
        {
            char String[100];
            va_list arg;
            va_start(arg,format);
            vsprintf(String,format,arg);
            va_end(arg);
            Serial_SendString(String);
        }
		/*此时在主函数中*/
Serial_Printf("Num=%d",66);
```

#### 三）发送汉字

方法一：

在原来的编码格式下，在魔术棒-C/C++-中加入“--no-multibyte-chars”，并且在串口助手处使用UTF8的发送方式



方法二：

更改编码格式GB2312
