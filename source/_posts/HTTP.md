---
title: HTTP
categories:
  - 网络协议
  - HTTP
tags:
  - HTTP
date: 2019-12-01 20:13:07
---



Http1.1虽然有着pipeline，但响应和请求还是按顺序出现的。如果前面的响应阻塞了，后面的响应也会跟着阻塞。

 <!-- more --> 

Http2引入了多路复用，每个响应带有stream id，一个连接里可以穿插着多个stream id，可以不按照顺序出现。Http流量控制可以对单个stream进行。而TCP的流控只能对一个连接进行控制，粒度没有TCP的细。

