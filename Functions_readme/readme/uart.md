# uart with IP

## Brief Introduction

设计上数据打算通过AXI4总线来传输，需要设计有对应AXI4接口的串口IP核。在XILINX的A7系列下有自带支持AXI4的IP核【AXI_Uartlite】，因此主要工作转变为：

1. 识别从AXI4数据总线接收来的串口指令，发送给【AXI_Uartlite】
2. 识别从【AXI_Uartlite】返回的串口数据，从AXI4数据总线发出
3. 控制【AXI_Uartlite】

![uart_diagram](https://github.com/Lvwings/FiberAcquisitionCard/blob/feature/Image_update/Functions_readme/image/uart/uart_diagram.PNG?raw=true)

AXI接口采用自封装IP ，两个IP目前基于AXI_Lite，采用单次传输

【SLAVE】 为总线接口，接收总线数据，返回参数

<img src="https://github.com/Lvwings/AXI-Interface/blob/master/image/SLAVE.PNG?raw=true" alt="IP" style="zoom:67%;" />

【MASTER】为【AXI_Uartlite】通信接口，控制串口，发送和读取串口数据；输入【MASTER】的数据和地址需要保持至少两个时钟有效

<img src="https://github.com/Lvwings/AXI-Interface/blob/master/image/MASTER.PNG?raw=true" alt="MASTER" style="zoom:67%;" />

而控制部分由DATA模块来实现，主要有如下几个功能：

1. 初始化串口
2. 串口读模式控制
3. 串口写模式控制
4. 返回数据识别

## Register Definition

目前在总线上给串口分配的地址为：0x100D0000 - 0x100D0FFF 共计4K

而在【SLAVE】模块中，使能了16个寄存器用于存储接收数据，总线数据为32位，占用地址为:0x100D0000 - 0x100D003C , 目前使用到的寄存器设置如下：

| 写寄存器地址 | 含义                                                 |
| ------------ | ---------------------------------------------------- |
| 0x100D0000   | 用于复位参数使用，不作为数据传输                     |
| 0x100D0004   | 设置工作方式，32位数据对应【Module，M_Mode，M_para】 |
| 0x100D0008   | 读取全部参数，当32为数据为0xAAAAAAAA时发送读取指令   |

（一个地址存储一个字节8-bit数据，因此一个总线数据占用4个地址）

同样在【SLAVE】模块中，使能了16个寄存器用于存储上传的参数，总线数据为32位，占用地址为:0x100D0000 - 0x100D003C , 目前使用到的寄存器设置如下：

| 读寄存器地址 | 含义                             |
| ------------ | -------------------------------- |
| 0x100D0000   | 用于复位参数使用，不作为数据传输 |
| 0x100D0004   | 参数RAMAN1                       |
| 0x100D0008   | 参数RAMAN2                       |
| 0x100D000C   | 参数EDFA                         |

## 【AXI_Uartlite】

【AXI_Uartlite】为xilinx自带的IP核，通过AXI_LITE接口进行控制和回传数据，UART接口作为外部接口，interrupt提供数据状态。

而interrupt有效的两种情况：

1. 接收FIFO有数据

2. 发送FIFO为空

通过读取串口寄存器状态可以区分这两种情况，串口IP内部寄存器地址映射如下：

![register_address](https://github.com/Lvwings/FiberAcquisitionCard/blob/feature/Image_update/Functions_readme/image/uart/register_address.png?raw=true)

## Initialization

初始化最主要的功能是：

1. 使能interrupt功能

2. 初始化TX_FIFO和RX_FIFO

需要对控制寄存器（0x0C）写入控制字0000_0013，具体含义如下：

![control_register](https://github.com/Lvwings/FiberAcquisitionCard/blob/feature/Image_update/Functions_readme/image/uart/control_register.png?raw=true)

这部分代码很好实现，后续的输出赋值基本采用如下方式：

```verilog
	if (time_cnt < time_valid) begin	// 2 clk					
        time_cnt		<= time_cnt + 1;
        wr_addr 		<= CTRL_REG;
        wr_addr_valid 	<= 1;	
        wr_data 		<= CMD_INIT;
        wr_data_valid 	<= 1;
    end
```

在下发初始化的指令后，需要查询是否设置成功，需要对状态寄存器（0x08）进行读操作，具体含义如下：

![state_register](https://github.com/Lvwings/FiberAcquisitionCard/blob/feature/Image_update/Functions_readme/image/uart/state_register.png?raw=true)

由于查询指令在写/读/初始化都需要用到，我将其拉出根据状态机状态进行赋值：

```verilog
	...
// 初始化结果查询、写数据前查询写FIFO | interrupt有效类型判断 
else if (tx_current_state == TX_STAT || rx_current_state == RX_STAT) begin										
        if (time_cnt_rd < time_valid) begin
            time_cnt_rd		<= time_cnt_rd + 1;
            rd_addr 		<= STAT_REG;
            rd_addr_valid 	<= 1;
        end
        else begin
            time_cnt_rd		<= 0;
            rd_addr 		<= 0;
            rd_addr_valid 	<= 0;						
        end	
    end
```

## Read Mode

读模式主要由interrupt触发，但是interrupt有两个触发条件：

1. RX_FIFO中有数据

2. TX_FIFO中为空

因此需要读状态寄存器（0x08）来判断是否为有数据进入。

![RX_fifo](https://github.com/Lvwings/FiberAcquisitionCard/blob/feature/Image_update/Functions_readme/image/uart/RX_fifo.png?raw=true)

当查询到 RX FIFO 有数据时，即可对RX FIFO进行读取，查询RX FIFO寄存器（0x00）

```verilog
// 读数据查询读FIFO
if (rx_current_state == RX_REQ) begin		
        if (time_cnt_rd < time_valid) begin
            time_cnt_rd		<= time_cnt_rd + 1;
            rd_addr 		<= RX_REG;
            rd_addr_valid 	<= 1;
        end
        else begin
            time_cnt_rd		<= 0;
            rd_addr 		<= 0;
            rd_addr_valid 	<= 0;						
        end					
    end
```

之后等待`READ_DATA_VALID`有效即可读到数据。

```verilog
    RX_DATA : begin
        if (READ_DATA_VALID) begin
            rx_data <= READ_DATA[7:0];
            rd_over		<=	1'b1;
    	end	
    	...
```

读模式下时序如下所示

![read_sequence](https://github.com/Lvwings/FiberAcquisitionCard/blob/feature/Image_update/Functions_readme/image/uart/read_sequence.PNG?raw=true)

## Write Mode

写模式主要有两个触发条件：

1. 总线下发查询全部参数或设置工作方式
2. 周期查询指令（4s）

写模式有效由`uart_tx_start`控制，其实现方式如下：

```verilog
case (DOWNLOAD_DATA_VALID)
    // 设置工作方式
    4'h1   : begin		
        if (tx_over) begin	 
            uart_tx_start	<= 1;// 在发送数据过程中，新命令可能会覆盖旧指令
            cmd_type		<= SET_PARAM_CMD;
            SET_PARAM		<= dw_reg1;
        end
        else begin
            uart_tx_start	<= 0;
            cmd_type		<= cmd_type;
            SET_PARAM		<= SET_PARAM;
        end					
    end
    // 查询全部参数
    4'h2   : begin
        if (tx_over && (dw_reg2 == READ_PARAM)) begin
            uart_tx_start	<= 1;
            cmd_type		<= READ_PARAM_CMD;
        end
        else begin
            uart_tx_start	<= 0;
            cmd_type		<= cmd_type;					
        end
    end
    default : begin
        // 周期查询指令
        if (time_auto_rd_cnt == TIME_4S) begin
            uart_tx_start	<= 1;
            cmd_type		<= READ_PARAM_CMD;							
        end
        else begin
            uart_tx_start	<= 0;
            cmd_type		<= cmd_type;
            SET_PARAM		<= SET_PARAM;
        end
    end
endcase
```

由于读模式和写模式都会占用AXI的读地址通道`rd_addr`，这里设计优先级 读 > 写，当读模式占用读通道`rd_addr`时，写模式进入等待直到通道被释放：

```verilog
    TX_IDLE : begin
        ...
        else if (uart_tx_start && rd_busy)	// 读地址被占用
            tx_next_state = TX_WAIT;
        ...
	TX_WAIT : begin
        if (rd_busy)
            tx_next_state = TX_WAIT;
        else
            tx_next_state = TX_STAT;		// 查询状态寄存器
    end	    
```

另外由于AXI总线的时钟速率远高于UART时钟，而TX_FIFO的深度为16，因此在写命令时可能会将TX_FIFO写满。因此在每次写数据前，都需要对TX_FIFO进行读取，如果TX FIFO为满，放弃本次写入，重新查询状态寄存器直到TX FIFO为非满。

```verilog
    TX_RD : begin							// 判断TX FIFO 
        if (tx_one && tx_fifo)				// TX FIFO 为满
            tx_next_state = TX_STAT;	
        ...
```

在串口的通讯格式中，最后一字节可以选择校验方式：odd（奇校验）even（偶校验）none（不校验）。

在计算校验值时要注意时钟，每个数据应只被计算一次。

大致流程图如下所示：

![write_mode](https://github.com/Lvwings/FiberAcquisitionCard/blob/feature/Image_update/Functions_readme/image/uart/write_mode.PNG?raw=true)

设置工作方式写时序如下：

![write_sequence](https://github.com/Lvwings/FiberAcquisitionCard/blob/feature/Image_update/Functions_readme/image/uart/write_sequence.PNG?raw=true)

对应TX FIFO 满时的时序如下：

![writefifo](https://github.com/Lvwings/FiberAcquisitionCard/blob/feature/Image_update/Functions_readme/image/uart/writefifo.PNG?raw=true)

## Judge Mode

通常来说，串口通信为了保证数据被正确识别，除了每个数据最后的校验值，还会设置几个数据作为首部标志用以告知数据开始。

这些数据可能需要进行一定处理才上传，这里使用了两个数据构成首部标志：

```verilog
    JUDG_IDLE : begin
        if ({rx_data_d,rx_data} == JUDG_FLAG)
            judg_next_state = JUDG_CMD;
        else
            judg_next_state = JUDG_IDLE;
        ...
```

# uart without IP

## Brief Introduction

串口通信中一般使用的是串口异步通信，收发双方取得同步是通过在字符格式中设置起始位和停止位的方法来实现的。具体来说就是，在一个有效字符正式发送之前，发送器先发送一个起始位，然后发送有效字符位，在字符结束时再发送一个停止位，起始位至停止位构成一个数据帧：

![uart_frame](https://github.com/Lvwings/FiberAcquisitionCard/blob/feature/Image_update/Functions_readme/image/uart/uart_frame.png?raw=true)

在带有校验位的数据帧中，如果数据采用8位，那么整个数据帧占用11码元。

当采用不同调制方式时，可以在一个码元上负载多个bit位信息（例如曼切斯特编码，1码元=3bit）。而在单片机或FPGA中一般没有采用编码调制，这里1码元=1bit，因此有：

​    波特率9600 = 9600（码元/秒）= 9600 （位/秒）

​    有效数据的字节数率（11码元带有8位有效数据）：

​    波特率9600 = 9600 （位/秒）= 872.72（字节/秒）=0.85（KB/秒）

那么要驱动特定波特率的时钟：
$$
F_{clk} = Baud*16*div
$$
​	Baud是驱动目标波特率；16分频用于保证接收的准确性，防止抖动；div是额外分频系数，最小值为1。

## Read Mode

串口数据帧通过起始位来标明帧的开始，在读数据时通过判断RX的下降沿即可得知数据的到来，为了实现这个目的，运行时钟频率应高于数据频率，也就有了上文提到的需要16分频。

```verilog
    case(current_state)
        IDLE: begin
            if({rxd,rxd_d} == 2'b01) begin      // 遇到起始位
                next_state = START;
            end
            ...
```

在数据读取的过程中，为了保证接收数据的稳定性，数据应从的中间读取，避免数据边沿带来的读取误差。

```verilog
	if(count == 7) begin						// 时钟计数16分频中间位
        rx_data[data_cnt] <= rxd;
        rx_1_sum <= rx_1_sum + rxd;				// 校验位计算
    end
```

在接收完数据后，需要校验数据的正确性（通过校验位），以及数据帧的正确性（通过停止位）。

```verilog
	if((rx_1_sum[0] != rxd) & (count == 7)) begin       // 校验位
    	parity_err <= 1;
    end
	...
	if(!rxd && (count == 7)) begin				// 停止位
        framing_err <= 1;
    end
```

考虑到停止位可能为1.5位，在设计停止位校验时需要注意位数；同时为了给下一位读取判断保留时间，停止位判断结束要小于16个时钟。

## Write Mode

由于是异步读写，在写模式下数据操作会更简单。

在确定写数据后，按照串口的数据帧格式，依次将数据输出，注意输出默认值为1即可。

# summary

实际上在uart without IP中是实现【AXI_Uartlite】功能的粗糙版本，适用于理解串口的数据帧格式。使用【AXI_Uartlite】的好处在于：

1. 不需要去做帧格式的处理
2. 集成了输入输出缓存FIFO
3. 有合理的反馈机制