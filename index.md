## 串口发送/接收函数

```
HAL_UART_Transmit();串口发送数据，使用超时管理机制
HAL_UART_Receive();串口接收数据，使用超时管理机制
HAL_UART_Transmit_IT();串口中断模式发送
HAL_UART_Receive_IT();串口中断模式接收
HAL_UART_Transmit_DMA();串口DMA模式发送
HAL_UART_Transmit_DMA();串口DMA模式接收
```

#### 串口发送数据

```
HAL_UART_Transmit(UART_HandleTypeDef *huart, uint8_t *pData, uint16_t Size, uint32_t Timeout)
```

- **串口扫描接收数据**

```
uint8_t data=0;
while (1)
{
    //串口接收数据
    if(HAL_UART_Receive(&huart1,&data,1,0)==HAL_OK){
            //将接收的数据发送
             HAL_UART_Transmit(&huart1,&data,1,0);
        }
}

```

#### 中断接收数据

```
HAL_UART_Receive_IT(UART_HandleTypeDef *huart, uint8_t *pData, uint16_t Size)
```

- 大致过程是，设置数据存放位置，接收数据长度，然后使能串口接收中断。接收到数据时，会触发串口中断。
  再然后，串口中断函数处理，直到接收到指定长度数据，而后关闭中断，进入中断接收回调函数，不再触发接收中断。(只触发一次中断)

**_2. 串口中断函数_**

```
HAL_UART_IRQHandler(UART_HandleTypeDef *huart);  //串口中断处理函数
HAL_UART_TxCpltCallback(UART_HandleTypeDef *huart);  //串口发送中断回调函数
HAL_UART_TxHalfCpltCallback(UART_HandleTypeDef *huart);  //串口发送一半中断回调函数（用的较少）
HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart);  //串口接收中断回调函数
HAL_UART_RxHalfCpltCallback(UART_HandleTypeDef *huart);//串口接收一半回调函数（用的较少）
HAL_UART_ErrorCallback();串口接收错误函数

```

**_3. 串口接收中断回调函数_**

```
HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart);
```

- 功能: HAL 库中断进行后,不会直接退出,而是进入到该中断回调函数,用户可以在中设置代码,`串口中断接收后,进入该函数`.

**_4. 串口中断处理函数_**

```
HAL_UART_IRQHandler(UART_HandleTypeDef *huart);
```

- 功能:对接收数据进行判断,(判断是发送中断还是接收中断),然后进行数据的发送和接受,在终端函数中使用该函数.

- **接收数据**

```
 /* UART in mode Receiver ---------------------------------------------------*/
  if((tmp_flag != RESET) && (tmp_it_source != RESET))
  {
    UART_Receive_IT(huart);
  }
```

- **发送数据**

```
  /* UART in mode Transmitter ------------------------------------------------*/
  if (((isrflags & USART_SR_TXE) != RESET) && ((cr1its & USART_CR1_TXEIE) != RESET))
  {
    UART_Transmit_IT(huart);
    return;
  }
```

**_5. 串口查询函数_**
HAL_UART_GetState(); 判断 UART 的接收是否结束，或者发送数据是否忙碌

```
while(HAL_UART_GetState(&huart4) == HAL_UART_STATE_BUSY_TX)   //检测UART发送结束
```

#### 重写 printf 函数

- _#include<stdio.h>_

```
/**
  * 函数功能: 重定向c库函数printf到DEBUG_USARTx
  * 输入参数: 无
  * 返 回 值: 无
  * 说    明：无
  */
int fputc(int ch, FILE *f)
{
  HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, 0xffff);
  return ch;
}

/**
  * 函数功能: 重定向c库函数getchar,scanf到DEBUG_USARTx
  * 输入参数: 无
  * 返 回 值: 无
  * 说    明：无
  */
int fgetc(FILE *f)
{
  uint8_t ch = 0;
  HAL_UART_Receive(&huart1, &ch, 1, 0xffff);
  return ch;
}
```

```flow
   start=>start: HAL_UART_Receive_IT(中断接收函数)
   a=>operation: USART2_IRQHandler(void)(中断服务函数)
   b=>operation: HAL_UART_IRQHandler(UART_HandleTypeDef *huart)(中断处理函数)
   c=>operation: UART_Receive_IT(UART_HandleTypeDef *huart) (接收函数)
   d=>end: HAL_UART_RxCpltCallback(huart);(中断回调函数)
   start->a->b->c->d
```

**代码:**

```
//在main函数中先调用一次接收中断函数,配置相关标志位
HAL_UART_Receive_IT(&huart1, (uint8_t *)&aRxBuffer, 1);
```

#### DMA 串口接收

- 当 DMA 串口开始接收后，DMA 通道会不断的将发送的数据转移到内存。

```flow
    start=>start: 1开始串口DMA接收
    a=>operation: 2串口收到数据,DMA将数据传输给内存
    b=>operation: 3一帧数据发送完毕,串口暂时空闲,出发串口空闲中断
    c=>operation: 4在中断服务函数中,可以计算刚才收到了多少个字节的数据
    end=>end: 5储存接收到的数据,清除标志位,开始下一帧数据接收
    start->a->b->c->end
```
---
  ```mermaid
  graph TB
    st(start)-->op[aition]
    op-->co{yes/no}
    co--no-->sub(function)
    sub-->op
    co--yes-->out>out]
    out-->en(stop)


