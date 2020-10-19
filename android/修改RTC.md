## 修改RTC

说明：飞凌6818的源码中，默认使用外部的RTC8010芯片，但实验室的十通道底板中无该芯片，所以需要修改源码切换RTC方式，改为使用6818核心板自带的RTC功能。



需要修改如下地方

修改~/ForlinxS5p6818/source/okxx18-source-android51/kernel/arch/arm/configs/s5p6818_drone_android_lollipop_defconfig文件

```
#
# I2C RTC drivers
#

# CONFIG_RTC_DRV_RX8010=y  将8010注释掉
```

```
#
# on-CPU RTC drivers
#

CONFIG_RTC_DRV_NXP=y  NXP设置为y
```

重新编译源码，即可完成RTC方式的切换。
