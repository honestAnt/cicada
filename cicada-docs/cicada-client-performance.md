[TOC]  

# 客户端性能说明文档  

## 概要  
本文档主要阐述Cicada客户端性能相关的问题  
 
## FAQ & 解决方案  
### 1、采用何种方案减少对应用程序的影响？  
cicada客户端主要涉及到两个功能：一个是日志收集功能，一个是将收集好的日志发送到远程服务器的功能。前者通常耗时较少，且没太大优化空间；后者涉及到IO，处理较慢，性能优化主要针对后者。  
**最终方案：批处理+异步发送。**  
 
### 2、都采用了那些措施增加日志吞吐量？  
一、批处理  
二、用高性能、低延迟的消息处理框架Disruptor替换BlockingQueue作为线程间传递消息的框架，提升消息处理效率；
 
### 3、日志传送过程中由于第三方原因（日志收集服务器挂掉、网络异常等）导致消息处理速度过慢，消息堆积可能导致内存溢出，如何处理？  
    增加了连接超时设置和传输超时设置，超过一定时长的日志直接扔掉  
 
### 4、对于各种原因（如程序异常）导致的单位时间内抓取的日志量过多的情况，如何处理？  
限流处理，对于超过流量限制的消息，直接做扔弃处理。默认TPS限制为2048条/s，此限制可以设置。   
## 测试结果 
### 测试环境  
| 环境 | 值 |    
| ---  | --- |    
| CPU | Intel(R) Core(TM) i7-4790 CPU @ 3.60GHz  8 cores |  
| RAM | 8 GB，1600MHz |  
| 硬盘IO | 1 TB，7200rpm |  
| 网卡 | 1 Gigabit Ethernet |    
| 操作系统 | 1 SMP Debian 3.16.7-ckt11-1+deb8u2 (2015-07-17) x86_64 GNU/Linux |  
| 网络ping值（ip具体值替换成*） | 64 bytes from 10.141.4. *: icmp_seq=2 ttl=55 time=0.718 ms<br/> 64 bytes from 10.141.4.*: icmp_seq=3 ttl=55 time=0.694 ms<br/>64 bytes from 10.141.4.*: icmp_seq=4 ttl=55 time=0.711 ms |   

###  1 、对应用响应时间的影响  
具体步骤如下：  
1、  记录没有Cicada的响应时间，记做A  
2、  记录有Cicada的响应时间，记做B  
3、  处理Cicada的时间 = B – A  
    为了保证数据的准确性，可以多测试几次，求平均值  
 
#### 测试最坏情况下，处理cicada的时间  
  一、1个RPC包含3条cicada  
  二、每个请求测试1w次，并发数为1，线性发送  

|序数|无cicada时处理时间（单位:ms/条）|有cicada时的处理时间(单位：ms/条)|单条处理时间(单位：ms/条)  |  
|----|---|---|---|  
|1|0.28|1.077|0.266|  
|2|0.234|0.995|0.254|  
|3|0.239|1.006|0.256|  
 
#### 测试并发情况下，处理cicada的时间  
一、同上  
二、每个请求测试1w次，并发数100  

| 序数 | 无cicada时处理时间（单位:ms/条）| 有cicada时的处理时间(单位：ms/条) | 单条处理时间(单位：ms/条) |  
| --- | --- | --- | --- |  
| 4 | 0.081 | 0.483 | 0.134 |  
| 5 | 0.068 | 0.495 | 0.142 |  
 
<b>总结：通常情况下，一个客户端请求，会调用3次cicada，按照上面的测试数据来看，时间消耗在0.415ms-0.775ms之间，请求越多处理时间越快（批处理的缘故）。对于多数应用来说这点时间上的影响基本可以忽略。</b>  
 
### 2、  系统资源使用情况  
测试方式：在系统中模拟用户行为并发发送日志请求。由于cicada客户端占用的资源较少，  
Cpu和mem的占比总是在不停的变化中，以下数据是截取某一个时刻的数值，仅作参考。   

|并发请求数 |Cpu%（共800%）|Mem%|  
|---|----|---|  
|75|2%|7% |  
|150|4%|7% |  
|225|5%|9.3% |  
|1000|18.3%|9.6%|    
<b>总结：随着并发数的增加，cpu%的增长基本成线性，并且数值较少，内存的变化也相对较少。</b>  