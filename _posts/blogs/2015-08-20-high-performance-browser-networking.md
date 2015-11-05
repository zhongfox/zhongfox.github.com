---
layout: post
categories: [blog, sundry]
tags : [http, tcp, performance, browser, networking]
title: Web性能权威指南读书笔记
---
{% include JB/setup %}

---

"合格的开发者知道这么做, 而优秀的开发者知道为什么那么做"

## 一. 延迟与带宽

* 分组从信息源发到的地所需的时间
* 逻辑或物理通信路径最大的吞吐量

延迟的构成

* 传播延迟 信号传播距离和速度的函数
* 传输延迟 把消息中比特转移到链路中的时间, 是消息长度和链路速率的函数
* 处理延迟
* 排队延迟

几个常量:

* 光在光纤中传播速度约 (20万公里/s)
* 1km 需要传播约 5ms

带宽

光纤的总带宽等于每个信道的数据传输速率乘以可以复用的信道数

网络边缘带宽(终端用户), 带宽很大程度取决于部署技术, 如拨号连接,DSL,各种无线技术,光纤到户,甚至局域网路由器性能等

用户可用带宽取决于客户端与目标服务器之间的最低容量连接(短板理论?)

---

## 二. TCP的构成

<img src="/assets/images/hpbn/tcp_3.png" />

* 序列号起始值都是随机值
* 客户端可以在发送ACK分组之后立即发送数据
* 服务必须等待接收到ACK后才能发送数据


### 流量控制

* 接收窗口(rwnd)
* 每个ACK分组都会携带相应的最新的rwnd值, 以便两端动态调整数据流速

### 慢启动

动态评估可用带宽

* 拥塞窗口大小(cwnd) 发送端对接收到另一端的ACK之前可以发送的数据量限制
* cwnd是各端的私用变量, 端与端不会交换
* 发送端可以发送的最大数据量取rwnd和cwnd的最小值
* 每收到一个TCP段, cwnd加一, 指数增长

### 带宽延迟积

数据链路的容量与其端到端的延迟乘积, 结果就是任意时刻处于在途未确认状态的最大数据量

窗口数据量/往返时间=实际传输速率(带宽)

### 队首阻塞

TCP按序交付和可靠交付导致队首阻塞

### TCP核心原理及影响

* 三次握手增加了一次往返时间
* 慢启动应用到每个新连接
* 流量及拥塞控制影响所有连接的吞吐量
* 吞吐量由当前拥塞窗口大小控制

### 服务器配置调优

* 增大TCP的初始拥塞窗口
* 禁用空闲慢启动
* 启用窗口缩放
* TCP快速打开

### 应用程序行为调优

* 再快也快不过什么也不发, 能少发就少发
* 不能让数据传输更快, 但可以考虑让传输距离更短
* 重用TCP连接是提升性能的关键

### 性能检查清单

* 服务器内核升级到最新版(Linux 3.2+)
* cwnd 为10
* 禁用空闲后慢启动
* 确保启动窗口缩放
* 减少传输冗余数据
* 压缩传输数据
* 服务器部署到离用户近的地方以减少往返时间
* 尽可能重用TCP连接

---

## 十. Web性能要点

**Page Load Time** 浏览器中加载旋转图标停止旋转的时间. 更技术的定义时浏览器中onload事件, 这个事件由浏览器在文档及其所有依赖资源(js,css,图片等)下载完毕时触发.


<img src="/assets/images/hpbn/dkyc.png" />

* DOM和CSSOM共同构建渲染树
* js执行会阻塞DOM解析和构造
* js执行会阻塞CSS的处理
* 渲染和js执行都受CSS的阻塞, 因此需要让CSS最快下载完
* 渲染和脚本执行在同一个线程交错执行, 不可能并发修改生成DOM

### 用户体验

    0 ~100 ms 很快
    100~300 ms 有一点点慢
    300~1000 ms 机器在工作吧
    > 1000 ms 先干点别的吧
    > 10000 ms 不能用了

* 更多带宽其实部(太)重要: 对大文件(在线视频)应用效果明显, 对普通web不太重要
* 延迟是性能瓶颈

这真实TCP握手, 流量和拥塞控制, 由丢包导致的队首拥塞等底层协议特点影响性能的直接后果

大多数HTTP数据流都是小型突发性数据量, 而TCP则是为持久连接和大块数据传输而进行过优化的.

网络往返时间在大多数情况下都是TCP吞吐量和性能的限制因素

### Navigation Timing

<img src="/assets/images/hpbn/nt.png" />

### 浏览器优化

* 基于文档优化
* 推测性优化

浏览器优化主要技术

* 资源预取和排定优先次序
* DNS预解析
* TCP预连接
* 页面预渲染

如何利用浏览器优化:

* CSS和JS等重要资源应该尽早在文档中出现??
* 应该尽早交付CSS, 从而解除渲染阻塞并让JS执行
* 非关键性JS应该推迟, 以避免阻塞DOM和CSSDOM构建
* HTML文档解析器递增解析
* 添加优化tag到html文档中

----

## 十一. HTTP 1.X

### 持久连接的优点

N次请求节省的总延时是(N-1) x RTT, 现代Web应用中, N平均值是90, 而且还在增加

### HTTP 管道

持久连接HTTP必须严格满足先进先出, client只有等待之前的请求响应完成后, 才能发送后续请求

HTTP 管道可以实现client端并行发送请求, server端可以选择并行处理请求, 但是响应还是需要按序返回(http 1.X不支持多路复用)

HTTP 管道存在一些不见文档记载的问题, 因此在web应用中应用有限.

### 使用多个TCP连接

* 作为绕过HTTP限制的权宜之计 (HTTP1.x不支持多路复用)
* 作为绕过TCP低起始拥塞窗口值的权宜之计(还可以升级服务器cwnd为10)
* 作为让客户端绕过不能使用TCP缩放窗口的权宜之计

大部分浏览器并发数量为6

**代价**

* 更多的套接字占用clint和server更多的资源
* 并行TCP流之间竞争带宽
* 复杂度(处理多个套接字)
* 并行TCP流能力也有限制

### 域名分区

好处: 突破浏览器限制

代价:

* 新主机名增加DNS查询
* server要分离资源进行多主机托管

实践中, 把多个域名解析到同一个IP是常见做法, 浏览器连接限制是针对主机名的, 而不是IP


**分区滥用**: 分区过多, 会导致几十个TCP流都得不到充分利用(DNS查询, 慢启动等)


### 度量和控制协议开销

* 每个浏览器发送的http请求, 都会携带额外500~800字节的元数据(主要是header), 还不包括最大的一块cookie
* http首部都是以纯文本进行发送, 不会进过任何压缩(意思gzip是只有内容才压缩??)
* Cookie在很多应用中都是常见的性能瓶颈

### 连接于拼合

* 连接: 多个js或者css组合成一个文件
* 拼合: 多张图片组合为复合图片

优点:

* 减少协议开销
* 应用层管道(相对真实的但是没有流行的http管道)

代价:

* 合并的资源共用一个url(缓存键)
* 合并资源可能包含当前页不需要的内容
* 单个资源的更新导致真个资源包的下载
* 应用内存占用: 浏览器虽然只用图片的一部分, 但是需要加载图片全部内容到内存
* 执行速度的影响: js/css的不允许增量执行(必须下载完整个文件再执行, html是可以增量处理)

### 嵌入资源

嵌入资源是另一种非常流行的优化方法, 可以减少请求次数

代价:

* 不能单独缓存
* 嵌入资源更新将破坏宿主缓存
* js/css嵌入没有多余开销, 但是非文本(图片)必须base64编码, 导致小心增长33%

通常只考虑嵌入1~2k以下的资源

---



