## 				Android开发高手课

- 耗时分析工具Traceview
- 移动APM质量平台



##### 崩溃优化

1. Java崩溃:Java代码中出现了未捕获异常,导致程序异常退出.
2. Native崩溃:一般都是因为Native代码中访问非法地址,也可能是地址对齐出现了问题,或者发生了程序主动abort,这些都会产生相应的signal信号,导致程序异常退出

##### RAM

手机运行内存(RAM)是手机中作为App运行过程中临时性数据暂时存储的内存介质.

##### 内存造成的问题

1. 异常:OOM、内存分配失败
2. 卡顿:Java内存不足会导致频繁GC.



##### 卡顿优化

