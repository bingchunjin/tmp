---
layout: post
title: SIFLOWER校准方案说明手册
categories: PRODUCE
description: 介绍SIFLOWER校准方案
keywords: calibration
mermaid: true
---

# SIFLOWER校准方案说明手册

**目录**

* TOC
{:toc}

## 1 介绍

本文针对SIFLOWER芯片方案的WiFi校准相关指令进行说明，适用于生产校准开发相关人员  
目前对板子进行射频校准一共有两种方案：  

- uboot下PCBA模式校准
  PCBA校准适用于烧录具有pcba镜像的siflower方案芯片的产品，在uboot下且进入PCBA测试模式，结合矽昌PCBA工具使用  
  PCBA工具使用以及配置参考[PCBA工具使用手册](https://siflower.github.io/2020/09/10/pcba_tool_interface_guide/)  

- 系统下ate模式校准
  系统下校准指令ate_cmd主要是通过ate_tool这个package实现的，编译镜像时选上这个package即可，所有指令均是在系统下直接操作  
  ate_cmd说明参考[SIFLOWER射频测试手册](https://siflower.github.io/2020/12/01/siflower_ate_test_guide/)
  
## 2 PCBA校准

### 2.1 PCBA校准流程

pcba校准在uboot下进行，是一段运行在uboot下的特殊程序，专门用于配合PC端工具进行产测，步骤如下

- 板子烧录带PCBA的镜像，上电进入PCBA模式
- PC端运行PCBA校准工具
- PCBA校准内容
  - 天线顺序
     依次校准2.4G_ANT1、2.4G_ANT2、5G_ANT1、5G_ANT2
  - 校准顺序
      1，晶振校准
      校准开始读取board.ini初始值XO值进行校准，目标范围-2.5ppm ~ +2.5ppm，校准完后将频偏值写入flash供后续射频校准使用
      2，各个信道模式校准，默认如下
      ```c
      2.4G  
      CH1/CH6/CH11信道的11b_11M，11n_20M_MCS7,11N_40M_MCS7  
      5G  
      CH36/CH64/CH149信道的11a_54M、11ac_20M_MCS8，11ac_40M_MCS9，11ac_80M_MCS9

      校准的信道、模式可以通过配置文件配置  
      ```  

      3，校准逻辑  
      1) 首先按照wifi_limit.txt中配置的目标功率，通过调节gain值，校准每个频点，满足evm标准和目标功率门限，即将此gain值作为校准值保存  
      2) 然后依次校准完所有模式的速率最高点，再通过test_power_index.txt设置的偏移值完成同一模式下未校准的速率的gain值，这样就完成一个信道的校准值  
      3) 相同band信道校准值进行copy，完成整个校准分区值，依次是2.4G_1,2.4G_2,5G_1,5G_2  
      4) 最后将完整的校准值写入校准分区保存  
      
      4，校准结果查看  
      每个板子校准pass/fail的结果会分别保存到pass_log和fail_log目录，可以对应查看   
      另外板子校准pass会解锁PCBA模式，可以在系统下通过ate_cmd查看  

	**注：目前PCBA工具不支持校准完之后综测，但是如果需要校准值验证功能，此功能是可以在这个基础上进行开发和添加的**  

## 3 ate_cmd校准  

ate_cmd作为siflower方案射频测试的控制命令，可以基于此命令对射频性能进行测试，也可以基于此命令的强大功能开发校准工具  
源码在```openwrt-18.06/package/siflower/bin/atetools/src/main.c```  

- 进入ate模式  
在使用ate_cmd指令之前，需要先使用ate_init指令进行初始化，让板子进入ate模式，也可以在初始化时增加参数对天线进行控制  

```
ate_init //默认打开四路天线
ate_init lb1  //打开lb1
ate_init lb2  //打开lb2
ate_init hb1  //打开hb1
ate_init hb2  //打开hb2
```

- 退出ate模式  

```
当各项测试完成后，可以使用sfwifi reset fmac退出，恢复正常模式
```

### 3.1 频偏校准  

1，在系统下使用ate_cmd命令控制板子做2.4G 或者5G 的TX常发;  
2，通过仪器获取TX结果，并将获取到的ppm与设定范围（-2.5 +2.5）做对比，在门限范围则停止XO校准，将此XO值写入板子  
如超出设定范围，则通过ate_cmd调整xo值，如此重复，直至ppm满足设定范围（-2.5 +2.5）。  

具体操作，举例如下：  

1，在ate_init进入ate模式后，选择任意信道模式发送调制信号如：2412_11n_20M_mcs7  

```
ate_cmd wlan0 fastconfig -l 1024 -f 2412 -c 2412 -w 1 -u 1 -m 2 -i 7 -g 0 -B 1 -p 15 -y
```

2，使用综测仪测量频偏值  

3，输入下面的命令调整频偏，通过仪器可观察变化。  

```
ate_cmd wlan0 fastconfig -O value  //value的范围是0x00-0xff(0~255)
```

注意：使用 -O 指令时value参数值，必须是用10进制，需要转换一下。其中value的值需要不断调整修改，不同的值会影响频偏的变化，然后观察频偏结果，当频偏满足±2.5ppm后，进行步骤4。  

4，保存调整好的频偏值  

输入下面的命令储存校准值,比如校准好频偏后value结果为31，则需要将31转化为0x1f写入  

```
ate_cmd save 0x1f1f    
```

5，输入下面的命令停止发送，完成XO校准  

```
ate_cmd wlan0 fastconfig -q
```


### 3.2 TX校准  

#### 3.2.1 2.4G TX校准  

- 2.4G TX校准指令  

指令通过ate_cmd加信道/模式/带宽/速率/等参数组合而成，可以根据需要测试的信道模式来进行发送，然后通过综测仪解析结果。  

如发送ANT1_2412信道11n,40M带宽，mcs7信号的指令为
```
ate_cmd wlan0 fastconfig -l 1024 -f 2412 -c 2412 -w 2 -u 2 -m 2 -i 7 -g 0 -B 1 -p 15 -y
```
这个指令可以控制板子2.4G常发，如果仪器测量的结果不满足需求，则调整-p后面的gain值参数来调整发射功率  

- 2.4G 停止发送指令  

ate_cmd wlan0 fastconfig -q

- 5G TX校准指令  
  
指令通过ate_cmd加信道/模式/带宽/速率/等参数组合而成，可以根据需要测试的信道模式来进行发送，然后通过综测仪解析结果。  

如发送ANT1_5180信道11ac,80M带宽，MSC9信号的指令为

```
ate_cmd wlan1 fastconfig  -l 1024 -f 5180 -c 5210 -w 3 -u 3 -m 4 -i 9 -g 0 -B 1 -p 5 -y
``` 

这个指令可以控制板子5G常发，如果仪器测量的结果不满足需求，则调整-p后面的power_idx参数来调整发射功率  

- 5G停止发送指令  

```
ate_cmd wlan1 fastconfig -q
```
各个参数详细说明参考[SIFLOWER射频测试手册](https://siflower.github.io/2020/12/01/siflower_ate_test_guide/)  


### 3.3 校准步骤  

- 1，板子上电，系统启动完成后使用ate_init初始化（单路初始化，单路依次校准)；完成晶振校准写入XO值，并且写入WiFi version  

- 2，板子通过ate_cmd指令进行TX发送；  

- 3，仪器读取测试结果与evm标准和目标功率门限做比较(首先判断EVM，在evm满足的前提下，调整gain值满足目标功率门限)，如满足设定则板端保存当前测试项目的gain值作为校准值；如不满足，则调整-p 参数后再次重复步骤2，直至获取的power和EVM的测试结果与设定相符，满足后保存此gain值；(具体判断由PC端工具控制仪器实现，目标功率以及目标频偏也是工具这边设置）  

- 4，将测试pass的gain值进行不同的offset补偿写入未校准的其它速率；  

- 5，copy已经偏移完成的信道校准值到同band未测试的信道；  

- 6，完成所有信道校准值，通过ate_cmd指令写入FLASH对应分区。  

### 3.3.1 ate_cmd手动校准示例  

- 需要使用ate_cmd校准如下频点   
  
```
2.4G（ANT1、ANT2） CH1/CH6/CH11信道的11b_11M，11n_20M_MCS7,11N_40M_MCS7  
5G  （ANT1、ANT2） CH36/CH64/CH149信道的11ac_20M_MCS8，11ac_40M_MCS9，11ac_80M_MCS9 
```

- 手动校准步骤（此步骤可用于对接各家产测工具，按照如下逻辑下发指令）  
  
1，ate_init 初始化对应天线    
2, 校准XO 参考频偏校准测试    
3, 校准完XO后写入wifi version（两张校准表生效，一定要写V4。指令为```ate_cmd save ver V4```   )  
4, 初始化power_save文件。然后利用ate_cmd常发需要测试的各个频点，通过仪器读取结果，调整gain值满足evm标准和目标功率门限，会将最后一次满足目标的-p 值写入对应power_save_ant1/2.txt ,校准完所有点如下：

![power_save.png](/assets/images/siflower_calibration/power_save.png)

**注：上述所有频点必须全部校准，否则偏移会出错**   

5，完成一路二路所有校准点后，开始在校准频点的基础上进行offset，得到完整的两个校准表，并查看最终校准值保存文件。    
   使用命令```ate_cmd tx_calibrate_over```会在校准点的基础上与 /etc/atetools/gain_offset.txt中的值进行偏移，获得最后的校准表，读取如下  

![power_offset.png](/assets/images/siflower_calibration/power_offset.png)

**注：gain_offset.txt中的offset值可以按照客户板子实际测试情况进行调整偏移量，一般来说由射频测试完整之后提供，然后按照txt格式生成一个客户自己的偏移值文档放入软件对应位置即可** 

6, 综测，即存入txt文件的校准值检验  
如读取5G ant2 powe_save_ant2.txt保存值常发校验  

```ate_cmd wlan1 fastconfig -l 1024 -f 5180 -c 5180 -w 1 -u 1 -m 4 -i 8 -g 0 -B 2 -p 203 -y```

只需要将-p 后面的参数固定**203**，使用ate_cmd常发时，就会去对应的power_save_ant.txt文件中拿取对应位置的gain值进行常发。   
可用于验证即将写入值的准确性，实现综测功能  

7，写入factory校准分区  

使用```ate_cmd wlan1 fastconfig -p save_all```会将上一步填充好的的save_power.txt的内容写入校准分区分区保存   

8，完成校准，重启校准表生效  

重新上电后进入ate模式，使用指令```ate_cmd wlan1 fastconfig -p read_all```可以读取校准值   

![power_read.png](/assets/images/siflower_calibration/power_read.png)

**注：这部分描述是手动输入ate_cmd指令进行操作，目前提供的PC端atetool工具，便是基于此命令做成了自动化工具，如需在系统下开发产测校准可以参考实现**   

- 如何使用校准分区值发送  
如读取2.4G ant1 factory分区gain值常发，命令如下  

```ate_cmd wlan1 fastconfig -l 1024 -f 2412 -c 2422 -w 2 -u 2 -m 2 -i 7 -g 0 -B 1 -p 200 -y```
只需要将-p 后面的参数固定**200**，使用ate_cmd常发时，就会去校准分区中拿取对应位置的gain值进行常发，可用于验证写入校准值的准确性  

### 3.4 ate_cmd 其它功能  

- 读单个频点的校准值（直接读flash）  

ate模式下使用 如```ate_cmd read pow 2412 0 0 0```读取2412/11b/20Mhz/1Mbps的校准值    
解释：ate_cmd save pow为固定参数 后面4个参数为 channel bw mode rate    

- 写单个频点的校准值（直接写flash）  

ate模式下使用```ate_cmd save pow 15 2412 0 0 0```则将2412/11b/20Mhz/1Mbps的校准值写入15    
解释：ate_cmd save pow为固定参数 后面五个参数为 gain channel bw mode rate  

- 初始化power_save_ant1.txt/power_save_ant2.txt

可以使用```ate_cmd init_power_save```即可初始化power_save_ant1.txt为初始全部为零的状态，校准之前需先做这一步操作。  

- 读取/保存XO值  
  
使用```ate_cmd save value```(value为XO值，十六进制)即可保存XO校准值到factory分区)      
使用```ate_cmd read XO_value```即可读取factory分区保存的XO校准值  

- 读取/保存校准温度  

使用```ate_cmd save temp value``` (value为温度数值，十进制)即可保存校准时芯片的温度，用于后续RF温补    
使用```ate_cmd read temp``` 即可读取factory分区保存的校准时芯片的温度   

- 读取gmac trx delay校准值  

使用```ate_cmd read trx_delay```即可读取gmac自动校准保存到factory分区的trx_delay值  

- 切换天线  

除了通过ate_init来初始化天线以为，为了节省时间同样实现了利用ate_cmd切天线，但是使用时必须是在ate_init的模式   
```
ate_cmd wlan0 fastconfig -T 0x33 打开lb1
ate_cmd wlan0 fastconfig -T 0xcc 打开lb2
ate_cmd wlan1 fastconfig -T 0x33 打开hb1
ate_cmd wlan1 fastconfig -T 0xcc 打开hb2
```

## 4 校准分区内容介绍  

不管是PCBA校准或者是ate_cmd校准，最终的目的都是将校准值写入factory分区对应位置，供WiFi驱动加载使用，校准过程写入的值介绍如下：  

以16M nor flash为例，Factory分区起始位置在0x90000  

- 前面2kB为系统信息使用，后面为WiFi校准值保存  

|说明|addr|length|
|--|--|--|
|系统信息分区起始地址|0x90000|2048|
|wifi version值起始地址|0x90800|2|
|xo校准值起始地址|0x90802|2|
|2.4G ant1 校准值起始地址|0x90804|364|
|5G ant1 校准值起始地址|0x90970|1325|
|2.4G ant2 校准值起始地址|0x90e9d|364|
|5G ant2 校准值起始地址|0x91009|1325|

- 校准值  

目前默认使用双路校准表，gain值范围是0~15；
旧版本使用单路校准表功率范围是0~31；(已弃用)  

校准值实际写入值，以2.4G_ANT1和5G_ANT1为例：
![ant1.png](/assets/images/siflower_calibration/ant1.png)  

校准值内容解释：  
校准值按照2.4G/5G按照信道模式速率依次连续写入，这里方便说明做了分行处理，  
  - 2.4G
    每行的28个数据分别代表一个2.4G信道，每个值按照11b/11g/11n_20M/11n_40M的速率从低到高依次排列
    每一行代表2.4G各个信道  

  - 5G 
    每行53个数据分别代表一个5G信道，每个值按照11a/11n_20M/11n_40M/11ac_20M/11ac_40M/11ac_80M的速率从低到高依次排列
    每一行代表5G各个信道    

## 5 关于使用功率步进  

目前默认使用1db 步进规则，即gain值增加1，功率对应增加1。如果需要追求精度,目前最低支持0.5db step,设置方法如下：  
由于gain值输入无法使用小数点，如14.5这种，所以需要通过16进制或运算来转换  

第一次发送假如使用15这个gain 下一次则使用(0x80|15)的十进制结果143，这样获得的功率则会在31的基础上减少0.5dB  
下一次再使用14，则会在133这个gain的基础上再减少0.5dB ,再下一次就是132即(0x80 | 14)，举个例子  

|gain||说明|实际指令|power(db)|
|--|--|--|--|--|
|15|||ate_cmd wlan1 fastconfig -l 1024 -f 5180 -c 5210 -w 3 -u 3 -m 4 -i 9 -g 0 -p 15 -y| 20 |
|143|减少0.5db||ate_cmd wlan1 fastconfig -l 1024 -f 5180 -c 5210 -w 3 -u 3 -m 4 -i 9 -g 0 -p 143 -y| 19.5 |
|14|减少0.5db||ate_cmd wlan1 fastconfig -l 1024 -f 5180 -c 5210 -w 3 -u 3 -m 4 -i 9 -g 0 -p 14 -y| 19.0 |
|142|减少0.5db||ate_cmd wlan1 fastconfig -l 1024 -f 5180 -c 5210 -w 3 -u 3 -m 4 -i 9 -g 0 -p 142 -y| 18.5 |
|13|减少0.5db||ate_cmd wlan1 fastconfig -l 1024 -f 5180 -c 5210 -w 3 -u 3 -m 4 -i 9 -g 0 -p 13 -y| 18 |
|141|减少0.5db||ate_cmd wlan1 fastconfig -l 1024 -f 5180 -c 5210 -w 3 -u 3 -m 4 -i 9 -g 0 -p 141 -y| 17.5 |
|12|减少0.5db||ate_cmd wlan1 fastconfig -l 1024 -f 5180 -c 5210 -w 3 -u 3 -m 4 -i 9 -g 0 -p 12 -y| 17 |
|140|减少0.5db||ate_cmd wlan1 fastconfig -l 1024 -f 5180 -c 5210 -w 3 -u 3 -m 4 -i 9 -g 0 -p 140 -y| 16.5 |
|11|减少0.5db||ate_cmd wlan1 fastconfig -l 1024 -f 5180 -c 5210 -w 3 -u 3 -m 4 -i 9 -g 0 -p 11 -y| 16 |
|139|减少0.5db||ate_cmd wlan1 fastconfig -l 1024 -f 5180 -c 5210 -w 3 -u 3 -m 4 -i 9 -g 0 -p 139 -y| 15.5 |
|10|减少0.5db||ate_cmd wlan1 fastconfig -l 1024 -f 5180 -c 5210 -w 3 -u 3 -m 4 -i 9 -g 0 -p 10 -y| 15 |
|...|...||...| ... |
|0|减少0.5db||ate_cmd wlan1 fastconfig -l 1024 -f 5180 -c 5210 -w 3 -u 3 -m 4 -i 9 -g 0 -p 0 -y| 5 |

## 总结

目前矽昌共有两种校准方案可以使用，根据客户实际需求选择，实现对产品进行WiFi校准。  

## FAQ

**1，ate_init脚本的作用是什么？需要如何对接？**  
在ate测试之前为了避免其它影响，使用ate_init脚本进行初始化，会重新进行天线配置，把wifi驱动卸载再加载，并且关闭上层WiFi接口 然后进入ATE模式，等待ate_cmd指令  
这初始化主要是因为ate是一种常发模式，不能兼容其他，目的就是把其它上层wifi调用清干净，这样我们就可以进行macbypass常发了。  
注：ate_init加载驱动和正常流程加载驱动是一样的，唯一不同的地方是，ate_init会在卸载驱动的同时把hostapd相关配置删掉，这样上层就不会起ap接口，没有任何调用了。  

**2，完成晶振校准后写入WiFi version，写入的是什么？是为了写入标志生效校准表吗？**  
写入如V4对应双路校准表生效、以前旧的方案2.4G/5G两路使用一张校准表，使用version V3，为了兼容之前的旧版本，所以通过version来区分  

**3，ate指令校准和PCBA工具校准是否存在功能上的差别？**  
两种方案在最终在射频收发上是一样的，只是一个跑在uboot下面，是我们自创的一种比较快速的测试模式。  
另外一个在系统下面执行，是一种比较传统的射频测试方式。两种方案选任意一种即可  

**4，校准表的加载过程，以及没有校准的板子是使用什么校准数据？**  
驱动使用的校准表有三个来源，优先级分别为
overlay表 > factory分区 > default表   

(a)overlay表：
在板子上位于`usr/bin/txpower_calibrate_table.sh`，为第一优先级。默认不存在，可以由`usr/bin/txpower_calibrate_table.sh`脚本将`usr/bin/txpower_calibrate_table.txt`生成该校准表.  

(b)factory表：校准后存在，第二优先级。  

(c)default表：默认存在在板子上位于`lib/firmware/default_txpower_calibrate_table.bin`(外置pa会使用`lib/firmware/default_txpower_calibrate_expa_table.bin`)，为第三优先级。如果前两张表都没有，默认使用这张校准表。  
使用时，按照优先级从高到低依次检查是否存在以及格式是否正确，并且日志中会指出最终使用的哪一张表  

**5，芯片功率控制精度以及芯片的功率跳动范围？**  
芯片功率精度±0.5db ，功率跳动范围在1db以内  

**6，功率温补相关**  
由于射频功率随着温度上升会有下降，所以需要通过温度差异对功率进行补偿，来达到功率稳定的目的  
温补分为信令和非信令模式  
1) 信令模式  
即正常软件状态下  
目前系统软件温补默认是关闭的，需要默认打开。编译时要选上配置文件如`target/linux/siflower/sf19a28_ac28_fullmask_def.config`中，如下配置
![config.png](/assets/images/siflower_calibration/config.png)  

2) 非信令模式  
ate模式或者校准时  
校准模式下为了功率的准确性，是不开启温补。可以通过如下指令关闭温补  
ate_init之后  
```
echo 1 > /sys/kernel/debug/ieee80211/phy2/siwifi/temp_disable  //关闭2.4G温补
echo 1 > /sys/kernel/debug/ieee80211/phy3/siwifi/temp_disable  //关闭5G温补

注：phy后面的数字会随着ate_init而增加，默认是phy0(2G)  phy1(5G)，ate_init一次之后则会增加为 phy2(2G)  phy3(5G) 依次类推
```

然后再开始使用ate_cmd 发送TX指令，使用ate_cmd指令发送TX的时候可以看如下  
**未开温补：**  

![without_temp.png](/assets/images/siflower_calibration/without_temp.png)  

**开启温补**  

![temp.png](/assets/images/siflower_calibration/temp.png)  

注意：  
不管信令或者非信令模式下，温补生效的前提是，校准的时候保存了当时的温度值到factory分区作为基准温度，温补的补偿值是根据实时温度和基准温度差值计算而来的，超过基准温度之后开始补偿，目前demo ac28是每升高5℃补偿0.5db（具体温补的数据根据不同板子可能有不同，可以实际测试修改），保证此时功率和校准时候功率一致，不随温度变化而降低。  
读取芯片温度指令  
```cat /sys/kernel/debug/aetnensis/temperature```  
写温补基准温度指令  
```ate_cmd save temp 20 (一般再校准时写入基准温度20℃，写完重启生效，永久保存在flash分区）```  
读取factory分区保存的温补基础温度  
```ate_cmd read temp```  

## 附录

ate_cmd手动校准指令集  

**11b:**  
①校准2.4g 1路 11b_11M 2412 ate_cmd wlan0 fastconfig -l 1024 -f 2412 -c 2412 -w 0 -u 0 -m 0 -i 3 -g 0 -B 1 -p 31 -y  
校准2.4g 2路 11b_11M 2412 ate_cmd wlan0 fastconfig -l 1024 -f 2412 -c 2412 -w 0 -u 0 -m 0 -i 3 -g 0 -B 2 -p 31 -y  
②校准2.4g 1路 11b_11M 2437 ate_cmd wlan0 fastconfig -l 1024 -f 2437 -c 2437 -w 0 -u 0 -m 0 -i 3 -g 0 -B 1 -p 30 -y  
校准2.4g 2路 11b_11M 2437 ate_cmd wlan0 fastconfig -l 1024 -f 2437 -c 2437 -w 0 -u 0 -m 0 -i 3 -g 0 -B 2 -p 30 -y  
③校准2.4g 1路 11b_11M 2462 ate_cmd wlan0 fastconfig -l 1024 -f 2462 -c 2462 -w 0 -u 0 -m 0 -i 3 -g 0 -B 1 -p 30 -y  
校准2.4g 2路 11b_11M 2462 ate_cmd wlan0 fastconfig -l 1024 -f 2462 -c 2462 -w 0 -u 0 -m 0 -i 3 -g 0 -B 2 -p 30 -y  
**11n_20m:**  
①校准2.4g 1路 11n_20M 2412 ate_cmd wlan0 fastconfig -l 1024 -f 2412 -c 2412 -w 1 -u 1 -m 2 -i 7 -g 0 -B 1 -p 26 -y  
校准2.4g 2路 11n_20M 2412 ate_cmd wlan0 fastconfig -l 1024 -f 2412 -c 2412 -w 1 -u 1 -m 2 -i 7 -g 0 -B 1 -p 26 -y  
②校准2.4g 1路 11n_20M 2437 ate_cmd wlan0 fastconfig -l 1024 -f 2437 -c 2437 -w 1 -u 1 -m 2 -i 7 -g 0 -B 1 -p 26 -y  
校准2.4g 2路 11n_20M 2437 ate_cmd wlan0 fastconfig -l 1024 -f 2437 -c 2437 -w 1 -u 1 -m 2 -i 7 -g 0 -B 2 -p 26 -y  
③校准2.4g 1路 11n_20M 2462 ate_cmd wlan0 fastconfig -l 1024 -f 2462 -c 2462 -w 1 -u 1 -m 2 -i 7 -g 0 -B 1 -p 26 -y  
校准2.4g 2路 11n_20M 2462 ate_cmd wlan0 fastconfig -l 1024 -f 2462 -c 2462 -w 1 -u 1 -m 2 -i 7 -g 0 -B 2 -p 26 -y  
**11n_40m:**  
①校准2.4g 1路 11n_40M 2422 ate_cmd wlan0 fastconfig -l 1024 -f 2412 -c 2422 -w 2 -u 2 -m 2 -i 7 -g 0 -B 1 -p 27 -y  
校准2.4g 2路 11n_40M 2422 ate_cmd wlan0 fastconfig -l 1024 -f 2412 -c 2422 -w 2 -u 2 -m 2 -i 7 -g 0 -B 2 -p 27 -y  
②校准2.4g 1路 11n_40M 2437 ate_cmd wlan0 fastconfig -l 1024 -f 2437 -c 2437 -w 2 -u 2 -m 2 -i 7 -g 0 -B 1 -p 27 -y  
校准2.4g 2路 11n_40M 2437 ate_cmd wlan0 fastconfig -l 1024 -f 2437 -c 2437 -w 2 -u 2 -m 2 -i 7 -g 0 -B 2 -p 27 -y  
③校准2.4g 1路 11n_40M 2462 ate_cmd wlan0 fastconfig -l 1024 -f 2462 -c 2462 -w 2 -u 2 -m 2 -i 7 -g 0 -B 1 -p 27 -y  
校准2.4g 2路 11n_40M 2462 ate_cmd wlan0 fastconfig -l 1024 -f 2462 -c 2462 -w 2 -u 2 -m 2 -i 7 -g 0 -B 2 -p 27 -y  

**11ac_20:**  
①校准5g 1路 11ac_20M_mcs8 5180 ate_cmd wlan1 fastconfig -l 1024 -f 5180 -c 5180 -w 1 -u 1 -m 4 -i 8 -g 0 -B 1 -p 6 -y  
校准5g 2路 11ac_20M_mcs8 5180 ate_cmd wlan1 fastconfig -l 1024 -f 5180 -c 5180 -w 1 -u 1 -m 4 -i 8 -g 0 -B 2 -p 6 -y  
①校准5g 1路 11ac_20M_mcs8 5320 ate_cmd wlan1 fastconfig -l 1024 -f 5320 -c 5320 -w 1 -u 1 -m 4 -i 8 -g 0 -B 1 -p 6 -y  
校准5g 2路 11ac_20M_mcs8 5320 ate_cmd wlan1 fastconfig -l 1024 -f 5320 -c 5320 -w 1 -u 1 -m 4 -i 8 -g 0 -B 2 -p 6 -y  
①校准5g 1路 11ac_20M_mcs8 5745 ate_cmd wlan1 fastconfig -l 1024 -f 5745 -c 5745 -w 1 -u 1 -m 4 -i 8 -g 0 -B 1 -p 6 -y  
校准5g 2路 11ac_20M_mcs8 5745 ate_cmd wlan1 fastconfig -l 1024 -f 5745 -c 5745 -w 1 -u 1 -m 4 -i 8 -g 0 -B 2 -p 6 -y  
**11ac_40:**  
①校准5g 1路 11ac_40M_mcs9 5190 ate_cmd wlan1 fastconfig -l 1024 -f 5180 -c 5190 -w 2 -u 2 -m 4 -i 9 -g 0 -B 1 -p 6 -y  
校准5g 2路 11ac_40M_mcs9 5190 ate_cmd wlan1 fastconfig -l 1024 -f 5180 -c 5190 -w 2 -u 2 -m 4 -i 9 -g 0 -B 2 -p 6 -y  
①校准5g 1路 11ac_40M_mcs9 5330 ate_cmd wlan1 fastconfig -l 1024 -f 5320 -c 5310 -w 2 -u 2 -m 4 -i 9 -g 0 -B 1 -p 6 -y  
校准5g 2路 11ac_40M_mcs9 5330 ate_cmd wlan1 fastconfig -l 1024 -f 5320 -c 5310 -w 2 -u 2 -m 4 -i 9 -g 0 -B 2 -p 6 -y  
①校准5g 1路 11ac_40M_mcs9 5755 ate_cmd wlan1 fastconfig -l 1024 -f 5745 -c 5755 -w 2 -u 2 -m 4 -i 9 -g 0 -B 1 -p 6 -y  
准5g 2路 11ac_40M_mcs9 5755 ate_cmd wlan1 fastconfig -l 1024 -f 5745 -c 5755 -w 2 -u 2 -m 4 -i 9 -g 0 -B 2 -p 6 -y  
**11ac_80:**  
①校准5g 1路 11ac_80M_mcs9 5210 ate_cmd wlan1 fastconfig -l 1024 -f 5180 -c 5210 -w 3 -u 3 -m 4 -i 9 -g 0 -B 1 -p 6 -y  
校准5g 2路 11ac_80M_mcs9 5210 ate_cmd wlan1 fastconfig -l 1024 -f 5180 -c 5210 -w 3 -u 3 -m 4 -i 9 -g 0 -B 2 -p 6 -y  
①校准5g 1路 11ac_80M_mcs9 5350 ate_cmd wlan1 fastconfig -l 1024 -f 5320 -c 5290 -w 3 -u 3 -m 4 -i 9 -g 0 -B 1 -p 6 -y  
校准5g 2路 11ac_80M_mcs9 5350 ate_cmd wlan1 fastconfig -l 1024 -f 5320 -c 5290 -w 3 -u 3 -m 4 -i 9 -g 0 -B 2 -p 6 -y  
①校准5g 1路 11ac_80M_mcs9 5775 ate_cmd wlan1 fastconfig -l 1024 -f 5745 -c 5775 -w 3 -u 3 -m 4 -i 9 -g 0 -B 1 -p 6 -y  
校准5g 2路 11ac_80M_mcs9 5775 ate_cmd wlan1 fastconfig -l 1024 -f 5745 -c 5775 -w 3 -u 3 -m 4 -i 9 -g 0 -B 2 -p 6 -y