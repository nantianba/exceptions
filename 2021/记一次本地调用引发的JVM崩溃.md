# 记一次本地调用引发的JVM崩溃
## 背景
* 应用运行在tomcat上，并使用JNA调用了liba.so中的本地方法
* liba.so文件出现在两个地方，一个是第三方提供的jar包依赖中，一个是放在resources下
* 项目中还有其他so文件libb.so
* 服务单台机器QPS约200

## 症状综述
1. 观察CPU曲线发现周期性的打满，且服务出现大量超时，且线程从正常的200左右上升到400，其中增加的线程多为http线程
2. 随后发现异常原因为生产linux环境调用本地方法时直接JVM崩溃
3. 修复so文件之后，单元测试可通过，但tomcat环境下运行JVM依然崩溃

## 排查过程
1. 乍看异常指标，以为程序中存在异常死循环导致CPU飙升引起大量阻塞，同时引起锁竞争导致200QPS下被阻塞HTTP线程增加，同时阻塞引起超时增加。
   查看catalina.out文件得知，生产环境tomcat在不断异常重启，重启原因为JVM异常崩溃。
```
#
# A fatal error has been detected by the Java Runtime Environment:
#
#  [thread 139713910867712 also had an error]
SIGSEGV (0xb) at pc=0x00007f10fc018871, pid=804369, tid=0x00007f11ae0b1700
#
# JRE version: OpenJDK Runtime Environment (8.0_202-b08) (build 1.8.0_202-b08)
# Java VM: OpenJDK 64-Bit Server VM (25.202-b08 mixed mode linux-amd64 compressed oops)
# Problematic frame:
# C  [【libb.so】+0x5871]  【方法1】+0x1c1
#
# Core dump written. Default location: //core or core.804369
#
# An error report file with more information is saved as:
# /tmp/hs_err_pid804369.log
#
# If you would like to submit a bug report, please visit:
#   http://bugreport.java.com/bugreport/crash.jsp
# The crash happened outside the Java Virtual Machine in native code.
# See problematic frame for where to report the bug.
#
```
* 因此CPU飙升原因为不断重启初始化阶段的CPU消耗，线程上升的原因在于机器拉入集群后接入大量请求，so加载初始化时间比较长，导致请求存在一些锁竞争，
在此过程中不断有生产流量进入，且新的请求也要阻塞等待加载，导致HTTP线程增加。当加载完成后，由于SO调用中发生异常，导致JVM崩溃，随后tomcat重启。

2. 在hs_err_pid.log中查看崩溃的本地方法栈
```
Stack: [0x00007fd0bb376000,0x00007fd0bb3b7000],  sp=0x00007fd0bb3b3680,  free space=245k
Native frames: (J=compiled Java code, j=interpreted, Vv=VM code, C=native code)
C  [【libb.so】+0x5871]  【方法1】+0x1c1
C  [【liba.so】+0xb713]  【方法2】+0x7a
C  [liba.so+0xb8fe]  【方法3】+0x123
C  [liba.so+0x13644]  【方法4】+0x4c
C  0x00007fd0bb3b4620
```
* 根据调用链可以看到，表面上是调用libb.so方法出现的问题，实际是liba.so调用了libb.so中的方法引起的崩溃

3. 单元测试中单独调用liba.so，发现正常通过，因此怀疑是程序中其他地方先加载过libb.so，两个libb.so存在不一致的地方。
于是单元测试中先调用libb.so，再调用liba.so，JVM崩溃，验证了猜想。
```mermaid
liba --> 单元测试通过 
libb --> liba -->JVM crash
```

4. 通知第三方在so中修复了崩溃异常，并更新了项目resources目录下的liba.so，上述的单元测试通过，但发布生产后问题依旧
经过各种实验后无法解决。但突然发现在第三方包中存在libb.so，且与自己项目下resources中的so路径相同。
* 猜测在tomcat环境中加载的so和单元测试中加载的不是同一个，而是加载了JAR中so文件。为验证，手动删除了jar包中的libb.so文件，在本地tomcat环境中启动，未出现crash。
* 猜测成立，于是告知第三方更新JAR包中so文件，发布生产，恢复正常。

* war包中结构包含了WEB-INF、classes、lib三个文件夹，如果项目为多module结构，非servlet的module将会打包为jar放在lib下。
* 对于同一个相对路径下的so文件，可能出现出人意料的加载。
* 单元测试下，idea直接将项目中的so文件放在test-classes目录下，因此加载的行为是可预测的。

## 总结
* 生产JVM崩溃的异常首要排查目标是catalina.out和hs_err_pid.log文件
* 不要盲目相信单元测试，war包有着与单元测试编译类不同的结构
* 提供第三方包时尽量减少用户配置和复杂度，做到即拆即用，避免类似冲突和灵异事件出现

