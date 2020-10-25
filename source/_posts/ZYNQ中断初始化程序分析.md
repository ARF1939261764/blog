---
title: ZYNQ中断初始化程序分析
typora-copy-images-to: ./ZYNQ中断初始化程序分析
typora-root-url: ./ZYNQ中断初始化程序分析
abbrlink: 12566
date: 2020-10-18 19:59:20
tags:
---

&emsp;&emsp;用ZYNQ的BSP初始化中断时感觉有点绕，这里以MIO中断初始化为例，分析一下。

![39578544_p36](/39578544_p36.jpg)

<!-- more -->

# 整体代码

```c
#include <stdio.h>
#include "platform.h"
#include "xil_printf.h"
#include "xparameters.h"
#include "xgpiops.h"
#include "sleep.h"
#include "xscugic.h"

XGpioPs Gpio;
XScuGic intc;

uint8_t key_press;
/***********************************************************************************
 中断处理函数
 **********************************************************************************/
void intr_handler(void *callback_ref)
{
  XGpioPs *gpio = (XGpioPs *) callback_ref;
  if (XGpioPs_IntrGetStatusPin(gpio, 11))
  {
    XGpioPs_IntrClearPin(&Gpio, 11);
    key_press = 1;
  }
}
/***********************************************************************************
中断初始化函数
 **********************************************************************************/
int setup_interrupt_system(XScuGic *gic_ins_ptr,XGpioPs *gpio,uint16_t GpioIntrId)
{
  XScuGic_Config *IntcConfig;
  IntcConfig = XScuGic_LookupConfig(XPAR_SCUGIC_SINGLE_DEVICE_ID);                         /*获取中断控制器和中断分配器的基地址*/
  XScuGic_CfgInitialize(gic_ins_ptr,IntcConfig,IntcConfig->CpuBaseAddress);                /*设置默认中断处理函数,关闭Gic,初始化中断分配器*/
  Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,\
                               (Xil_ExceptionHandler)XScuGic_InterruptHandler,gic_ins_ptr);/*设置IRQ的中断处理函数*/
  Xil_ExceptionEnable();                                                                   /*IRQ使能(通过清除ARM7的CPSR的IRQ中断屏蔽位)*/
  XScuGic_Connect(gic_ins_ptr,GpioIntrId,(Xil_ExceptionHandler)intr_handler,(void *)gpio); /*设置GPIO中断处理函数*/
  XScuGic_Enable(gic_ins_ptr, GpioIntrId);                                                 /*使能Gic*/
  XGpioPs_SetIntrTypePin(gpio, 11, XGPIOPS_IRQ_TYPE_EDGE_FALLING);                         /*中断触发类型*/
  XGpioPs_IntrEnablePin(gpio, 11);                                                         /*使能GPIO11的中断功能*/
  return 0;
}
/***********************************************************************************
main函数
 **********************************************************************************/
int main()
{
  XGpioPs_Config *ConfigPtr;
  ConfigPtr = XGpioPs_LookupConfig(XPAR_XGPIOPS_0_DEVICE_ID);  /*获取GPIO外设的基地址*/
  XGpioPs_CfgInitialize(&Gpio, ConfigPtr,ConfigPtr->BaseAddr); /*初始化XGpioPs结构体,关闭所有GPIO中断*/
  XGpioPs_SetDirectionPin(&Gpio,11,0);                         /*将GPIO的11脚设为输入*/
  setup_interrupt_system(&intc,&Gpio,XPAR_XGPIOPS_0_INTR);     /*设置中断*/
  while(1)
  {
    if(key_press)
    {
      key_press = 0;
      printf("key press\n");
    }
  }
}
```

# 分析

## GPIO初始化

&emsp;&emsp;既然是GPIO输入中断，那配置GPIO必然是少不了的，以前弄STM32时的流程就是打开GPIO时钟，然后设置GPIO为输入模式，ZYNQ不需要单独为某个外设打开时钟，所以只需要将指定的IO配置为输入即可。代码如下：

```c
XGpioPs_Config *ConfigPtr;
ConfigPtr = XGpioPs_LookupConfig(XPAR_XGPIOPS_0_DEVICE_ID);  /*获取GPIO外设的基地址*/
XGpioPs_CfgInitialize(&Gpio, ConfigPtr,ConfigPtr->BaseAddr); /*初始化XGpioPs结构体,关闭所有GPIO中断*/
XGpioPs_SetDirectionPin(&Gpio,11,0);                         /*将GPIO的11脚设为输入*/
```

### XGpioPs_Config结构体

XGpioPs_Config定义如下：

```c
/*xgpiops.h*/
typedef struct {
	u16 DeviceId;		/**< Unique ID of device */
	u32 BaseAddr;		/**< Register base address */
} XGpioPs_Config;
```

&emsp;&emsp;用于存放设备ID和设备的基地址，这个设备在这里就是指GPIO外设。

### XGpioPs_LookupConfig函数

XGpioPs_LookupConfig函数如下：

```c
/*xgpiops_sinit.c*/
XGpioPs_Config *XGpioPs_LookupConfig(u16 DeviceId)
{
	XGpioPs_Config *CfgPtr = NULL;
	u32 Index;

	for (Index = 0U; Index < (u32)XPAR_XGPIOPS_NUM_INSTANCES; Index++) {
		if (XGpioPs_ConfigTable[Index].DeviceId == DeviceId) {
			CfgPtr = &XGpioPs_ConfigTable[Index];
			break;
		}
	}

	return (XGpioPs_Config *)CfgPtr;
}
```

&emsp;&emsp;这个函数根据设备ID在`XGpioPs_ConfigTable`中查找存放设备基地址的结构体。

&emsp;&emsp;函数中用一个for循环遍历数组`XGpioPs_ConfigTable`，当查找到指定设备ID时，退出for循环，返回查找结果。

```c
XGpioPs_Config XGpioPs_ConfigTable[XPAR_XGPIOPS_NUM_INSTANCES] =
{
	{
		XPAR_PS7_GPIO_0_DEVICE_ID,
		XPAR_PS7_GPIO_0_BASEADDR
	}
};
```

&emsp;&emsp;`XGpioPs_ConfigTable`定义如上，每个数组元素存放一个设备的设备ID和设备基地址。

### XGpioPs_CfgInitialize函数

&emsp;&emsp;XGpioPs_CfgInitialize函数比较长，就不整个贴出来了，下面分块来分析。

```c
s32 XGpioPs_CfgInitialize(XGpioPs *InstancePtr, const XGpioPs_Config *ConfigPtr,u32 EffectiveAddr);
```

&emsp;&emsp;XGpioPs_CfgInitialize函数原型如上，其功能主要是为了初始化一个XGpioPs结构体变量，将硬件外设与XGpioPs结构体变量联系起来，并关闭所有GPIO的中断功能。XGpioPs结构体定义如下：

```c
typedef struct {
	XGpioPs_Config GpioConfig;	/**< Device configuration */
	u32 IsReady;			    /**< Device is initialized and ready */
	XGpioPs_Handler Handler;	/**< Status handlers for all banks */
	void *CallBackRef; 		    /**< Callback ref for bank handlers */
	u32 Platform;			    /**< Platform data */
	u32 MaxPinNum;			    /**< Max pins in the GPIO device */
	u8 MaxBanks;			    /**< Max banks in a GPIO device */
    u32 PmcGpio;                /**< Flag for accessing PS GPIO for versal*/
} XGpioPs;
```

* `XGpioPs_Config GpioConfig`：存放设备信息(设备ID，设备基地址)。
* `u32 IsReady`：设备初始化就绪标志，在XGpioPs_CfgInitialize执行开始时，该变量被清零，执行即将结束时，被置位。
* `XGpioPs_Handler Handler`：状态处理函数？。。不知道是做啥的。
* `void *CallBackRef`：前面状态处理函数的参数。
* `u32 Platform`：平台信息，区分硬件平台(`XPLAT_MICROBLAZE`,`XPLAT_ZYNQ`,`XPLAT_ZYNQ_ULTRA_MP`)。
* `u32 MaxPinNum`：最大IO数量。
* `u8 MaxBanks`：最大Bank数量。
* `u32 PmcGpio`：不知道啥意思，代码中没有初始化这变量。

**初始化结构体**

```c
/*
 * Set some default values for instance data, don't indicate the device
 * is ready to use until everything has been initialized successfully.
 */
InstancePtr->IsReady = 0U;
InstancePtr->GpioConfig.BaseAddr = EffectiveAddr;
InstancePtr->GpioConfig.DeviceId = ConfigPtr->DeviceId;
InstancePtr->Handler = (XGpioPs_Handler)StubHandler;
InstancePtr->Platform = XGetPlatform_Info();
/* Initialize the Bank data based on platform */
if (InstancePtr->Platform == (u32)XPLAT_ZYNQ_ULTRA_MP) {
    /*do something*/
}
else if (InstancePtr->Platform == (u32)XPLAT_versal)
{
     if(InstancePtr->PmcGpio == (u32)FALSE)
     {
         
         /*do something*/
     }
     else
     {
         /*do something*/
     }
}
else {
    /*
     *	Max pins in the GPIO device
     *	0 - 31,  Bank 0
     *	32 - 53, Bank 1
     *	54 - 85, Bank 2
     *	86 - 117, Bank 3
     */
    InstancePtr->MaxPinNum = (u32)118;
    InstancePtr->MaxBanks = (u8)4;
}
```

&emsp;&emsp;结构体初始化代码如上，注意`XGetPlatform_Info()`函数，通过这个函数，我们在ZYNQ7020上得到的结果是`XPLAT_ZYNQ`，所以在最后`MaxPinNum`和`MaxBanks`分别被初始化为118和4，即表示GPIO数量最大为118，Bank数量最大为4，这与实际情况相符。

**关闭中断**

```c
for (i=(u8)0U;i<InstancePtr->MaxBanks;i++) {
      if (InstancePtr->Platform == XPLAT_versal){
          /*do something*/
      }
      else
      {
          /*znyq*/
          XGpioPs_WriteReg(InstancePtr->GpioConfig.BaseAddr,((u32)(i) * XGPIOPS_REG_MASK_OFFSET) +
                 XGPIOPS_INTDIS_OFFSET, 0xFFFFFFFFU);
      }
}
```

&emsp;&emsp;主要是通过向`INT_DIS `(中断失能寄存器(Disable))寄存器写`0xFFFFFFFFU`来实现，for循环总线循环4次，每次对应每个Bank的`INT_DIS`寄存器，四个寄存器的名字分别为：

* `XGPIOPS_INTDIS_OFFS ET`
* `INT_DIS_1`
* `INT_DIS_2`
* `INT_DIS_3`

&emsp;&emsp;这个名字可以在手册UG585上查到:

![image-20201019004256499](/image-20201019004256499.png)

&emsp;&emsp;至此XGpioPs_CfgInitialize函数就分析完了。

## 中断初始化

&emsp;&emsp;终于到今天的主题了，下面好好分析一波。

```c
/***********************************************************************************
定义XScuGic结构体
 **********************************************************************************/
XScuGic intc;
/***********************************************************************************
中断初始化函数
 **********************************************************************************/
int setup_interrupt_system(XScuGic *gic_ins_ptr,XGpioPs *gpio,uint16_t GpioIntrId)
{
  XScuGic_Config *IntcConfig;
  IntcConfig = XScuGic_LookupConfig(XPAR_SCUGIC_SINGLE_DEVICE_ID);                           /*获取中断控制器和中断分配器的基地址*/
  XScuGic_CfgInitialize(gic_ins_ptr,IntcConfig,IntcConfig->CpuBaseAddress);                  /*设置默认中断处理函数,关闭Gic,初始化中断分配器*/
  Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
                               \(Xil_ExceptionHandler)XScuGic_InterruptHandler,gic_ins_ptr); /*设置IRQ的中断处理函数*/
  Xil_ExceptionEnable();                                                                     /*IRQ使能(通过清除ARM7的CPSR的IRQ中断屏蔽位)*/
  XScuGic_Connect(gic_ins_ptr,GpioIntrId,(Xil_ExceptionHandler)intr_handler,(void *)gpio);   /*设置GPIO中断处理函数*/
  XScuGic_Enable(gic_ins_ptr, GpioIntrId);                                                   /*使能Gic*/
  XGpioPs_SetIntrTypePin(gpio, 11, XGPIOPS_IRQ_TYPE_EDGE_FALLING);                           /*中断触发类型*/
  XGpioPs_IntrEnablePin(gpio, 11);                                                           /*使能GPIO11的中断功能*/
  return 0;
}
/***********************************************************************************
在主函数中调用
 **********************************************************************************/
int main(void)
{
    /*... other code ...*/
    setup_interrupt_system(&intc,&Gpio,XPAR_XGPIOPS_0_INTR);     /*设置中断*/
    /*... other code ...*/
}
```

&emsp;&emsp;再把中断初始化代码摆一遍，方便查看。

### XScuGic_LookupConfig函数

```c
/**
 * This typedef contains configuration information for the device.
 */
typedef struct
{
	u16 DeviceId;		/**< Unique ID  of device */
	u32 CpuBaseAddress;	/**< CPU Interface Register base address */
	u32 DistBaseAddress;	/**< Distributor Register base address */
	XScuGic_VectorTableEntry HandlerTable[XSCUGIC_MAX_NUM_INTR_INPUTS];/**<
				 Vector table of interrupt handlers */
} XScuGic_Config;
```

&emsp;&emsp;这个函数和前面的`XGpioPs_LookupConfig`函数功能类似，不做过多阐述，唯一有一点不同的是，它这个函数获取到的结构体里面的地址有两个，一个中断控制器地址，一个中断分配器地址，上面结构体是通过`XScuGic_LookupConfig`函数获取到的结构体，可以看到里面有两个地址和一个中断向量表。

### XScuGic_CfgInitialize函数

&emsp;&emsp;这个函数主要负责：

* 如果CPU是CPU1，则判断CPU1是否可用。
* 设置默认中断向量表。
* 关闭中断控制器。
* 初始化中断分配器
* 初始化CPU的GIC接口

下面一步步分析。

#### 判断CPU1是否可用

```c
#ifdef ARMA9
	if (XPAR_CPU_ID == 0x01) {
		Xil_AssertNonvoid((Xil_In32(XPS_EFUSE_BASEADDR
			+ EFUSE_STATUS_OFFSET) & EFUSE_STATUS_CPU_MASK) == 0);
	}
#endif
```

&emsp;&emsp;首先会判断宏`ARMA9`是否定义，`ARMA9`就是表示Cortex-A9，这里这个宏是存在的，然后判断CPU ID，这里实际上不会进行判断，这里我们用的CPU是CPU0。就先假设是CPU1吧，如果是CPU1，那么它会读一个寄存器，然后与上`EFUSE_STATUS_CPU_MASK`，再和0做比较。读的这个寄存器地址实际上是0xF800D010，`EFUSE_STATUS_CPU_MASK`则是0x80，所以这里是判断地址为0xF800D010寄存器的第7bit是否为0，如果为0则表示正确，否则表示出现错误。

&emsp;&emsp;这个寄存器的定义我在UG585里面没有找到，但是在xilinx的社区找到了一个回答，可以参考一下。

![image-20201019235811930](/image-20201019235811930.png)

&emsp;&emsp;即bit7为1表示CPU1被禁用了，为0表示CPU1没有被禁用。另外注意这个寄存器是<font color='red'>一次性可编程</font> 的，也就是说如果把这个寄存器的bit7编程为1，那么CPU1就永久被禁用了。

#### 设置默认中断向量表

```c
for (Int_Id = 0U; Int_Id < XSCUGIC_MAX_NUM_INTR_INPUTS;Int_Id++) {
  /*
  * Initalize the handler to point to a stub to handle an
  * interrupt which has not been connected to a handler
  * Only initialize it if the handler is 0 which means it
  * was not initialized statically by the tools/user. Set
  * the callback reference to this instance so that
  * unhandled interrupts can be tracked.
  */
  if ((InstancePtr->Config->HandlerTable[Int_Id].Handler == (Xil_InterruptHandler)NULL)) {
    InstancePtr->Config->HandlerTable[Int_Id].Handler = (Xil_InterruptHandler)StubHandler;
  }
  InstancePtr->Config->HandlerTable[Int_Id].CallBackRef = InstancePtr;
}
```

&emsp;&emsp;这段代码用于设置默认中断向量表，这个中断向量表是IRQ中断向量表。这里会将所有为空的中断向量全部替换成默认中断服务函数。

#### 关闭GIC，初始化中断分配器，初始化CPU中断的一些关键参数

```c
XScuGic_Stop(InstancePtr);
DistributorInit(InstancePtr, Cpu_Id);
CPUInitialize(InstancePtr);
```

&emsp;&emsp;就3个函数，函数内部的代码就不分析了，不影响理解初始化流程。

### Xil_ExceptionRegisterHandler函数

```c
XExc_VectorTable[Exception_id].Handler = Handler;
XExc_VectorTable[Exception_id].Data = Data;
```

&emsp;&emsp;这个函数的代码很简单，功能也很明了，就是为IRQ中断设置中断服务函数。这里中断服务函数被设置为了`XScuGic_InterruptHandler`。当IRQ中断服务函数发生时，这个函数会从IRQ中断服务向量表中查找对应外设的中断服务函数，然后调用执行。

### Xil_ExceptionEnable函数

&emsp;&emsp;这个函数通过调用汇编指令，通过清除ARM核的`CPSR`寄存器的IRQ中断禁止位来打开IRQ中断。

### XScuGic_Connect函数

&emsp;&emsp;这个函数通过操作IRQ中断向量表来设置指定中断的中断服务函数。

### XScuGic_Enable函数

&emsp;&emsp;顾名思义，中断使能，这个函数用来使能指定外设的中断。

### XGpioPs_SetIntrTypePin，XGpioPs_IntrEnablePin

&emsp;&emsp;这两个函数分别用来设置中断触发类型和中断那些引脚使能中断功能。

## 总括

&emsp;&emsp;总结一下吧，我们这里初始化中断，我们会初始化两个中断向量表，一个**系统中断向量表**，**一个IRQ中断向量表**，大致结构就是这样的：

![image-20201023000604612](/image-20201023000604612.png)

&emsp;&emsp;**当IRQ中断发生时，程序会首先跳转到系统中断中断向量表，系统中断向量表中有跳转指令，会跳转到对应的系统中断服务函数，然后系统中断服务函数则会在根据中断ID(通过GIC的寄存器得到)在IRQ中断向量表中查询各个外设的中断服务函数，并调用执行，从而实现中断功能。**

&emsp;&emsp;系统的启动地址既是系统中断向量表的位置，即系统一启动便是执行复位中断的中断服务函数(不考虑重定位中断向量表)，想想也应该这样，系统只有在复位时才会重启，所以复位后执行的第一条指令应该就是复位中断服务函数，而这7个中断向量实际上是对应着7条跳转指令的，有两个方法可以验证这个想法：

**看源码**

&emsp;&emsp;打开连接文件，可以看到有一个`vectors`块是在代码最前面的，顾名思义，这就是中断向量所在的块了，并且还加了KEEP指令，防止之这个块被优化掉，说明这个块很重要，这更加验证了这个想法。

```assembly
SECTIONS
{
.text : {
   KEEP (*(.vectors))
   *(.boot)
   *(.text)
   *(.text.*)
   *(.gnu.linkonce.t.*)
   *(.plt)
   *(.gnu_warning)
   *(.gcc_execpt_table)
   *(.glue_7)
   *(.glue_7t)
   *(.vfp11_veneer)
   *(.ARM.extab)
   *(.gnu.linkonce.armextab.*)
} > ps7_ddr_0
```
&emsp;&emsp;再在zynq sdk工程目录下搜索文件内容`vectors`

![image-20201022233105379](/image-20201022233105379.png)

&emsp;&emsp;可以看到这里定义了一个`vector`块，打开我们可以看到如下代码：

```
.section .vectors
_vector_table:
	B	_boot
	B	Undefined
	B	SVCHandler
	B	PrefetchAbortHandler
	B	DataAbortHandler
	NOP	/* Placeholder for address exception vector*/
	B	IRQHandler
	B	FIQHandler
```

&emsp;&emsp;没错，就是它了，很明显，7种中断对应7条跳转指令，还有一个`NOP`指令用来填充保留位置。

**看编译后的汇编文件**

&emsp;&emsp;这个方法更加直接明了了。

![image-20201022233411273](/image-20201022233411273.png)

&emsp;&emsp;这个0x100000就是我们zynq的启动地址，在链接脚本中我们可以验证。

![image-20201022233506439](/image-20201022233506439.png)

&emsp;&emsp;扯远了扯远了，回到正轨，当中断发生后，系统会第一时间跳转到这个向量表，然后通过向量表的跳转指令跳转到各自的中断服务函数，所以当发生IRQ中断时，程序会跳转到`IRQHandler`中，`IRQHandler`是一个汇编函数段，代码如下：

```assembly
IRQHandler:          /* IRQ vector handler */

  stmdb  sp!,{r0-r3,r12,lr}    /* state save from compiled code*/
#if FPU_HARD_FLOAT_ABI_ENABLED
  vpush {d0-d7}
  vpush {d16-d31}
  vmrs r1, FPSCR
  push {r1}
  vmrs r1, FPEXC
  push {r1}
#endif

#ifdef PROFILING
  ldr  r2, =prof_pc
  subs  r3, lr, #0
  str  r3, [r2]
#endif

  bl  IRQInterrupt      /* IRQ vector */

#if FPU_HARD_FLOAT_ABI_ENABLED
  pop   {r1}
  vmsr    FPEXC, r1
  pop   {r1}
  vmsr    FPSCR, r1
  vpop    {d16-d31}
  vpop    {d0-d7}
#endif
  ldmia  sp!,{r0-r3,r12,lr}    /* state restore from compiled code */

  subs  pc, lr, #4      /* adjust return */
```

&emsp;&emsp;忽略宏定义包含的代码段，有一条`bl IRQInterrupt`指令，这条指令会跳转到代码段`IRQInterrupt`，而`IRQInterrupt`则是`verctor.c`中定义的一个C语言函数，代码如下：

```c
void IRQInterrupt(void)
{
  XExc_VectorTable[XIL_EXCEPTION_ID_IRQ_INT].Handler(XExc_VectorTable[
          XIL_EXCEPTION_ID_IRQ_INT].Data);
}
```

&emsp;&emsp;代码很简单，从前面说的系统中断向量表中取出IRQ中断服务函数，然后执行，并且还传入进了一个参数。

&emsp;&emsp;根据前面的分析，这时候执行的`XExc_VectorTable[XIL_EXCEPTION_ID_IRQ_INT]`中的函数应该就是`XScuGic_InterruptHandler`函数，

```c
    /*获取中断ID*/
    IntIDFull = XScuGic_CPUReadReg(InstancePtr, XSCUGIC_INT_ACK_OFFSET);
    InterruptID = IntIDFull & XSCUGIC_ACK_INTID_MASK;
    /*判断中断ID是否大于最大中断ID号*/
    if (XSCUGIC_MAX_NUM_INTR_INPUTS <= InterruptID) {
        goto IntrExit;
    }
    /*获取外设中断服务函数的函数指针*/
    TablePtr = &(InstancePtr->Config->HandlerTable[InterruptID]);
    if (TablePtr != NULL) {
        /*执行外设中断服务函数*/
        TablePtr->Handler(TablePtr->CallBackRef);
    }
IntrExit:
    /*写中断结束寄存器*/
    XScuGic_CPUWriteReg(InstancePtr, XSCUGIC_EOI_OFFSET, IntIDFull);
```

&emsp;&emsp;可以看出，`XScuGic_InterruptHandler`函数实际上就是在IRQ中断向量表中查找对应ID的中断服务函数，然后执行，而这个IRQ中断向量表就是前面GIC结构体中的那个向量表。

&emsp;&emsp;至此，ZYNQ的GPIO中断初始化过程分析就完成了，撒花。

# 完

