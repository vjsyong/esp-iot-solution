[[中文]](./readme_cn.md)

# ESP32 ULP SPI Example
This document will demonstrate how to bitbang an SPI interface to communicate and retrieve data from an MS5611 sensor with as little power consumption as possible

## 1. Hardware Overview
The ULP(Ultra Low Power Co-processor) onboard the ESP32 does have a hardware SPI interface. This example will illustrate how to use RTC GPIO to emulate the SPI interface (Supports 4 types of SPI Modes), then use the emulated interface to retrieve data from a MS5611 sensor module 

The MS5611 is a high-precision barometric sensor IC. According to the specifications, it contains a 24bit ADC and is accurate to 10cm at 25 degrees celsius with a precision of ±1.5mbar. This module provices both I2C and SPI interfaces, which can be selected through the PS pin. When the PS pin is pulled high, it is configured in I2C mode; Conversely, when the PS pin is pulled low, it is configured in SPI mode. The MS5611 module supports both SPI Mode 0 and Mode 3. In this example, we will use SPI mode 0.
#### 1.1 Sensor Module
![](../../../documents/_static/ulp_spi/GY-63-MS5611.jpeg)

#### 1.2 Wiring Diagram
![](../../../documents/_static/ulp_spi/sch.png)

#### 1.3 SPI Pin Connections
|NUM|GPIO|SPI|MS5611|
|---|---|---|---|
|1|GPIO_25|SPI_CS|CSB|
|2|GPIO_26|SPI_MOSI|SDA|
|3|GPIO_27|SPI_SCLK|SCL|
|4|GPIO_4|SPI_MISO|SDO|

## 2 Software Overview
本例程設計到的相關彙編宏和彙編函數眾多，歸納在下面的三大表格中。具體彙編代碼就不貼上來了，請對照表格中的函數說明閱讀彙編代碼，便於理解。
This example application utilizes many assembly related commands and macros, which will be listed in the tabled below. The full assembly reference won't be shown here. Please use the list below to retrieve the mapped macro.

#### 2.1 RTC GPIO 宏
|NUM|Assembly Macro|Assembly Function|
|---|---|---|
|1|read_MISO| Read MOSI Pin |
|2|clear_SCLK| Set SCLK LOW |
|3|set_SCLK| Set SCLK HIGH |
|4|clear_MOSI| Set MOSI LOW |
|5|set_MOSI| Set MOSI HIGH |
|6|clear_CS| Set CS LOW |
|7|set_CS| Set CS HIGH |

#### 2.2 彙編函數
|NUM|Assembly Function|Inputer Parameters|Return Parameters|Remarks|
|---|---|---|---|---|
|1|SPI_Init|SPI_MODE_SET| - |Initialize RTC GPIO，Set SCLK and MOSI Pin values according to SPI Mode|
|2|CS_​​Disable| - | - | To preserve uniform API styling, this is wrapped in a macro |
|3|CS_Enable| - | - | Macro to call clear_CS |
|4|SPI_Write_Byte|R2| - | 支持SPI 的4 種模式的寫操作，默認是寫一個字節(8bit), 若使能宏SPI_BIT16，可支持16bit 寫操作|
|5|SPI_Read_Byte| - |R2| 默認讀一個字節(8bit), 使能宏SPI_BIT16 後，可支持16bit 讀操作，支持SPI 的4 種模式讀操作 |
|6|waitMs|R2| - |延遲函數，作用是延遲1ms，參數由R2 傳入 |
|7|MS5611_Init| CMD_RESET| - |初始化SPI ， 初始化MS5611 傳感器，具體是發送RESET 命令（0x1E），然後延遲3ms 等待傳感器重載 |
|8|MS5611_Save_Data|addr_pointer，prom_table| addr_pointer | Base_Addr（prom_table） + Offset_addr（addr_pointer） 模式，把讀到的傳感器RROM 數據存入內存表中，最後更新addr_pointer 值 |
|9|MS5611_Read_PROM|PROM_NB，CMD_PROM_RD| - |讀MS5611 PROM，地址為（0xA0,0xA2,0xA4,0xA6,0xA8,0xAA,0xAC,0xAE）共8 x 16bit 的數據|
|10|MS5611_Convert_D1|CMD_ADC_D1_4096， CMD_ADC_READ| D1_H，D1_L|Convert D1, 設置OSR = 4096, 返回24bit 的數據，讀到的數據保存在D1_H, D1_L 中|
|11|MS5611_Convert_D2|CMD_ADC_D2_4096，CMD_ADC_READ|D2_H，D2_L|Convert D2, 設置OSR = 4096, 返回24bit 的數據，讀到的數據保存在D2_H, D2_L 中|

#### 2.3 棧管理宏
|NUM|彙編宏|說明|
|---|---|---|
|1|push|這個宏是把Rx（R0，R1，R2）的數據壓到R3 所指向的棧裡，棧指針向下增長|
|2|pop|POP 宏是把R3 所指向的棧裡的數據保存到Rx（R0,R1,R2）中，棧指針回退|
|3|psr|PSR 宏是計算出函數執行完畢後需要返回的地址，將其保存到棧上，即當前地址加上偏移+16 就是子函數跳轉執行完畢後需要返回的地址|
|4|ret|此宏是把棧上保存的返回地址取出來，然後跳轉到這個地址|
|5|clear|此宏作用是將變量賦值清零，reset 作用|

## 3 讀MS5611 傳感器
讀取MS5611 傳感器，大致分為四步驟。第一步，RESET 芯片；第二步，讀取PROM 中的校準數據；第三步，啟動AD 讀取溫度和壓力的RAW 數據；第四步，根據校準值、溫度和壓力的RAW 值計算出真實壓力和溫度值。
#### 3.1 RESET MS5611
![](../../../documents/_static/ulp_spi/S1.png)
#### 3.2 READ PROM
![](../../../documents/_static/ulp_spi/S2.png)
#### 3.3 ADC D1,D2
|Digital pressure value(D1)|Digital temperature value(D2)|
|---|---|
|![](../../../documents/_static/ulp_spi/S3.png)|![](../../../documents/_static/ulp_spi/S4.png)|
#### 3.4 Calculate temperature and compensated pressure
![](../../../documents/_static/ulp_spi/S5.png)

## 4 總結
本例ULP 協處理器讀MS5611 編寫過程中，發現當前的傳感器精度越來越高，但是對計算的需求也越來越大（本例是有符號的64bit乘除運算），這樣給沒有乘除法的ULP 協處理器帶來了很大的困難，本例子是喚醒CPU 進行計算處理並打印出信息的。
 

