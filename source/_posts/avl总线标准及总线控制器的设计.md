---
title: avl总线标准及总线控制器的设计
typora-copy-images-to: avl总线标准及总线控制器的设计
categories:
  - FPGA
tags:
  - FPGA
  - 总线控制器
typora-root-url: ./avl总线标准及总线控制器的设计
abbrlink: 17968
date: 2020-06-30 23:37:58
---

&emsp;&emsp;总线控制器负责调度各个主机的访问请求，总线控制器提供多个标准的、统一的主、从机接口，使得多个主机与多个从机之间可以使用统一的接口时序进行通讯，提高了系统的可扩展性，又因为总线控制器可以协调多个主机的访问请求，使得对于每个主机都可以访问任意的从机，提高了系统的灵活性。

![39578544_p4](/39578544_p4.jpg)

<!-- more -->

# 总线介绍

&emsp;&emsp;由于AXI、AHB标准复杂，实现一个完整的控制器难度较大，而avalon、ICB总线标准易于实现，且功能正好，所以本次设计的总线时序参考avalon、ICB总线时序，并结合实际情况制定了总线的操作时序，并命名为AVL总线。

&emsp;&emsp;总线端口分为主机端口和从机端口，主机端口于从机端口之间通过两个独立的通道进行通信：命令通道、数据通道。主机的读/写请求以命令的方式通过命令通道发送出去，从机反馈的数据则通过返回通道返回给主机，这样分离了命令与反馈，让主从机之间通信更加简单明了。

![image-20200701000419265](/image-20200701000419265.png)



AVL总线有如下特点：

* 只支持地址对齐访问。

* 支持插入寄存器级，提高最高运行速度。

* 支持滞外交易。

* 支持流水线传输。

* 支持突发传输，最大突发传输数量为256个字。

* 所有请求的结果顺序返回、顺序完成。

* 每次读写都会发出地址。

* 采用ready-valid式握手规则。

* 支持固定优先级总裁或轮询仲裁。

# 端口定义

AVL总线端口定义如下表所示。

![image-20200701000803939](/image-20200701000803939.png)

在System verilog中定义如下所示

```verilog
interface i_avl_bus;
  logic[31:0] address;
  logic[3:0]  byte_en;
  logic       read;
  logic       write;
  logic[31:0] write_data;
  logic       begin_burst_transfer;
  logic[7:0]  burst_count;
  logic       request_ready;
  logic[31:0] read_data;
  logic       read_data_valid;
  logic       resp_ready;
  modport master(
    output address,byte_en,read,write,write_data,begin_burst_transfer,burst_count,resp_ready,
    input  request_ready,read_data,read_data_valid
  );
  modport slave(
    input  address,byte_en,read,write,write_data,begin_burst_transfer,burst_count,resp_ready,
    output request_ready,read_data,read_data_valid
  );
  modport monitor(
    input  address,byte_en,read,write,write_data,begin_burst_transfer,burst_count,resp_ready,request_ready,read_data,read_data_valid
  );
endinterface
```



# 总线时序

&emsp;&emsp;下面讲介绍总线典型的时序

 ![AVL总线读时序](/AVL%E6%80%BB%E7%BA%BF%E8%AF%BB%E6%97%B6%E5%BA%8F.bmp)

&emsp;&emsp;上图为总线的读时序，其中包含了4次读请求，下面详细介绍：

* 第2个时钟周期时，来自主机的地址、字节使能、读使能信号出现再总线上。

* 第3个上升沿时读请求被从机锁存，request_ready信号被拉高，表示从机接受了这个命令。第3个上升沿过后，返回数据出现在了总线上。

* 第4个上升沿时read_data_valid、resp_ready为高，表示当前返回数据有效，且主机接受了这个数据。第4个上升沿过后，地址A1的读请求出现在了总线上。

* 第5个上升沿时request_ready为高，表示从机接受了这个读请求，但是第五个上升沿后，read_data_valid信号没有变为1，表示从机无法立即返回数据。

* 第6个上升沿后，返回数据出现在了总线上。

* 第7个上升沿时，主机接收了从机返回的数据。第7个上升沿后，读地址A2的请求出现在了总线上。

* 第8个上升沿时，request_ready信号为0，这说明从机当前无法接受新的请求，主机的请求还得在总线上继续保持。

* 第9个上升沿时，request_ready信号为1，说明从机接受了读请求，第9个上升沿后，读地址A3的读请求发出，地址A2的数据被返回。

* 第10个上升沿时，从机接受了地址A3的读请求，并返回了地址A3的数据。主机接受了地址A2的数据。

* 第11个上升沿时，resp_ready信号为0，这说明主机正忙，无法接受从机返回的数据，所以从机返回的数据得继续保持。

* 第12个上升沿时，resp_ready信号为1，主机接受了从机返回的数据。至此，4次读操作完成。

 ![AVL总线写时序](/AVL%E6%80%BB%E7%BA%BF%E5%86%99%E6%97%B6%E5%BA%8F.bmp)

&emsp;&emsp;上图为总线的写时序，其中包含了4次写操作。

* 第2个上升沿时，主机给出了写请求。

* 第3个上升沿时，request_ready为1，表示从机接受了此次写请求。

* 第4个上升沿时，主机给出了地址A1的写请求。

* 第5个上升沿时，从机接受了地址A1的写请求，主机同时给出了地址A2的写请求。

* 第7个上升沿时，request_ready信号为低，表示从机未接受地址A3的写请求，主机需要继续保持地址A3的写请求。

* 第8个上升沿时，从机接受了地址A3的写请求。至此，5次写操作完成。

 ![AVL总线突发读时序](/AVL%E6%80%BB%E7%BA%BF%E7%AA%81%E5%8F%91%E8%AF%BB%E6%97%B6%E5%BA%8F.bmp)

&emsp;&emsp;对比常规读和突发读时序可以发现，突发读时序相对于常规读时序多了begin_burst_transfer和burst_count信号，且地址变为了连续的4个地址，即突发传输规定，在主机发出第一个请求时，begin_burst_transfer信号需要被拉高，并通过burst_count给出突发传输数量，其值等于突发传输数量减1，且接下来的地址信号每次必须递增4。

&emsp;&emsp;由于突发读传输属于典型读请求的一个特例，所以当从机不支持突发传输时，可以忽略begin_burst_transfer和burst_count信号，按照典型的读请求来处理，从而简化硬件设计，对于支持突发传输的从机，则可以通过这两个信号得知接下来的若干个读请求是批量、顺序的读请求，从而可以对访问效率进行优化。

 ![AVL总线突发写时序](/AVL%E6%80%BB%E7%BA%BF%E7%AA%81%E5%8F%91%E5%86%99%E6%97%B6%E5%BA%8F.bmp)

&emsp;&emsp;上图为总线突发写时序，与突发读传输类似，突发写传输也是典型写请求的一个特例，对于不支持突发传输的从机，可以将突发写请求当作普通的写请求处理。而对于支持突发传输的从机，则可以对突发传输进行速度优化。

# 硬件实现

&emsp;&emsp; 下图为总线控制器的结构框图

![总线控制器](/%E6%80%BB%E7%BA%BF%E6%8E%A7%E5%88%B6%E5%99%A8.png)

&emsp;&emsp;上半部分是多主一总线控制器模块，其主要由多路选择器、分发器、仲裁器、FIFO组成。仲裁器负责检查总线上的访问请求，并决定当前哪一个主机可以访问总线，仲裁器支持固定优先级仲裁、轮询仲裁两种，当采用固定优先级仲裁时，主机端口编号越小，优先级越高，如果多个主机同时发出访问请求，则优先让优先级高的主机访问，如果采用轮询优先级，则每发出一次请求，发出请求的主机优先级将为最低，然后继续按端口标号从低到高寻找是否有其它主机发出访问请求。每次发出一个请求后，多主一从模块会利用FIFO保存发送请求的主机的编号，并作为返回通道分发器的选择信号，由于FIFO的存在，使得多主一从模块支持滞外交易，FIFO的深度决定了滞外交易的最大数量。

&emsp;&emsp;下半部分为多从一主总线控制器模块的结构框图，与多主一从类似，其主要由多路复用器、地址映射模块、FIFO、分发器模块组成，地址映射模块主要负责确定地址与端口的对应关系，然后控制分发器将命令分发给正确的从机，命令发送完后，多从一主模块会利用FIFO将命令分发的从机号保存住，用于控制反馈通道的多路复用器，选择正确的从机，FIFO的深度决定了滞外交易的最大数量。

```verilog
module avl_bus_12n #(
  parameter     SLAVE_NUM                     = 4,
                SEL_FIFO_DEPTH                = 4,
            int ADDR_MAP_TAB_FIELD_LEN[0:31]  = '{32{32'd22}},
            int ADDR_MAP_TAB_ADDR_BLOCK[0:31] = '{
                            {22'd01,10'd0},{22'd02,10'd0},{22'd03,10'd0},{22'd04,10'd0},
                            {22'd05,10'd0},{22'd06,10'd0},{22'd07,10'd0},{22'd08,10'd0},
                            {22'd09,10'd0},{22'd10,10'd0},{22'd11,10'd0},{22'd12,10'd0},
                            {22'd13,10'd0},{22'd14,10'd0},{22'd15,10'd0},{22'd16,10'd0},
                            {22'd17,10'd0},{22'd18,10'd0},{22'd19,10'd0},{22'd20,10'd0},
                            {22'd21,10'd0},{22'd22,10'd0},{22'd23,10'd0},{22'd24,10'd0},
                            {22'd25,10'd0},{22'd26,10'd0},{22'd27,10'd0},{22'd28,10'd0},
                            {22'd29,10'd0},{22'd30,10'd0},{22'd31,10'd0},{22'd32,10'd0}
                          }
)(
  input  logic     clk,
  input  logic     rest,
  i_avl_bus.slave  avl_in,
  i_avl_bus.master avl_out[SLAVE_NUM-1:0]
);
/*Code*/
endmodule
```



&emsp;&emsp;一主多从模块System Verilog定义如上面的代码所示，其接受4个配置参数，其中有三个参数与地址映射有关：`SLAVE_NUM`、`ADDR_MAP_TAB_FIELD_LEN`、`ADDR_MAP_TAB_ADDR_BLOCK`，`SLAVE_NUM`用于指定从机数量，`ADDR_MAP_TAB_FIELD_LEN`与`ADDR_MAP_TAB_ADDR_BLOCK`分别为1个32bit宽度，32bit长度的数组，组合起来为一个地址映射表，用于指定地址与从机端口的映射关系。以上面的代码为例，其中`SLAVE_NUM`值默认为4，说明该模块为一个1主机4从机的总线控制器，所以我们只需要关心`ADDR_MAP_TAB_FIELD_LEN`与`ADDR_MAP_TAB_ADDR_BLOCK`中前4个数即可，它们的值如下表所示。

| Index | `ADDR_MAP_TAB_FIELD_LEN` | `ADDR_MAP_TAB_ADDR_BLOCK` |
| :---: | :----------------------: | :-----------------------: |
|   0   |            22            |        0x00000400         |
|   1   |            22            |        0x00000800         |
|   2   |            22            |        0x00000C00         |
|   3   |            22            |        0x00001000         |

&emsp;&emsp;`ADDR_MAP_TAB_FIELD_LEN`中的值均为22，这表明地址映射模块进行地址映射时只会比较地址的高22bit，如果主机发出请求的地址与地址映射表中某个元素的值相同，那就表示访问的地址对应的从机端口号就是那个元素在数组中的下标。例如上表中，`ADDR_MAP_TAB_FIELD_LEN`第0个元素值为22，`ADDR_MAP_TAB_ADDR_BLOCK`值为0x00000400，这表明从机端口0对应的地址区间为0x00000400-0x000007FF(高22bit相同)。这便是地址映射实现的方法，模块在收到这个参数够，根据这些参数生成对应的电路即可实现地址映射功能。