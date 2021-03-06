# android USB打印

版本|时间|内容|负责人
--|--|--|--
1.0|2020.5.12|创建|周星宇

## 0. 方案调研

#### 方案1: ghostscript+foo2zjs

#### 方案2: cups+hplip

#### 方案3: 使用PJL




## 1. 库文件编译

首先需要在编译linux内核的时候添加对USB打印机的支持
```
CONFIG_USB_PRINTER=y
```

或者通过menuconfig添加
`Device Drivers ---> [*] USB support ---> <*> USB Printer support`

### 1.1 安装交叉编译器

下载
注意，一定要用arm-linux-gcc 5.0以前的版本，否则无法编译cups

``` shell
export PATH=/usr/local/arm/5.4.0(文件夹版本号)/bin:$PATH
```

然后运行profile文件

``` shell
source  /etc/profile
```

 参考: https://blog.csdn.net/weixin_42108484/article/details/84295214

### 1.2 交叉编译CUPS

首先在官网上下载源码 https://github.com/apple/cups/releases

配置

``` shell
./configure --build=i686-linux \
CC=arm-linux-gcc \
CXX=arm-linux-g++ \
CCLD=arm-linux-gcc \
CCAUX=gcc \
RANLIB=arm-linux-ranlib \
AR=arm-linux-ar \
--host=arm-linux \
--target=arm-linux \
--build=i686-linux \
--prefix=/home/forlinx/printTools/cups \
--datadir=/home/forlinx/printTools/cups/usr/share \
--includedir=/home/forlinx/printTools/cups/usr/include \
--libdir=/home/forlinx/printTools/cups/lib \
--sysconfdir=/home/forlinx/printTools/cups/etc \
LDFLAGS="-lrt" \
--disable-shared \
--disable-gnutls \
--disable-network-build \
--disable-gssapi \
--disable-dbus
```

编译

``` shell
make && make install DSTROOT=<安装的绝对路径>
```
编译通过后会在安装路径找到生成的库文件，将库文件拷贝到开发板的文件系统中即可使用


生成过后还需要修改路径文件`cups/bin/cups-config`,把里面对路径的配置修改成安卓文件系统的对应路径

``` shell
prefix=/system
exec_prefix=/system
bindir=/system/bin
includedir=${prefix}/include
libdir=${exec_prefix}/lib
datarootdir=/system/share
datadir=/system/share
sysconfdir=/system/etc
cups_datadir=/system/share/cups
cups_serverbin=/system/lib/cups
cups_serverroot=/system/etc/cups
INSTALLSTATIC=
```

参考:
+ https://blog.csdn.net/qq_27681837/article/details/79866906
+ https://blog.csdn.net/lfn546489908/article/details/80762951

#### 问题解决
如果编译报错
```
undefined reference to `clock_gettime'
```
说明系统找不到`clock_gettime`库，需要在链接时加入依赖

修改Makedefs文件，找到LDFLAGS变量加入`-lrt`参数

``` Makefile
LDFLAGS		=	-lrt
```

### 1.3 交叉编译GhostScript


### 1.4 交叉编译libusb

下载libusb源码:https://github.com/libusb/libusb/releases

执行下列命令进行编译

``` shell
./configure --build=i686-linux \
CC=arm-linux-gcc \
CCLD=arm-linux-gcc \
CCAUX=gcc \
--host=arm-linux \
--target=arm-linux \
--prefix=/home/forlinx/output/libusb \
--disable-udev
```

``` shell
make && make install
```

其中:
+ –build=i686-linux表示该软件在x86平台被编译
+ –host=arm-linux表示该软件编译完成后在arm平台上运行
+ –prefix后面为软件安装目录。

ps: libusb-1.0.23以上的版本编译还会报网络的错误，使用1.0.22版本就可以编译通过

交叉编译方法参考：
https://blog.csdn.net/xfc_1939/article/details/53422071

### 1.5 交叉编译python

因为编译arm平台的python需要用到Pc平台的python。所以需要首先编译PC平台的python。将下载的压缩包解压两份，分别将文件夹命名为`Python-2.7.13-pc`和`Python-2.7.13-arm`

``` shell
./configure
make python Parser/pgen
```
pc端的编译好后，给为arm平台的python源码打上补丁、

``` shell
patch -p1 < ../Python-2.7.13-xcompile.patch
```

然后进行编译

``` shell
./configure --build=i686-linux \
CC=arm-linux-gcc \
CCLD=arm-linux-gcc \
CCAUX=gcc \
--host=arm-linux \
--target=arm-linux \
--prefix=/home/forlinx/output/python \
--enable-ipv6 \
--enable-shared \
ac_cv_file__dev_ptmx="yes" \
ac_cv_file__dev_ptc="no"
```

``` shell
make HOSTPYTHON=../Python-2.7.13-pc/python HOSTPGEN=../Python-2.7.13-pc/pgen \
BLDSHARED="arm-linux-gcc -shared" \
CROSS_COMPLIE=arm-linux- \
CROSS_COMPLIE_TARGET=yes \
HOSTARCH=arm-linux \
BUILDARCH=i686-linux \
-j2
```

``` shell
make install HOSTPYTHON=../Python-2.7.13-pc/python \
BLDSHARED="arm-linux-gcc -shared" \
CROSS_COMPILE=arm-linux- \
CROSS_COMPILE_TARGET=yes \
prefix=/home/forlinx/output/python
```

#### 异常处理
如果在install的过程中出现如下错误

```
ImportError: No module named _struct
```

在Makefile文件中删除PYTHONPATH变量的赋值即可

参考:
+ https://blog.csdn.net/tangweiguo0305/article/details/81223472

### 1.6 交叉编译hplip

下载hplip: https://sourceforge.net/projects/hplip/files/hplip/

下载需要注意，版本既不能太高也不能太低。
+ 3.18以后的版本在hpcups模块中加入了libImageProcess.so依赖，而该库没有arm版本。
+ 3.15以前版本cupsext里面会编译失败

笔者用的是3.16.7版本

#### 配置


``` shell
./configure --prefix=/home/forlinx/output/hplip \
CC=arm-linux-gcc \
CCLD=arm-linux-gcc \
CCAUX=gcc \
--host=arm-linux \
--target=arm-linux \
--build=i686-linux \
LDFLAGS="-L/home/forlinx/output/cups/usr/lib \
-L/home/forlinx/output/libusb/lib \
-L/home/forlinx/output/python/lib \
-lrt" \
CFLAGS="-I/home/forlinx/output/cups/usr/include \
-I/home/forlinx/output/libusb/include \
-I/home/forlinx/output/python/include" \
CPPFLAGS="-I/home/forlinx/output/cups/usr/include \
-I/home/forlinx/output/libusb/include \
-I/home/forlinx/output/python/include" \
--with-cupsbackenddir=/home/forlinx/output/cups/usr/lib/cups/backend \
--with-cupsfilterdir=/home/forlinx/output/cups/usr/lib/cups/filter \
--with-drvdir=/home/forlinx/output/cups/usr/share/cups/drv \
--with-mimedir=/home/forlinx/output/cups/usr/share/cups/mime \
--enable-cups-drv-install \
--enable-hpcups-install \
--enable-cups-ppd-install \
--disable-network-build \
--disable-scan-build \
--disable-qt4 \
--disable-fax-build \
--disable-dbus-build \
--disable-gui-build
```

``` shell
PYTHON="/usr/local/bin/python" \
PYTHONINCLUDEDIR="/home/forlinx/output/python/include/python2.7" \

```

如果configue出错的话通过查看config.log来定位问题的原因。

如果用普通用户权限configure报错，可以试试用su权限执行。

#### 编译

执行make命令前，首先要检查Makefile文件里的变量地址定义是否有误。一般情况下是库文件路径出错，例如在下面几行的定义中，头文件包含的路径和我们指定的路径不相符。

``` Makefile
#python头文件路径，换成正确的路径
PYTHONINCLUDEDIR = /usr/local/include/python2.7

#libusb头文件路径，换成正确的路径
am__append_8 = -I/usr/include/libusb-1.0
#......
am__append_20 = -I/usr/include/libusb-1.0
```

改完后执行`make`命令。成功后执行`make install`

#### 异常处理

如果指定了libusb库文件路径，编译时仍然报找不到`libusb-1.0.so.0`文件的错误。则将LDFLAGS变量里用`-L`指定的路径后面再用`-Wl,-rpath=`指定一次，如下所示:
``` Makefile
LDFLAGS = -L/home/forlinx/output/libusb/lib -Wl,-rpath=/home/forlinx/output/libusb/lib
```

如果报prnt/hpcups里面的某个变量未定义，原因是hpcups源码里没有包含cups的头文件。在`CommonDefinitions.h`里加入下面这几个头文件的包含

``` cpp
#include <cups/adminutil.h>
#include <cups/array.h>
#include <cups/backend.h>
#include <cups/dir.h>
#include <cups/file.h>
#include <cups/http.h>
#include <cups/ipp.h>
#include <cups/ppd.h>
#include <cups/pwg.h>
#include <cups/raster.h>
```

其他的错误见参考文献

参考:
+ https://github.com/jianglei12138/hplip
+ 论文:《基于ARM平台的Linux打印系统设计与实现》


## 2.安装打印环境

使用adb将`printTools`文件夹拷贝到开发板的`/sdcard`路径

然后执行如下脚本

``` shell
#重新挂载文件目录，获得读写权限
mount -o remount,rw /system
mount -o remount,rw /

cd /sdcard
busybox tar zxvf printTools.tar.gz

cd printTools/cups
cp -r etc/* /system/etc
cp usr/bin/* /system/bin/
cp -r usr/lib/* /system/lib/
cp -r usr/lib/* /lib/
cp -r usr/sbin/* /sbin/
cp -r usr/share/* /system/usr/share/

cd ../python
cp -r bin/* /system/bin/
cp -r lib/* /system/lib/
cp -r lib/* /lib/
cp -r share/* /system/usr/share/
busybox ln -s /system/bin/python2.7 /system/bin/python2
busybox ln -s /system/bin/python2 /system/bin/python

cd ../hplip
cp -r bin/* /system/bin/
cp -r lib/* /system/lib/
cp -r lib/* /lib/
cp -r share/hplip /system/usr/share/
cp -r share/ppd/* /system/etc/cups/ppd/

busybox chmod a+x /system/bin/*
```

## 3.软件设计
