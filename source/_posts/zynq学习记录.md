---
title: 'zynq学习记录'
categories:
  - - FPGA
    - ZYNQ
tags:
  - ZYNQ
  - FPGA
typora-root-url: ./zynq学习记录
typora-copy-images-to: ./zynq学习记录
abbrlink: 19301
date: 2020-07-26 01:06:18
---

&emsp;&emsp;开始学习ZYNQ，加油。

![70171811_p0](/70171811_p0.jpg)

<!-- more -->

------

# 点亮第一个LED

&emsp;&emsp;开始学习xilinx FPGA及ZYNQ的使用，记录第一步：hello world与点亮第一个led灯;

## 步骤

### 新建工程

![image-20200726011516555](/image-20200726011516555.png)

### 点击**Create Block Design**

![02](/02.png)

### 命名

![image-20200726012028674](/image-20200726012028674.png)

### 添加CPU

![image-20200726012044376](/image-20200726012044376.png)

### 设置CPU

&emsp;&emsp;将**PS-PL Configuraltion -> AXI Non Secure Enablement -> GP Master AXI Interface -> M AXI GP0 Interface**取消勾选。

![image-20200726012116630](/image-20200726012116630.png)

&emsp;&emsp;勾选**UART1**和**GPIO MIO**，注意**UART1**的位置。

![image-20200726012358194](/image-20200726012358194.png)

然后设置DDR

![image-20200726012625377](/image-20200726012625377.png)

确认保存后点击Run Block Automation

![image-20200726012703131](/image-20200726012703131.png)

弹出来的对话框确认即可

![image-20200726012803852](/image-20200726012803852.png)

### 生成HDL Wrapper

点击Create HDL Wrapper

![image-20200726012825134](/image-20200726012825134.png)

生成完后源文件结构会变成如下结构，多了个包装。

![image-20200726012933883](/image-20200726012933883.png)

然后右击cpu点击Generate Output Proudcts

![image-20200726013006927](/image-20200726013006927.png)

然后导出Hardware

![image-20200726013052723](/image-20200726013052723.png)

### 运行SDK

![image-20200726013119176](/image-20200726013119176.png)

打开后的界面，设置终端连接

![image-20200726013136934](/image-20200726013136934.png)

### 运行Hello world

点击运行，终端出现`Hello World`。

![image-20200726013154065](/image-20200726013154065.png)

### LED测试

将代码替换如下：

![image-20200726013250851](/image-20200726013250851.png)

```c
#include <stdio.h>
#include "platform.h"
#include "xil_printf.h"
#include "xparameters.h"
#include "xgpiops.h"
#include "xstatus.h"
#include "xplatform_info.h"
#include "sleep.h"

XGpioPs Gpio;

#define GPIO_DEVICE_ID		XPAR_XGPIOPS_0_DEVICE_ID

int main()
{
    XGpioPs_Config *ConfigPtr;/*GPIO配置结构体,用于存放GPIO外设的设备ID和设备基地址*/
    print("Hello World\n\r");/*打印一个hello world*/
    ConfigPtr = XGpioPs_LookupConfig(GPIO_DEVICE_ID);/*根据设备ID寻找设备寄存器基地址*/
    XGpioPs_CfgInitialize(&Gpio,ConfigPtr,ConfigPtr->BaseAddr);/*初始化外设*/
    XGpioPs_SetDirectionPin(&Gpio, 9, 1);/*IO设置为输出*/
    XGpioPs_SetOutputEnablePin(&Gpio, 9, 1);/*输出使能*/
    while(1)
    {
    	 XGpioPs_WritePin(&Gpio, 9, 0x0);
    	 usleep(100000);
    	 XGpioPs_WritePin(&Gpio, 9, 0x1);
		 usleep(100000);
    }
}
```

下载即可看到IO灯在闪(注意IO分配)。

------

# EMIO的使用

&emsp;&emsp;emio是ZYNQ连接到PL内部的一些IO，这些IO既可以连接到PL逻辑内部，供PS-PL之间传递信息使用，也可以由PL端直接旁路到PL的IO端，作为普通IO使用。

## 步骤

&emsp;&emsp;在点亮led工程的基础上，做如下操作即可：

### 勾选EMIO，并设置宽带

![image-20201010002827973](/image-20201010002827973.png)

### 右键新添加的GPIO端口，并选择`Make External`选项

![image-20201010003001674](/image-20201010003001674.png)

![image-20201010003120035](/image-20201010003120035.png)

### 然后重新生成`wrapper`和`output product`

### 运行一下综合后点击`Schematic`，给新增加的GPIO分配管脚

![image-20201010003405404](/image-20201010003405404.png)

### 然后重新导出硬件

&emsp;&emsp;软件测试代码如下：

```c
#include <stdio.h>
#include "platform.h"
#include "xil_printf.h"
#include "xgpiops.h"
#include "xparameters.h"
#include "sleep.h"

XGpioPs Gpio;

int main()
{
    XGpioPs_Config* xgps_Config;
    
    init_platform();

    print("Hello World\n\r");

    xgps_Config = XGpioPs_LookupConfig(XPAR_XGPIOPS_0_DEVICE_ID);

    XGpioPs_CfgInitialize(&Gpio,xgps_Config,xgps_Config->BaseAddr);

    XGpioPs_SetDirectionPin(&Gpio,9,1);

    XGpioPs_SetDirectionPin(&Gpio,54,0);

    XGpioPs_SetOutputEnablePin(&Gpio,9,1);

    while(1)
    {
    	XGpioPs_WritePin(&Gpio,9,XGpioPs_ReadPin(&Gpio,54));
    }

    cleanup_platform();
    return 0;
}
```

和mio的使用类似。