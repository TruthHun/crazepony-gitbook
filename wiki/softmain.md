
#  软件框架讲解


飞控源代码部分，都是属于一砖一瓦敲出来的。没有使用实时操作系统（RTOS），我们称之为裸机代码，托管在[Github](https://github.com/Crazepony/crazepony-firmware-none)上，名字为crazepony-firmware-none，尾缀none表示未使用操作系统裸跑的意思。 

那么，现在就结合裸机代码，来说说Crazepony的软件框架。

> 本文档以Crazepony 5.2版本为基础。Crazepony 5.0版本及以前的代码主要由马骏（CamelGo）完成。贡献者黄永祥在5.1版本中对飞控代码进行了重构，将Crazepony的稳定性推向了一个新的高度。贡献者Nieyong在5.2版本中对代码进行了整理。

## 软件流程图
Crazepony软件流程如下图所示：

![](/assets/img/main.png)

Craepony主控使用的是STM32芯片，没有上实时操作系统，依靠中断嵌套来完成整体功能。程序核心就是通过定时器，在主循环中通过不断查询判断各个条件，这样就产生了几个大小不一样的时间段，我们根据需要就可以完成以多大的频率扫描一次遥控器指令、多久更新一次传感器数据、多久更新一次控制等等飞控需要实现的功能，尽可能的利用主控的资源。

## 初始化
进入主函数之后就是STM32处理器及各个部分的初始化。

![](/assets/img/main-init.jpg)


接下来就是进入主循环`while(1)`之中了，主循环也就是整个程序功能实现的关键，程序进入这里面就循环在里面运行了，当然中断会打断去运行中断服务程序运行完之后再回到这里运行。

## 主循环-100Hz循环
主循环体中首先有`if(loop100HzCnt >= 10){}`这个结构，其中loop100HzCnt这个变量是在TIM4中断服务程序中累加的，1ms累加一次，也就是说定时每10ms就去完成一次其中的工作。

那么100Hz需要做一次的工作是什么呢？读取mpu6050数据，气压计数据并进行整合。因为采用软解姿态，读取的数据为加速度计和陀螺仪的AD值，将数据进行标定、滤波、校正后通过四元素融合得到三轴欧拉角度。如下图。

![](/assets/img/caiji.png)

加速度传感器采集数据容易失真，造成姿态解算出来的欧拉角错误，只用角度单环情况下，使系统很难稳定运行，因此可以加入角速度作为内环，角速度由陀螺仪采集数据输出，采集值一般不存在受外界影响情况，抗干扰能力强，并且角速度变化灵敏，当受外界干扰时回复迅速增强了系统的鲁棒性。 

Crazepony采用双闭环PID控制，如下图所示。

![](/assets/img/jdpid.png)

角度作为外环，角速度作为内环，进行姿态双环PID控制。角度环的输出值作为角速度环的输入建立自稳系统。


## 主循环-50Hz循环
在`if(loop50HzFlag){}`进入50Hz（20ms执行一次）循环。`loop50HzFlag`标志位是在TIM4中断中每20ms置位一次的，这里解析了收到的遥控器无线发送过来的指令，结合当前的姿态计算更新这些控数据给核心控制算法输出控制飞控，我们就可以控制飞控前进后退，上升下降等等操作了。如下图。

![](/assets/img/loop50Hz.jpg)

## 主循环-10Hz循环
同样的思路`if(loop10HzFlag){}`也就是以10Hz的频率去执行下面功能。在这里可以通过蓝牙向我们的手机APP传送一些飞控的姿态信息，然后查询飞控的电量没有足够的话就让飞控降落下来，查询高度啊超出可控范围也把飞控降下来，查询是否和遥控器失联啊，失联就降下飞控等等安全飞行的控制。如下图

![](/assets/img/loop10Hz.jpg)

最后就是if(pcCmdFlag)这个了，这是一个与上位机调试有关的东西，主循环查询这个标志位，标志位是由上位机发送过来的指令置位的，它主要是处理pc机发送过来的指令，PID参数读取，修改等等。