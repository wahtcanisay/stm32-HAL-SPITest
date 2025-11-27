# STM32-HAL-SPITest
按钮负责调控板载LED亮灭状态，LED的亮灭状态同时由外部flash保存
## 硬件原理 
![按钮部分接线图](https://github.com/wahtcanisay/stm32-HAL-SPITest/blob/main/%E6%8C%89%E9%92%AE%E9%83%A8%E5%88%86%E6%8E%A5%E7%BA%BF%E5%9B%BE.png)  

## 软件实现

### cubemx部分
首先在`SYS`中`Debug`选择`Serial Wire`  
PC13设置为初始高电平输出开漏模式，PA0选择输入上拉模式。

### keil5部分
本次实验想要做到在按钮松开之后才发生LED状态的变化，需要用到两个变量存储按钮前后的状态。  
```c
uin8_t pre = 1, cur = 1;//按钮状态
uint8_t ledState = 0;//LED状态
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
        }
    }

}
```