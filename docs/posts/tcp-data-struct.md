---
title: TCP 协议学习笔记——数据结构
authors:
  - tomczhen
date: 2019-06-09T21:00:00+08:00
categories:
    - Network
tags:
    - TCP
---
> 参考资料：
>
> * [RFC 793 - TRANSMISSION CONTROL PROTOCOL](https://tools.ietf.org/html/rfc793)

<!-- more -->

## TCP Header

```
 0                   1                   2                   3   
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Acknowledgment Number                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Data |     |N|C|E|U|A|P|R|S|F|                               |
|       | RSV | |W|C|R|C|S|S|Y|I|         Window size           |
| Offset|     |S|R|E|G|K|H|T|N|N|                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum            |         Urgent Pointer        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options                    |    Padding    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          payload                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

| 字段 | 长度 | 说明 |
| :--- | :---: | :--- |
| Source Port | 16 bits | 源端口
| Destination port | 16 bits | 目标端口
| Sequence Number | 32 bits |
| Acknowledgment Number | 32 bits |
| Data Offset | 4 bits |
| Reserved | 3 bits | 保留位，默认为 0
| Flags | 9 bits |
| Window size | 16 bits |
| Checksum | 16 bits |
| Urgent pointer | 16 bits
| Options | 0 ~ 320 bits |
| Padding | 0 ~ 31 bits | 确保 TCP Header 大小为 32 bits 的整数倍

### Checksum

TCP 使用校验和的方式校验，长度为 16 bits，需要包括一个伪头部。

#### 伪头部(Pseudo-Header)

```
+--------+--------+--------+--------+
|           Source Address          |
+--------+--------+--------+--------+
|         Destination Address       |
+--------+--------+--------+--------+
|  zero  |  PTCL  |    TCP Length   |
+--------+--------+--------+--------+
```

| 字段 | 长度 | 说明 |
| :--- | :---: | :--- |
| Source Address | 32 bits | 源地址
| Destination Address | 32 bits | 目标地址
| zero | 8 bits | 0 填充
| PTCL | 8 bits | TCP 协议号
| TCP Length | 16 bits | TCP 长度

其中 Source Address 与 Destination Address 来 IP Header，TCP 协议号为 0x06 ([List of IP protocol numbers](https://en.wikipedia.org/wiki/List_of_IP_protocol_numbers)) ，TCP Length 为 TCP Header + Payload 的长度（不包含 Pseudo-Header）

```python
from struct import pack
def pseudo_header(src_addr:int,dst_addr:int,length:int)->bytes:
    zero_padding = 0
    protocol_num = 0x06
    header = pack('!LLBBH',src_addr,dst_addr,zero_padding,protocol_num,length)
    return header
```

#### 计算方法

```
 0                   1                   
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|             Pseudo-Header             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                Header                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
/               Payload                 /
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

将 Pseudo-Header、TCP Header、TCP Payload 按 16 bits 分段，如果最后一个段不满 16 bits，则使用 0 填充，在计算 Checksum 时 TCP Header 中的 Checksum 替换为 0。

```python
from struct import pack, unpack
from functools import reduce
def checksum(pseudo_header:bytes,header:bytes,payload:bytes):
    payload = b''.join(pseudo_header,header,payload)
```