---
title: 经典syn丢包问题排查
date: 2025-04-18 00:44:42
tags: [网络,丢包]
---

## 背景

> auth/base服务迁移到新虚机(5.4.x  kernel)

## 现象

> auth/base服务偶现 请求 console.devops.xxxx.com 80 端口超时现象

## 排查

### 排除机器性能问题

> 从监控看没有性能问题，怀疑可能机器性能不够导致丢包. 从nginx日志看上线后来自这2个服务请求量是之前2倍+

- 操作

  > 将console.devops.xxxx.com nginx机器从2C4G 升级到4C8G , 且机器也重启了

### 重现问题排除网络问题

> 通过源端telnet 10.103.x.vip 80 其实就能很好的重现问题。
>
> 目标段抓包 也能比较明显的看到目标段收到syn包后没有回包。说明问题是出在目标段，排除网络问题

### 排除VIP问题

- 打通auth/base 到 10.103.x.a/b 实际ip并做了硬解析

  >  硬解析后， 超时的问题出现少了。但是base服务还是会超时。 auth服务没有发生超时。
  >
  >  当然我们知道问题并没有解决， 因为telnet 10.103.x.b 80 仍然有问题。
  >
  >  09:07:00.261785 IP 10.103.x.c.53560 > 10.103.x.b.80: Flags [S], seq 730499767, win 64240, options [mss 1460,sackOK,TS val 1341202159 ecr 0,nop,wscale 7], length 0
  >  09:07:01.311726 IP 10.103.x.c.53560 > 10.103.x.b.80: Flags [S], seq 730499767, win 64240, options [mss 1460,sackOK,TS val 1341203210 ecr 0,nop,wscale 7], length 0
  >  09:07:03.359850 IP 10.103.x.c.53560 > 10.103.x.b.80: Flags [S], seq 730499767, win 64240, options [mss 1460,sackOK,TS val 1341205258 ecr 0,nop,wscale 7], length 0
  >  09:07:07.391755 IP 10.103.x.c.53560 > 10.103.x.b.80: Flags [S], seq 730499767, win 64240, options [mss 1460,sackOK,TS val 1341209290 ecr 0,nop,wscale 7], length 0
  >  09:07:15.839834 IP 10.103.x.c.53560 > 10.103.x.b.80: Flags [S], seq 730499767, win 64240, options [mss 1460,sackOK,TS val 1341217738 ecr 0,nop,wscale 7], length 0
  >  09:07:32.224230 IP 10.103.x.c.53560 > 10.103.x.b.80: Flags [S], seq 730499767, win 64240, options [mss 1460,sackOK,TS val 1341234122 ecr 0,nop,wscale 7], length 0
  >  09:08:04.480167 IP 10.103.x.c.53560 > 10.103.x.b.80: Flags [S], seq 730499767, win 64240, options [mss 1460,sackOK,TS val 1341266378 ecr 0,nop,wscale 7], length 0
  >  09:08:04.480213 IP 10.103.x.b.80 > 10.103.x.c.53560: Flags [S.], seq 1265105279, ack 730499768, win 28960, options [mss 1460,sackOK,TS val 1335626468 ecr 1341266378,nop,wscale 7], length 0

### 确定问题 

> 通过10.103.x.c 机器重现问题
>
> 源地址系统内核版本:  5.4.247
>
> 目标nginx内核版本:  3.10.0

#### 重现步骤

- 开3个窗口 

- 窗口1（docker 容器内）: 

  > 容器内的source-ip 是做了SNAT 的 

- ```
  docker run -it artifacts.xxxxx.com/docker-repo/library/busybox:latest /bin/sh
  ```

- 窗口2(主机): 

  ```
  telnet 10.103.x.b 80
  ```

- 窗口1

  ```
  telnet 10.103.x.b 80 失败
  ```

- nginx服务器:

  ```
  tcpdump -i eth0 -nn  "tcp and  host 10.103.x.c and  port 80 and (tcp[tcpflags] & 2 != 0)"
  ```

  > 可以观察到syn 包一直在发， 没有syn-ack包

- nginx服务器修改 tcp_recyle

  ```
  #/etc/sysctl.conf
  net.ipv4.tcp_tw_recycle = 0
  net.ipv4.tcp_timestamps = 1
  
  sysctl -p
  
  sysctl -a | grep tw_recy
  ```

- 再次重现出现不了syn包被丢弃的现象

#### [分析]为什么丢包

服务端的内核参数 net.ipv4.tcp_tw_recycle 和 net.ipv4.tcp_timestamps 的值都为 1时，服务器会检查每一个 SYN报文中的时间戳（Timestamp，跟同一ip下最近一次 FIN包时间对比），若 Timestamp 不是递增的关系，就扔掉这个SYN包

那也就是说虽然在窗口1 和 窗口B  都是在相同主机， 但是前者是容器网络， 后者实主机网络， 他们访问10.103.x.b 80。所生成的 tcp timestamp不是严格递增的。

回到线上确实是模拟的这种场景。 auth/base是容器网络， 还有其他服务是主机网络， 但是他们都会访问 console.devops.xxx.com 

 #### 基于重现步骤总结

> 基于重现了问题看可以确定是
>
> net.ipv4.tcp_tw_recycle = 1
> net.ipv4.tcp_timestamps = 1
>
> 这两个内核参数的作用下。 来自同一个NAT -ip的syn包因为TSVal 值比之前的Fin TSVal值小导致的丢包 
>
> 这两个内核参数的作用下。 来自同一个NAT -ip的syn包因为TSVal 值比之前的Fin TSVal值小导致的丢包 



## 再深入一步，怎么观测到 

### 基于systemTap

- 基于重现步骤我们在目标段可以抓到如下的包

> 1. 16:37:56.290959 IP 10.103.x.c.56564 > 10.103.x.b.80: Flags [S], seq 2325280669, win 64240, options [mss 1460,sackOK,TS val 2098268712 ecr 0,nop,wscale 7], length 0
>
> 2. 16:37:56.291007 IP 10.103.x.b.80 > 10.103.x.c.56564: Flags [S.], seq 185803862, ack 2325280670, win 28960, options [mss 1460,sackOK,TS val 1103418279 ecr 2098268712,nop,wscale 7], length 0
>
> 3. 16:37:56.291614 IP 10.103.x.c.56564 > 10.103.x.b.80: Flags [.], ack 1, win 502, options [nop,nop,TS val 2098268713 ecr 1103418279], length 0
>
> 4. 16:38:02.862396 IP 10.103.x.c.56564 > 10.103.x.b.80: Flags [P.], seq 1:6, ack 1, win 502, options [nop,nop,TS val 2098275284 ecr 1103418279], length 5: HTTP
>
> 5. 16:38:02.862434 IP 10.103.x.b.80 > 10.103.x.c.56564: Flags [.], ack 6, win 227, options [nop,nop,TS val 1103424851 ecr 2098275284], length 0
>
> 6. 16:38:02.862754 IP 10.103.x.b.80 > 10.103.x.c.56564: Flags [P.], seq 1:326, ack 6, win 227, options [nop,nop,TS val 1103424851 ecr 2098275284], length 325: HTTP: HTTP/1.1 400 Bad Request
>
> 7. 16:38:02.862777 IP 10.103.x.b.80 > 10.103.x.c.56564: Flags [F.], seq 326, ack 6, win 227, options [nop,nop,TS val 1103424851 ecr 2098275284], length 0
>
> 8. 16:38:02.863024 IP 10.103.x.c.56564 > 10.103.x.b.80: Flags [.], ack 326, win 500, options [nop,nop,TS val 2098275285 ecr 1103424851], length 0
>
> 9. 16:38:02.863261 IP 10.103.x.c.56564 > 10.103.x.b.80: Flags [F.], seq 6, ack 327, win 501, options [nop,nop,TS val *2098275285* ecr 1103424851], length 0
>
> 10. 16:38:02.863274 IP 10.103.x.b.80 > 10.103.x.c.56564: Flags [.], ack 7, win 227, options [nop,nop,TS val 1103424851 ecr 2098275285], length 0
>
> 
>
>
> 11. 16:38:10.624991 IP 10.103.x.c.34496 > 10.103.x.b.80: Flags [S], seq 1325490191, win 64240, options [mss 1460,sackOK,TS val 1109072523 ecr 0,nop,wscale 7], length 0
> 12. 16:38:11.646833 IP 10.103.x.c.34496 > 10.103.x.b.80: Flags [S], seq 1325490191, win 64240, options [mss 1460,sackOK,TS val 1109073546 ecr 0,nop,wscale 7], length 0
> 13. 16:38:13.695027 IP 10.103.x.c.34496 > 10.103.x.b.80: Flags [S], seq 1325490191, win 64240, options [mss 1460,sackOK,TS val 1109075594 ecr 0,nop,wscale 7], length 0

> 上面的包 
>
> 1-10 是正常的一次telnet 的包
>
> 11-13  是目标端不断收到syn 包， 因为源端没有收到syn-ack导致 源端按照1,2,4..时间间隔重复发送syn包   
>
> 通过这个包我们知道问题是出现在这台机器 ， 因为他没有发送syn-ack回包 
>
> 成功的syn包 TSVal:  2098275285  结合下面的systemtap脚本 可以看到 上次的tcp tsval 是来自 10.103.x.c- ip的最后一次Fin 包里的 TS val
> 失败的syn包 TSVal:  1109072523

- 可观测手段基于systemtap 脚本观测到的

  ```check.stap
  global tm_tcpm_ts
  
  probe begin {
          printf("Starting detecting...\n")
  }
  
  // $tm can't read in function tcp_peer_is_proven in our envirionment (kernel 3.10.0-693.11.1.el7.x86_64).
  // So the alternative way is read it from another function tcpm_check_stamp which is called in tcp_peer_is_proven.
  probe kernel.function("tcpm_check_stamp").return {
      tm_tcpm_ts = $tm->tcpm_ts;
  }
  
  // Print tm->tcpm_ts, req->ts_recent, their delta and return code. 
  probe kernel.function("tcp_peer_is_proven").return {
      //tm_tcpm_ts = @cast(tcp_get_metrics_req($req, $dst), "struct tcp_metrics_block")->tcpm_ts;
      if ($return==0) {
          printf("tm->tcpm_ts:\t%d\treq->ts_recent:\t%d\tdelta:%d\treturn:\t%d\n",
  			tm_tcpm_ts,
  			$req->ts_recent,
  			(tm_tcpm_ts - $req->ts_recent),
  			$return);
      }
  }
  
  ```

  ```
  # stap  check.stap
  Starting detecting...
  # 1109072523 < 2098275285 导致被丢包
  tm->tcpm_ts:	2098275285	req->ts_recent:	1109072523	delta:989202762	return:	0
  tm->tcpm_ts:	2098275285	req->ts_recent:	1109073546	delta:989201739	return:	0
  tm->tcpm_ts:	2098275285	req->ts_recent:	1109075594	delta:989199691	return:	0
  ```

### 可观测2

> 通过netstat， 经过此次问题之后我们目前将netstat 几个可能导致丢包都采集了, nginx入口机器建议都采集 

```
netstat -s | grep "passive connections rejected because of time stamp"  也能看到增加
```

## 解决方案及结论

> 源机器：内核 5.4.247 目标机器:  内核 3.10.x 
> 源机器docker 容器 host 和 容器 bridge 两种模式同时请求  目标机器， 会触发 syn包被丢弃

- 解决:

```
# 目标机器禁用 tcp_tw_recyle  ，注意此参数在linux  4.12之后已经被移出了
#/etc/sysctl.conf
net.ipv4.tcp_tw_recycle = 0
net.ipv4.tcp_timestamps = 1

sysctl -p

sysctl -a | grep tw_recy
```
