# 记一次本地调用引发的JVM崩溃2
## 背景
* 应用运行在tomcat上，并使用JNA调用了liba.so中的本地方法

## 症状综述
1. 本地windows和linux环境下使用同样正常，运行部分数据调用本地方法会触发JVM崩溃

## 排查过程
1. 根据上一次的经验，首先排查catalina.out和hs_err_pid.log文件，发现出现以下报错
```
*** Error in `/usr/java/jdk1.8/bin/java': free(): invalid next size (normal): 0x00007fb6a01b21e0 ***
======= Backtrace: =========
/lib64/libc.so.6(+0x81299)[0x7fb7d9f9a299]
/usr/java/jdk1.8.0_202/jre/lib/amd64/server/libjvm.so(+0x63e375)[0x7fb7d9643375]
/home/deploy/.cache/JNA/temp/jna1997716433451596580.tmp(+0xc5f3)[0x7fb6a844b5f3]
/home/deploy/.cache/JNA/temp/jna1997716433451596580.tmp(Java_com_sun_jna_Native_invokeVoid+0x21)[0x7fb6a844bcf1]
[0x7fb7c7cd37bb]
======= Memory map: ========
......
```
* 错误显示出错的原因是 free(): invalid next size (normal)，没有直接指示与liba.so相关的出错信息，猜测可能是liba.so内发生了数组的越界访问，导致其他线程发生了异常
* 且不同测试数据可能也可能不会触发崩溃，加深了对上一个猜测的怀疑，数据长度过长，可能导致越界访问

2. 排查java代码中与数组相关的操作，发现如下代码
```
    byte[] outBytes = new byte[128];
    int[] outSize = new int[1];
    NativeLibrary.instance.native_mathod(bytes, bytes.length, outBytes, outSize);
```
* 其中outBytes传入so，作为输出数据的缓冲
* 当bytes过长时，outBytes数据长度也会随之增加，当长度超过128时，so内部可能会继续访问随后的地址，造成越界访问
* 最后，增加了outBytes的长度，崩溃随之消失

## 总结
* 相比于直接打印调用栈的情形，这种异常崩溃不与当前使用的本地库相关，更加难以排查，但是思路是数组越界访问这种可能影响到其他数据和线程异常

