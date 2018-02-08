---
title: RH850内部RTC校准
date: 2018-02-07 19:09:05
tags: [RTC, RH850]
categories: 学习笔记
---
# RTC时钟校准
RH850内部RTCA模块使用32.768KHz的subClock作为时钟源。时钟源的偏差将导致计时出现偏差，所以需要检测时钟源的实际频率，然后做出相应的补偿，以达到计时精度要求。  

## 时钟测量
配置Clock模块可以直接从`JP0_3(CSCXFOUT)`引脚输出时钟，要输出的时钟类型通过`CKSC_AFOUTS_CTL`寄存器选择`SubSoc`，然后通过`SYSFOUTDIV`使能输出和设置输出的分频比。通过频率计直接测量输出的频率。  

## RTC配置

### 工作模式
RTC模块有两种工作模式：  
- 32.768-kHz模式，时钟固定为32.768kHz，计数器从0记到32767为1s。
- 指定频率模式，时钟可以不使用32.768kHz，需要手动设定计数器的翻转值。       
  
使用RTC模块内部的校准功能需要配置成32.768-kHz模式。这种模式下有两种校准方式可以选择：  
- 每20s补偿特定值, 可以校准的频率范围为32.76180000 to 32.77420000 kHz。
- 每60s补偿特定值, 可以校准的频率范围为32.76593333 to 32.77006667 kHz
后者支持的频率范围小，但是可校准精度是前者的3倍。

### 校正值计算
RTC校正的根本原理是每一定周期增加或减少1S的计数值。频率偏高，计时偏快时，增加计数值，拉长1S的时间；频率偏低，计时偏慢时，减少计数值，缩短1S的时间。
`RTCAnSUBU`寄存器用于控制校正功能。其中
- `RTCAnDEV`"0"表示每20s校正一次, "1"表示每60s校正一次。
- `RTCAnF6`"0"表示增加, "1"表示减少。
- `RTCAnF[5:0]`设置增加或减少的值。
增加时，校正值等于(`RTCAnF[5:0]`减1)乘2，减少时，校正值等于(`RTCAnF[5:0]`**取反**加1)乘2。

  
以**32.7681kHz**为例： 
时钟周期 T = 1/32768.1, 运行60S的时钟周期数 n = 60 x 32768, 实际时间 t = T x n = (60 x 32768) / 32768.1 = 59.9998169s, 也就是RTC时间快了0.000183105s，因此需要在60s后增加6（0.000183105 x 32768.1）个周期。  
  
以**32.7679kHz**为例：    
时钟运行1分钟的实际时间 t = （60 x 32768) / 32767.9 = 60.00018311s，RTC时间慢了0.00018311s，因此需要在60s后减少6（0.00018311 x 32767.9）个周期。因为是减少，减少值取反加1再乘2为6，寄存器值应该为**111101**。   

|SubSoc    | RTCAnDEV |  RTCAnF6 | RTCAnF[5:0] |
|:---------|:---------|:---------|:------------|
|32768.1   | 1        |  0       | 000100      |
|32767.9   | 1        |  1       | 111101      |
  

### 计算公式
实际时钟f，当f=[32765.93333, 32770.06667]，每60s校正一次，校正值 
```
n = (60 - ((60 x 32768) / f)) x f = 60 x (f - 32768)  
RTCAnSUBU.RTCAnDEV = 1  
```
  
当f=[32761.80000, 32765.93333)或f=(32770.06667, 32774.20000]时，每20s校正一次，校正值  
```  
n = 20 x (f - 32768)
RTCAnSUBU.RTCAnDEV = 0  
```
  
如果n>0，则
```
RTCAnSUBU.RTCAnF6 = 0
RTCAnSUBU.RTCAnF[5:0] = ((uint8)n / 2) + 1  
```
  
否则  
```
RTCAnSUBU.RTCAnF6 = 1
RTCAnSUBU.RTCAnF[5:0] = ((uint8)n / 2)
```
  
**最终得到RTCAnSUBU的值作为校正数据写入NVM。**   
校正值RTCAnF[5:0]共6位，最多可校正值为±124，根据上面的公式，60s校正一次时，f = 32768 + (n / 60), f可校正偏差±2.0667Hz。20s校正一次时，f = 32768 + (n / 20), f可校正偏差±6.2Hz。  
 
**计算代码示例**  
```C
#include "stdlib.h"
#include "stdio.h"

typedef signed int int32;
typedef unsigned int uint32;
typedef unsigned char uint8;

#define RTC_FREQ 327680000
#define RTC_CORRECT_MAX 124
#define RTC_CORRECT_MIN -124 

uint8 get_correct_value(int32 freq)
{
    int32 correct_value;
    uint8 correct_type = 0;
    uint8 result;

    correct_value = (60 * (freq - RTC_FREQ) + 5000) / 10000;

    if ((correct_value > RTC_CORRECT_MAX) || (correct_value < RTC_CORRECT_MIN))
    {   
        correct_type  = 0x80;
        correct_value = (20 * (freq - RTC_FREQ) + 5000) / 10000;

        if (correct_value > RTC_CORRECT_MAX) 
        {   
            correct_value = RTC_CORRECT_MAX;
        }   
        else if (correct_value < RTC_CORRECT_MIN)
        {   
            correct_value = RTC_CORRECT_MIN;
        }   
        else
        {   
        }   
    }   

    if (correct_value > 0)
    {   
        result = (uint8)correct_value / 2 + 1;  
        result |= correct_type;
    }   
    else
    {
        result = (uint8)correct_value / 2;
        result |= correct_type;
    }
    return result;
}

int main(void)
{
    uint8 i;
    uint8 value;
    int32 freqs[] =
    {
        // 20s decrease test
        327618000, 327619000, 327620000, 327657000, 327658000, 327659000,
        // 60s decrease test
        327659333, 327659667, 327660000, 327679000, 327679333, 327679667,
        // 60s increase test
        327680333, 327680667, 327681000, 327700000, 327700333, 327700667,
        // 20s increase test 
        327701000, 327702000, 327703000, 327740000, 327741000, 327742000,
    };

    printf("freq\t\tRTCAnDEV\tRTCAnF6\tRTCAnF[5:0]\n");
    printf("--------------------------------------------------\n");
    for (i = 0; i < 24; i++)
    {
        value = get_correct_value(freqs[i]);
        printf("%d\t%d\t\t%d\t%d%d%d%d%d%d\n", freqs[i], (value>>7)&0x01, (value>>6)&0x01,
                (value>>5)&0x01, (value>>4)&0x01, (value>>3)&0x01, (value>>2)&0x01,
			    (value>>1)&0x01, (value>>0)&0x01 );
    }
    return 0;
}
```
  
代码运行结果如下： 
```
freq		RTCAnDEV	RTCAnF6	RTCAnF[5:0]
--------------------------------------------------
32761.8000	1		1	000010
32761.9000	1		1	000011
32762.0000	1		1	000100
32765.7000	1		1	101001
32765.8000	1		1	101010
32765.9000	1		1	101011
32765.9333	0		1	000010
32765.9667	0		1	000011
32766.0000	0		1	000100
32767.9000	0		1	111101
32767.9333	0		1	111110
32767.9667	0		1	111111
32768.0333	0		0	000010
32768.0667	0		0	000011
32768.1000	0		0	000100
32770.0000	0		0	111101
32770.0333	0		0	111110
32770.0667	0		0	111111
32770.1000	1		0	010110
32770.2000	1		0	010111
32770.3000	1		0	011000
32774.0000	1		0	111101
32774.1000	1		0	111110
32774.2000	1		0	111111
```
  
## 校正流程
1. 发送EOL指令，使产品进入RTC校准模式，开启时钟输出功能。
2. 通过频率计测量32.768kHz晶振的实际频率。如果频率超过可校准范围[32765.93333, 32770.06667]，报不合格。
3. 跟据实际频率计算校正值。
4. 将校正值写入EEPROM。
5. 退出RTC校准模式。



