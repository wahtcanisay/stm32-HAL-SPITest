# STM32-HAL-SPITest
按钮负责调控板载LED亮灭状态，LED的亮灭状态同时由外部flash保存
## 硬件原理 
![按钮部分接线图](https://github.com/wahtcanisay/stm32-HAL-SPITest/blob/main/%E6%8C%89%E9%92%AE%E9%83%A8%E5%88%86%E6%8E%A5%E7%BA%BF%E5%9B%BE.png)  
![flash部分接线图](https://github.com/wahtcanisay/stm32-HAL-SPITest/blob/main/flash%E9%83%A8%E5%88%86%E6%8E%A5%E7%BA%BF%E5%9B%BE.png)
![flash内部地址图](https://github.com/wahtcanisay/stm32-HAL-SPITest/blob/main/flash%E5%86%85%E9%83%A8%E5%9C%B0%E5%9D%80%E5%9B%BE.png)
向flash写入数据之前要进行擦除，擦除最小单元是扇区(4k字节)   
写入数据的最小单元是页(256字节)，称为页编程  
每次进行擦除和页变成之前要对芯片进行解锁，称为写使能  
## 软件实现

### cubemx部分
首先在`SYS`中`Debug`选择`Serial Wire`    
PC13设置为初始高电平输出开漏模式，PA0选择输入上拉模式。   
在`Connectivity`中选择`SPI1`,`Mode`中选择`Full-Duplex Master`，即全双工模式,`Hardware Nss Signal`选择`Disable`。  
此时在右侧可以看见PA5为SCK引脚，PA6作为MISO引脚，PA7作为MOSI。  
选择PA4作为CS引脚与NSS相连，设置为通用输出推挽模式，手动设置输出高低电压，初始电压设为高电压。  
`SPI1`的基本参数设为`Motorola`,`8bit`,`MSB FirstA`,时钟参数中选择Prescaler：`8`(分频系数为8，由APB2频率除以分频系数得到1MHz)  
CPOL，CPHA分别选择`High`,`2 Edge`，高极性和第二边沿采集即模式3(这个flash模块只支持模式0(0,0)和模式3(1,1))  

### keil5部分
新接口：
```c
HAL_StatusTypeDef HAL_SPI_Transmit(SPI_HandleTypeDef *hspi,
                                   uint8_t *pData,
                                   uint16_t Size,
                                   uint32_t Timeout)
```
作用：向SPI接口发送数据  
参数：  
*hspi：SPI句柄指针  
*pDAta：发送的数据地址  
Size：发送数据数量，以字节为单位  
Timeout：超时时间，单位ms  
返回：数据发送状态   

***注：这里接收到MISO的数据无需在意，直接舍弃***   

```c
HAL_StatusTypeDef HAL_SPI_Receive(SPI_HandleTypeDef *hspi,
                                   uint8_t *pData,
                                   uint16_t Size,
                                   uint32_t Timeout)
```
作用：从SPI从机接口接收数据  
参数：  
*hspi：SPI句柄指针  
*pData：接收缓冲区指针  
Size：接收数据数量，以字节为单位  
Timeout：超时时间，单位ms  
返回：数据接收状态  
***注：这里要给接收缓冲区赋予初值0xff，如果不赋初值，就会把随机值发送给从机，会产生意想不到的效果***  
```c
HAL_StatusTypeDef HAL_SPI_TransmitReceive(SPI_HandleTypeDef *hspi,
                                          uint8_t *pTxData,
                                          uint8_t *pRxData,
                                          uint16_t Size,
                                          uint32_t Timeout)
```
作用：从SPI接口发送数据并接收数据  
参数：  
*hspi：SPI句柄指针  
*pTxData：发送数据地址  
*pRxData：接收数据缓冲区地址  
Size：收发数据数量
Timeout：超时时间  
***注：SPI是收发双向总线，接收和发送的字节数量必须相同***  

  
  
  
本次实验想要做到在按钮松开之后才发生LED状态的变化，需要用到两个变量存储按钮前后的状态。   
```c
static void SaveLEDState(uint8_t ledState);
static uin8_t LoadLEDState(void);
/*以上声明写在main.c的49行Private function prototypes之后的BEGIN(53行)和END之间*/

static void SaveLEDState(uint8_t ledState){
    //向flash中保存LED当前状态函数

    //#1.写使能
    uint8_t writeEnableCmd[] = {0x06};//写使能指令
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_RESET);//选中从机
    HAL_SPI_Transmit(&hspi1, writeEnableCmd, 1, HAL_MAX_DELAY);//发送写使能数据
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_SET);//取消选中

    //#2.扇区擦除
    uint8_t sectorEraseCmd[] = {0x20, 0x00, 0x00, 0x00};//8位擦除指令码 + 24位扇区首地址(擦除零号扇区)
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_RESET);//选中从机
    HAL_SPI_Transmit(&hspi1, sectorEraseCmd, 4, HAL_MAX_DELAY);//发送擦除指令
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_SET);//取消选中
    HAL_Delay(100);//等待擦除指令完成

    //#3.写使能
    uint8_t writeEnableCmd[] = {0x06};//写使能指令
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_RESET);//选中从机
    HAL_SPI_Transmit(&hspi1, writeEnableCmd, 1, HAL_MAX_DELAY);//发送写使能指令
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_SET);//取消选中

    //#4.页编程
    uint8_t pageProgCmd[5] = {0x02, 0x00, 0x00, 0x00, 0x00};//8位页编程指令码+24位地址+要写入数据(LED状态)
    pageProgCmd[4] = ledState;//写入LED状态
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_RESET);//选中从机
    HAL_SPI_Transmit(&hspi1, pageProgCmd, 5, HAL_MAX_DELAY);//发送写入数据指令
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_SET);//取消选
}

static uin8_t LoadLEDState(void){
    //读取LED状态
    uint8_t readDataCmd[] = {0x03, 0x00, 0x00, 0x00};//8位接收数据指令码+24位地址+要写入数据(LED状态)
    uint8_t ledState = 0xff;//接收LED状态缓冲区
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_RESET);//选中从机
    HAL_SPI_Transmit(&hspi1, readDataCmd, 4, HAL_MAX_DELAY);//发送读数据指令
    HAL_SPI_Receive(&hspi1, &ledState, 1, HAL_MAX_DELAY);//读出LED状态数据
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_SET);//取消选中

    return ledState; //返回LED状态
}
/*以上函数实现写在Private user code之后的BEGIN 0和END0之间*/


uin8_t pre = 1, cur = 1;//按钮状态
uint8_t ledState = 0;//LED状态

ledState = LoadLEDState(); //读取flash中存储的LED状态
if(ledState == 1){
    //读取到1点亮LED
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_13, GPIO_PIN_RESET);
}
else{
    //读取到0熄灭LED
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_13, GPIO_PIN_SET);
}
while(1){

    pre = cur;
    if(HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_SET){
        //按钮未被按下
        cur = 1;
    }
    else{ 
        //按钮被按下
        cur = 0;
    }
    
    if(pre != cur){
        //前后状态不相等，需要进一步判断到底是处于按下还是松开的瞬间
        HAL_Delay(10);//按键消抖
        if(cur == 0){
            //处于按下不做调整
        }
        else{
            //处于松开，调整LED状态
            if(ledState == 1){
                //LED此时处于点亮状态，熄灭LED并将状态置0
                HAL_GPIO_WritePin(GPIOA, GPIO_PIN_13, GPIO_PIN_SET);
                ledState = 0;
            }
            else{
                //与上相反
                HAL_GPIO_WritePin(GPIOA, GPIO_PIN_13, GPIO_PIN_RESET);
                ledState = 1;
            }

            SaveLEDState(ledState);//切换后保存LED状态
        }
    }

}
```
**注意：**  
**断电之后再上电可能出现单片机初始化过快，从机flash芯片外设初始化较慢导致读不出数据的问题**  
**可以在开始运行多次读LED状态，或者开始使用模式0(高极性、第一边沿采集)，或者在初始化之后加入迟延**  
