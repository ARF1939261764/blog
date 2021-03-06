---
title: ov5640-hdmi显示总结
typora-copy-images-to: ov5640-hdmi显示总结
categories:
  - - 摄像头
    - ov5640
  - - HDMI
  - - SDRAM
  - - FPGA
tags:
  - 摄像头
  - ov5640
  - HDMI
  - SDRAM
typora-root-url: ./ov5640-hdmi显示总结
abbrlink: 36961
date: 2020-06-29 13:06:05
---

&emsp;&emsp;博客建起来了，还是得写一些东西，不然白搞了，正好最近完成了一个ov5640-hdmi显示的东西，就写一下弄这个东西中遇到的一些问题吧。

![69759328_p0](/69759328_p0.jpg)

<!-- more -->

# 介绍

&emsp;&emsp;先大体说一下做这个东西的一些思路。

&emsp;&emsp;首先是这个东西需要干什么，简单说就是用FPGA驱动OV5640摄像头模块，获取OV5640的视频数据，然后通过HDMI接口将OV5640采集的视频数据显示在显示器上。本次视频分辨率定为1024*768，帧率为30FPS。另外除了需要将视频显示在HDMI显示器上，还需要实现截图功能，但到目前(2020年6月29日)截图功能还没有实现(但是预留了接口)。

![总体框架图1](/%E6%80%BB%E4%BD%93%E6%A1%86%E6%9E%B6%E5%9B%BE1-1593408603044.png)

&emsp;&emsp;这是本次设计的系统框图。上电等待20ms以上，然后由`IIC_Config`模块配置ov5640，将一大串寄存器数据写入到OV5640中，然后OV5640便会输出对应分辨率的视频数据，然后`Get Frame`模块则负责读取这些数据，并将其写入到`FIFO 1`中，`FIFO 1`是一个由内部Block RAM构成的异步FIFO，用于两个时钟域的数据传递，并为写SDRAM提供一个缓冲，因为SDRAM不一定是想写，就立马能够写的。`Frame write`模块则读取来自`FIFO 1`中的数据，将其写入到SDRAM，而`AVL Bus Controller`是一个总线控制器，负责将SDRAM转换成一个多端口、宏观上读写可以同时进行的存储器，总线仲裁采用轮询仲裁。`Frame read`模块负责从SDRAM中读数据，然后将读到的数据写入到`FIFO 2`中，`FIFO 2`的作用也主要是为了实现跨时钟域传递数据，另外提供一个缓冲作用。RGB2VGA主要是负责生成行、场同步信号和数据有效信号de，然后将这些信号连同数据一起送往`HDMI Controller`完成显示。

# 遇到的问题

&emsp;&emsp;遇到的问题得找到解决办法然后记下来。

* 通过SCCB总线配置OV5640时，需要提供xclk，否则OV5640不会响应SCCB总线的读写请求。如果摄像头模块上自带晶振，则用晶振提供就好了，如果没有则需要使用FPGA提供时钟。最开始受以前一些相对简单的芯片的IIC总线的影响，以为不需要提供时钟，然后折腾了大半天，我给到摄像头的时序和Demo的时序一模一样，可是我的就是读不出ID，没有应答信号，最后才发现是xclk的问题。想想也应该，以前哪些简单点的芯片总共就几个寄存器，但是ov5640那么多寄存器，里面还有mcu，那sccb控制器就可以看作一个外设挂在ov5640内部的总线上，那自然也要时钟了。

* OV5640的SCCB总线会发出应答信号，SCCB总线规范上说不用在意应答信号，但是实测OV5640的SCCB总线是有应答信号的。也就是说，除了不能(没有实测过)突发读写，其它时序都和IIC一模一样。

* OV5640的寄存器地址有16位，写寄存器地址时序可以参考下面的时序图。这是AT24C1024的写数据时序图，和SCCB的写时序一样，读时序类推即可。

  ![image-20200629165403544](/image-20200629165403544.png)

* SDRAM页突发读/写配合`BURST TERMINATE`命令可以实现可变突发长度，但是页突发读/写<font color="red">不支持</font>自动预充电。

  ![image-20200629165850233](/image-20200629165850233.png)

* 对于DMT时序，这个`HSync`和`VSync`一直都是周期重复的，尤其是`HSync`，而不是`VSync`不处于`Addr Time`时，`HSync`信号就没有了，如果是这样，显示器会没有画面。

  ![image-20200629170347268](/image-20200629170347268.png)

* SDRAM的潜伏期一般可以是2或者3，这个潜伏期的选择和SDRAM芯片的工作频率有关，频率很高的话就得让潜伏期为3，频率低就可以设置为2，以提高读写速度。下图是H57V2562GTR 潜伏期与时钟周期的关系。

* ![image-20200629173235119](/image-20200629173235119.png)



# 各个模块

## HDMI

&emsp;&emsp;**HDMI(High-Definition Multimedia Interface ，高清晰度多媒体接口)**，用于传输视频和音频。

![图片3](/%E5%9B%BE%E7%89%873-1593523003774.png)

<center>VGA场同步时序</center>

![图片3](/%E5%9B%BE%E7%89%873.png)

<center>VGA行同步时序</center>

&emsp;&emsp;首先回想VGA显示，VGA时序主要需要行场同步信号信号和RGB数据信号，这些信号由FPGA实现的VGA驱动产生，然后通过FPGA外围的DAC或者权电阻网络将RGB数字信号转换为模拟信号传输，VGA时序图如上。

&emsp;&emsp;如果只需显示图像，则HDMI和VGA类似，只不过产生行场同步信号/数据信号后，不由外部权电阻网络转换成模拟电压，而是由编码器通过TMDS编码算法将这些信号编码后，再通过串行化器将数字信号通过差分数据线发送给显示器。

![HDMI](/HDMI.png)

&emsp;&emsp;上图便是HDMI控制器的结构图(没有包含时序发生部分)，主要包含编码器和串行化器。而前面没有画出的时序发生部分和VGA的时序发生部分非常相似(或者完全一样？)，只是多了一个`de`信号，这个`de`信号在像素数据有效的时候为高，像素数据无效的时候为低。

&emsp;&emsp;编码器的编码算法在HDMI的规范文档上给出了，但是它给出的不包含行场同步信号的处理，所以不完整，而DVI得编码算法流程图和HDMI的一样，所以可以在DVI的规范文档里面找到完整的编码流程图。完整的编码流程图如下图所示。

![image-20200630204504342](/image-20200630204504342.png)

&emsp;&emsp;编码器在编码的时候可以采用流水线的方法提高编码器最高运行速度，流水线级数也可以任意长，但是要求平均每个时钟能完成一次编码，即每个像素时钟都可以输出一个编码结果。

&emsp;&emsp;编码器将视频流数据编码成3组10bit数据，然后串行化器将这10bit数据发送出去，由于每个像素时钟周期都要完成3组10bit的数据的发送，所以串行数据率是像素时钟的10倍，这个频率就比较高了，所以串行化器一般使用FPGA提供的双数据率IO(DDIO)，或者FPGA提供的串行化IP核，双数据率IO在每个时钟上升沿核下降沿都会发送数据，所以串行化器的工作频率就可以从原来的10倍像素时钟降到5倍像素时钟。

## SDRAM

&emsp;&emsp;SDRAM用于缓存数据帧，由于摄像头给出的、HDMI需要的数据都是大量的、实时的数据，且两者的数据率不一样，所以需要用SDRAM将摄像头给出的数据帧缓存起来，HDMI控制器再从SDRAM控制器中读取数据显示处理。

&emsp;&emsp;本次将SDRAM的操作封装成了一个遵循 AVL总线从机标准的存储器设备，avl总线是做毕设RISC-V CPU时自定义的一个总线标准，虽然现在发现有些问题，但这里还是能用。这个时序标准参考了avalon总线和蜂鸟E200 CPU里面的ICB总线。通过这个时序，能随机读写SDRAM任意位置的数据，且支持突发长度为1-256的突发读/写(这个总线是32bit的，所以对应到SDRAM就是长度为2-512的突发读写)。

&emsp;&emsp;SDRAM网上资料很多，这里主要想说的就是关于SDRAM时钟该怎么给。这里先定义两个符号：`clk`和`sdram_clk`，`clk`为SDRAM控制器的时钟，`sdram_clk`为给到SDRAM芯片的时钟。由于SDRAM各个信号的建立时间和保持时间都比较长，所以像FPGA操作内部SRAM时，数据跳变与时钟对齐这样的方法肯定是不行的，保持时间会出问题。正点原子还有Quartus的一篇官方文档**《Quartus II Handbook Version 8.1 Volume 5: Embedded Peripherals》**都是通过PLL给的，通过PLL将给到SDRAM的时钟移相，即让`clk`和`sdram_clk`保持一个相位差，这样也可以，我用的方法则是将给到SDRAM控制器的时钟反相后给到SDRAM，即`assign sdram_clk = ~clk;`这样做的话就相当于SDRAM控制器上升沿给数据/命令到SDRAM，SDRAM在SDRAM时钟的上升沿(即SDRAM控制器的下降沿)接收数据，那这样在SDRAM时钟的上升沿正好处于数据/命令稳定时的中间位置，我用的SDRAM时钟周期最低是7.6ns，则SDRAM时钟上升沿前后最少都有3.8ns的时间，这个时间可以满足我用的SDRAM所需要的1.5ns的建立时间、0.8ns的保持时间。所以发送给SDRAM的数据是没问题的，主要是接收SDRAM返回的数据看有没有问题。

&emsp;&emsp;接收SDRAM反馈的数据时也需要特殊处理，我用了一个16bit的寄存器在`clk`的下降沿(即`sdram_clk`的上升沿)去采样SDRAM芯片的数据，然后SDRAM内部在`clk`上升沿的时候去读取这个寄存器的数据以间接的到SDRAM的数据。这样做实践上是可行的。

![image-20200630224116739](/image-20200630224116739.png)

&emsp;&emsp;如上图，SDRAM返回的数据从T1(CL==2时)或T2(CL==3时)上升沿时开始准备出来，经过t<sub>AC</sub>后，数据变得可读，然后在下一个时钟周期的下降沿消失。所以我们需要判断的，就是在T2(CL==2时)或T3(CL==3时)上升沿时，数据是否可读，查看数据手册，我用的SDRAM芯片的t<sub>AC</sub>在CL==2或CLK==3时均小于最小SDRAM时钟周期的7.6ns，所以在SDRAM的上升沿可以读取到SDRAM返回的数据，所以前面反相的方法是可行的。

&emsp;&emsp;这样做的好处就是可以省一个PLL，并且减少代码的依赖性。

## OV5640

&emsp;&emsp;OV5640本次涉及的主要就两块：通过SCCB总线配置内部寄存器数据、将OV5640 DVP接口传过来的数据转换成16bit RGB565数据。

### SCCB

&emsp;&emsp;**Omnivision serial camera control bus**，其实就是IIC总线的一个子集，用于读写OV5640内部的寄存器。

### DVP接口

![image-20200630232622479](/image-20200630232622479.png)

&emsp;&emsp;时序和DMT时序标准定义的实习比较像，这个HREF就是DE信号，所有的数据通过OV5640输出的时钟`pclk`来同步，传递RGB565数据时，在HREF有效器件，每个`pclk`传递一个字节的数据，两个`pclk`时钟周期传递一个像素点数据。

&emsp;&emsp;关于寄存器配置，还只看了个大概，等过些天详细看一下、尝试一下后补上。

# 整体设计

![总体框架图1](/%E6%80%BB%E4%BD%93%E6%A1%86%E6%9E%B6%E5%9B%BE1-1593408603044.png)

&emsp;&emsp;再一次摆上系统框图。

&emsp;&emsp;主要说我认为的、一些容易出问题的地方。

## 多口SDRAM

&emsp;&emsp;SDRAM只有一个读写接口，同一时刻只能完成读或者写操作，而摄像头和HDMI控制器需要同时读写数据，如果需要截图，则会有3个地方需要同时操作SDRAM，所以需要对SDRAM的操作请求做一个调度，由于前面将SDRAM控制器封装成了一个符合AVL从机时序的存储器设备，所以可以通过AVL总线控制器将SDRAM扩展成一个多口的SDRAM，AVL总线控制器是毕设时弄的一个总线控制器，这个总线控制器可以支持主机、从机数量配置，支持固定优先级和轮询调度，具体实现可以看这里：[avl总线标准及总线控制器的设计](https://www.xiaobaidreamworks.com/2020/06/30/avl%E6%80%BB%E7%BA%BF%E6%A0%87%E5%87%86%E5%8F%8A%E6%80%BB%E7%BA%BF%E6%8E%A7%E5%88%B6%E5%99%A8%E7%9A%84%E8%AE%BE%E8%AE%A1/)。如果不想自己写总线控制器，也可以将SDRAM的操作封装成符合Avalon总线从机标准的存储器设备，然后用Qsys(也就是新版Quartus II里面的Platform Designer)提供的Avalon总线控制器进行扩展，当然这样也比较麻烦，每次修改代码都需要重新用Qsys生成，然后才能综合。另外也可以直接将SDRAM的操作封装成一个多口SDRAM，各个端口的操作请求由SDRAM控制器来调度，这样也可以，但是缺点就是灵活性不够。

## OV5640与SDRAM、HDMI与SDRAM之间的数据同步问题

&emsp;&emsp;由于多个端口都可以同时访问SDRAM，导致每个端口在发出读请求后，获得返回数据的时间是不确定的，然而HDMI的行场同步信号是固定的、周期的，所以怎么才能让`FIFO 2`读端口的数据正好和HDMI的行场同步信号匹配？另外摄像头给出的数据也是周期的，怎么才能让数据写道SDRAM中正确的位置？

&emsp;&emsp;最开始想的是将行场同步信号信号写入到FIFO，但这样会导致FIFO利用率大大下降。最后用的方法是利用摄像头和RGB2VGA的场同步信号进行同步，每当场同步信号有效时，对两个FIFO、读地址、写地址均进行重置，这样每帧数据均会进行一次同步，而行同步信号结束有效后，一般会有十几行、几十行的时间，这个时间来作为第一行图像数据的读取时间已经绰绰有余，然后利用HDMI的`de`信号作为FIFO2的读反馈信号，这样便可以保证HDMI端的数据与行场同步信号保持同步。

## 单缓冲、双缓冲、三缓冲之间的区别

&emsp;&emsp;单缓冲就是只用一个缓冲区来缓存图像数据，摄像头往里面写数据，HDMI从里面读数据，这样能行，但是由于摄像头写数据需要时间，可能刚写完半帧数据，然后HDMI就去读，然后读到的数据是上一帧和当前帧混合的数据，即上半帧是当前帧，下半帧是上一帧，当画面变化比较大时，这样会出现明显的撕裂效果，但这样也不是没有好处，好处就是延时减少了，如果帧率不高的话，这样能让画面的响应的时间明显提高。

&emsp;&emsp;双缓冲则是用两个缓冲区缓冲图像数据，当一个缓冲区接受摄像头写入的数据时，另一个缓冲区用于供HDMI读取数据显示，这样HDMI显示的永远是一个完整的帧数据，不会出现画面撕裂效果，但这样会增加一帧的延时，并且当摄像头和HDMI两者之间的速度有略微差别时，会导致帧率明显下降。比如当摄像头帧率比HMDI帧率高一点，当摄像头写完一帧准备读第二帧时，HDMI还没有显示完第一帧，则摄像头就得将第二帧丢弃掉，反之，当HDMI帧率比摄像头高一点时，当HDMI显示完一帧，准备显示第二帧时，却发现摄像头还没有写完第二帧，所以HDMI就不得不再显示一遍第一帧，这两者情况都会导致帧率的下降。

&emsp;&emsp;三缓冲则是为了缓解这个问题，三缓冲利用3个缓冲区来缓冲图像，这样第三个缓冲区就可以再速度存在差异的时候用于缓存一下额外的数据，避免数据丢掉。