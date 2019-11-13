



最近在准备支持Dubbo在K8s体系下的服务注册。上篇文章中也写到过实现的几种方式，本文主要介绍DNS注册机制的一些基础知识。

DNS的基础知识这里不再赘述，主要从实现上探索DNS服务发现的一些关键路径。

## nslookup介绍

> **nslookup** is a [network administration](https://en.wikipedia.org/wiki/Network_administration) [command-line](https://en.wikipedia.org/wiki/Command-line) tool available in many computer [operating systems](https://en.wikipedia.org/wiki/Operating_system) for querying the [Domain Name System](https://en.wikipedia.org/wiki/Domain_Name_System) (DNS) to obtain [domain name](https://en.wikipedia.org/wiki/Domain_name) or [IP address](https://en.wikipedia.org/wiki/IP_address) mapping, or other [DNS records](https://en.wikipedia.org/wiki/DNS_record). The name "nslookup" means "name server lookup".             ---引用自wikipedia

简单地说nslookup是一种网络管理命令行工具，可用于查询DNS域名和IP地址。 **(以下所有操作均来自于mac系统或者mac docker的镜像)**

nslookup支持交互模式和非交互模式，下面操作示例都是非交互模式。可以通过：`nslookup -type=ns 域名/IP` 进行查找。

```
type支持类型:
    A -->地址记录
    AAAA  
    MX   -->邮件服务器
    NS  --> 名字服务器
 		ATMA
    AFSDB Andrew  
 		CNAME 
    HINHO 
    ISDN
    MB
    MG
    MINFO
    MR    
    PTR
    RP
    RT
    SRV 
    TXT 
    X25
```

默认查找A record：

```
➜  nslookup www.cloudns.net
Server:		XXXXXXXX  (脱敏)
Address:	XXXXXXxx#53

Non-authoritative answer:
Name:	www.cloudns.net
Address: 77.247.178.152
```

查看type=mx的情况：

```
➜  nslookup -type=mx cloudns.net
Server:		XXXXXXXX  (脱敏)
Address:	XXXXXXxx#53

Non-authoritative answer:
cloudns.net	mail exchanger = 5 alt1.aspmx.l.google.com.
cloudns.net	mail exchanger = 5 alt2.aspmx.l.google.com.
cloudns.net	mail exchanger = 1 aspmx.l.google.com.
cloudns.net	mail exchanger = 10 alt4.aspmx.l.google.com.
cloudns.net	mail exchanger = 10 alt3.aspmx.l.google.com.

Authoritative answers can be found from:
alt1.aspmx.l.google.com	internet address = 172.253.112.26
alt1.aspmx.l.google.com	has AAAA address 2607:f8b0:4023::1b
alt2.aspmx.l.google.com	internet address = 173.194.77.26
alt2.aspmx.l.google.com	has AAAA address 2607:f8b0:4023:401::1a
aspmx.l.google.com	internet address = 74.125.127.26
aspmx.l.google.com	has AAAA address 2607:f8b0:4003:c09::1b
alt4.aspmx.l.google.com	internet address = 173.194.68.26
alt4.aspmx.l.google.com	has AAAA address 2607:f8b0:400d:c0c::1a
alt3.aspmx.l.google.com	internet address = 64.233.177.26
alt3.aspmx.l.google.com	has AAAA address 2607:f8b0:4002:c08::1b
```

上述的讲解都是基于nslookup，命令`dig` 也是NDS查找下的很实用的命令。

## A Record

```
➜  logs > nslookup www.google.com
Server:		10.65.0.1
Address:	10.65.0.1#53

Non-authoritative answer:
Name:	www.google.com
Address: 64.233.180.99
Name:	www.google.com
Address: 64.233.180.147
Name:	www.google.com
Address: 64.233.180.103
Name:	www.google.com
Address: 64.233.180.104
Name:	www.google.com
Address: 64.233.180.105
Name:	www.google.com
Address: 64.233.180.106

➜  logs > nslookup cloudns.net
Server:		10.65.1.1
Address:	10.65.1.1#53

Non-authoritative answer:
Name:	cloudns.net
Address: 77.247.178.152
```

发现nslookup有些场景(cloudns.net)下返回单个记录，有些场景(google.com)下返回了多条记录，但是这个都叫A Record.

K8s体系下ClusterIp和Headless Service 的A Record，摘录自[K8s DNS specification](https://github.com/kubernetes/dns/blob/master/docs/specification.md)

#### Records for a Headless Service （A Record）

> There must be an `A` record for each *ready* endpoint of the headless Service with IP address `` as shown below. If there are no *ready* endpoints for the headless Service, the answer should be `NXDOMAIN`.
>
> - Record Format:
>   - `<service>.<ns>.svc.<zone>. <ttl> IN A <endpoint-ip> `
> - Question Example:
>   - `headless.default.svc.cluster.local. IN A`
> - Answer Example:
>
> ```
>     headless.default.svc.cluster.local. 4 IN A 10.3.0.1
>     headless.default.svc.cluster.local. 4 IN A 10.3.0.2
>     headless.default.svc.cluster.local. 4 IN A 10.3.0.3
> ```

ClusterIp和Headless Service方式对Rpc的作用是不一样的，ClusterIp只返回集群Ip，所以集群IP将承担负载均衡的作用，像Dubbo这样的Rpc框架需要自己来掌控路由规则，负载均衡，服务治理等，所以ClusterIp方式不是很适合Rpc框架使用。而Headless Service这种返回IP列表的方式和Rpc框架无缝适配。

## Netty对DNS支持





## K8s下验证





http://127.0.0.1:8888/nslookup?ns=blogspot.in&rt=A
http://127.0.0.1:8888/nslookup?ns=akamaihd.net&rt=MX