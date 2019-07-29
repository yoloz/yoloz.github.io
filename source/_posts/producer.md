---
title: producer
comments: false
toc: false
date: 2019-01-26 15:55:51
categories: kafka
tags:
---
## 核心工作流程

* Producer首先使用用户主线程将待发送的消息封装进一个ProducerRecord类实例中。  
* 进行序列化后，发送给Partioner，由Partioner确定目标分区后，发送到Producer程序中的一块内存缓冲区中。  
* Producer的另一个工作线程（即Sender线程），则负责实时地从该缓冲区中提取出准备好的消息封装到一个批次的内，统一发送给对应的broker中。

<!--more-->

## 主要参数设置

### 参数acks设置

在消息被认为是“已提交”之前，producer需要leader确认的produce请求的应答数。该参数用于控制消息的持久性，目前提供了3个取值：  

* acks = 0: 表示produce请求立即返回，不需要等待leader的任何确认。这种方案有最高的吞吐率，但是不保证消息是否真的发送成功。
* acks = -1: 表示分区leader必须等待消息被成功写入到所有的ISR副本(同步副本)中才认为produce请求成功。这种方案提供最高的消息持久性保证，但是理论上吞吐率也是最差的。
* acks = 1: 表示leader副本必须应答此produce请求并写入消息到本地日志，之后produce请求被认为成功。如果此时leader副本应答请求之后挂掉了，消息会丢失。这是个折衷的方案，提供了不错的持久性保证和吞吐。

> 商业环境推荐：  

如果要较高的持久性要求以及无数据丢失的需求，设置acks = -1。其他情况下设置acks = 1

### 参数buffer.memory设置（吞吐量）

该参数用于指定Producer端用于缓存消息的缓冲区大小，单位为字节，默认值为：33554432(32M)。kafka采用的是异步发送的消息架构，prducer启动时会首先创建一块内存缓冲区用于保存待发送的消息，然后由一个专属线程负责从缓冲区读取消息进行真正的发送。

> 商业环境推荐：  

消息持续发送过程中，当缓冲区被填满后，producer立即进入阻塞状态直到空闲内存被释放出来，这段时间不能超过max.blocks.ms设置的值，一旦超过，producer则会抛出TimeoutException 异常，因为Producer是线程安全的，若一直报TimeoutException，需要考虑调高buffer.memory 了。
用户在使用多个线程共享kafka producer时，很容易把 buffer.memory 打满。

### 参数compression.type设置（lZ4）

producer压缩器，目前支持none（不压缩），gzip，snappy和lz4。

> 商业环境推荐：  

基于公司物联网平台，试验过目前lz4的效果最好。当然2016年8月，FaceBook开源了Ztandard。官网测试： Ztandard压缩率为2.8，snappy为2.091，LZ4 为2.101

### 参数retries设置(注意消息乱序, EOS)

重试时producer会重新发送之前由于瞬时原因出现失败的消息。瞬时失败的原因可能包括：元数据信息失效、副本数量不足、超时、位移越界或未知分区等。倘若设置了retries > 0，那么这些情况下producer会尝试重试

> 商业环境推荐：  

max.in.flight.requests.per.connection设置大于1，设置retries就有可能造成发送消息的乱序；
kafka后期版本已经支持"精确到一次的语义”，因此消息的重试不会造成消息的重复发送

### 参数linger.ms设置(吞吐量和延时性能)

默认是0，表示不做停留，这种情况下，可能有的batch中没有包含足够多的produce请求就被发送出去了，造成了大量的小batch，给网络IO带来的极大的压力

> 商业环境推荐：  

为了减少了网络IO，提升了整体的TPS。假设设置linger.ms=5，表示producer请求可能会延时5ms才会被发送

## Error: NETWORK_EXCEPTION

``` java
Got error produce response with correlation id xxx on topic-partition xxxxx, retrying (9 attempts left). Error: NETWORK_EXCEPTION
Got error produce response with correlation id xxx on topic-partition xxxxx, retrying (9 attempts left). Error: REQUEST_TIMED_OUT

```

暂时定位是producer压力较大，默认配置需要优化(细节可参上述说明)。  
  
> broker可优化参数如下：  

num.network.threads=cpu核数+1  
num.io.threads=cpu核数*2  
socket.send.buffer.bytes=1024000  
socket.receive.buffer.bytes=1024000  
