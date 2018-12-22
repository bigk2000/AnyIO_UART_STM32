# AnyIO_UART_STM32
emulate UART using any IO , full duplex, max baudrate >= 115200

问1：是否可以使用IO模拟UART通讯？
问2：模拟UART波特率最大波特率能够做到多少？是否可以达到115200或以上？
问3：模拟UART的时序误差最小可以做到多少？
问4：模拟UART波特率较高时，如何降低CPU负载？


使用IO模拟UART并不复杂，但如何做到高性能的模拟UART则是一个挑战；

是否有可能使纯软件模拟的UART满足以下特性：
1，全双工，半双工均可；
2，最大波特率可达115200及以上，甚至可以做到512000；
3，任意IO模拟TX RX；
4，发送时序可做到0误差；
5，CPU负载率小；

下面来探讨如何在STM32上实现满足以上特性的模拟UART。

基本思路：
  使用定时器触发DMA的方式模拟UART的收发；

具体实现，以通讯格式9600,8N1为例：
  将TIM2（或TIM3）的周期设为波特率周期104.2us，设一个通道的比较匹配值为52us；
  发送时，仅开启计数溢出触发DMA，设置DMA搬运次数为10（1个起始位， 8个数据位，1个停止位），格式化要发送的数据保存到发送数组中（注解），
  设置DMA把发送数组的数据搬运到GPIO；
  然后启动定时器，当DMA搬运完成时，这个字节就发送完毕了；
  接收时，仅开启计数匹配触发DMA，设置DMA搬运次数为10，设置DMA把GPIO搬运到接收数组，设置RX脚下降沿触发IO中断；
  当IO中断触发时，清零定时器并启动；  当DMA搬运完成中断触发时，停止定时器，此时字节已经接收完毕；
  
  注解：
    以TX脚位为PIN4，发送0x23为例，定义一个10元素的数组，元素长度为16bit（对应GPIO位宽）；
    格式化后的10个元素依次为：
    起始位 ：XXXX XXXX XXX0 XXXX
    数据位0：XXXX XXXX XXX0 XXXX
    数据位1：XXXX XXXX XXX1 XXXX
    数据位2：XXXX XXXX XXX0 XXXX
    数据位3：XXXX XXXX XXX0 XXXX
    数据位4：XXXX XXXX XXX1 XXXX
    数据位5：XXXX XXXX XXX1 XXXX
    数据位6：XXXX XXXX XXX0 XXXX
    数据位7：XXXX XXXX XXX0 XXXX
    停止位 ：XXXX XXXX XXX1 XXXX
    
  另注：用DMA直接搬运数据到IO会干扰GPIO上其他处于输出状态的IO，
       解决这个问题有2个办法：
       1，使用其他IO都是输入模式的GPIO作为TX脚（如STM32的GPIOC和GPIOD，pin脚少，容易实现仅一个输出模式的IO）；
       2，把DMA的目标地址设为GPIO_BSRR即可，但要注意数组格式化的方法不同，元素长度变为32bit，要置位pin脚时使用低16bit，清零pin脚时用高16bit；
       
程序代码：
  我已经在工程中验证了这个方法的可行性，经过测试，收发在115200时无误码（高波特率接收时，IO下降沿中断和接收完毕中断的优先级要高）；
  这里暂不提供DEMO了。。。。（坑啊。。。）
  
缺点：
  需要使用1个定时器，2个DMA通道；
  并非严格意义的全双工，使用1个定时器时，收发不能同时进行；（要实现同时收发，可以使用2个定时器分别触发DMA）

总结：
  发送时，CPU只要启动定时器就可以了，发送完成时可以不使用DMA中断，使用查询就可以；
  接收时，起始位的下降沿触发IO中断时，需要进入中断开启定时器，字节接收完成时也需要进入DMA中断读取接收到的字节；
  因此CPU只要在收发开始和结束时干预一下就好了；
  通过把DMA的目标地址设为GPIO_BSRR可以实现任意IO作为TX,RX脚；
  经过测量，应该是可以实现512000波特率收发的，但未验证；
  本方法可以用在有空闲定时器和DMA的任何MCU上；

