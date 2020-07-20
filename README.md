# Learn_iptables
linux iptables使用
iptables命令是Linux上常用的防火墙软件，是netfilter项目的一部分。可以直接配置，也可以通过许多前端和图形界面配置。
Arch Linux WIKI上有一个很形象的关于数据通过iptables的示意图：
```
                               XXXXXXXXXXXXXXXXXX
                             XXX     Network    XXX
                               XXXXXXXXXXXXXXXXXX
                                       +
                                       |
                                       v
 +-------------+              +------------------+
 |table: filter| <---+        | table: nat       |
 |chain: INPUT |     |        | chain: PREROUTING|
 +-----+-------+     |        +--------+---------+
       |             |                 |
       v             |                 v
 [local process]     |           ****************          +--------------+
       |             +---------+ Routing decision +------> |table: filter |
       v                         ****************          |chain: FORWARD|
****************                                           +------+-------+
Routing decision                                                  |
****************                                                  |
       |                                                          |
       v                        ****************                  |
+-------------+       +------>  Routing decision  <---------------+
|table: nat   |       |         ****************
|chain: OUTPUT|       |               +
+-----+-------+       |               |
      |               |               v
      v               |      +-------------------+
+--------------+      |      | table: nat        |
|table: filter | +----+      | chain: POSTROUTING|
|chain: OUTPUT |             +--------+----------+
+--------------+                      |
                                      v
                               XXXXXXXXXXXXXXXXXX
                             XXX    Network     XXX
                               XXXXXXXXXXXXXXXXXX
```
iptables的规则大概是iptables -> Tables -> Chains -> Rules

## iptables的表（tables）
iptables有Filter、NAT、Mangle、Raw 、security 5种内建表，在大多数使用情况下都不会用到 raw，mangle 和 security 表

- raw用于配置数据包，raw 中的数据包不会被系统跟踪。
- filter 是用于存放所有与防火墙相关操作的默认表。
- nat 用于 网络地址转换（例如：端口转发）。
- mangle 用于对特定数据包的修改（参考 损坏数据包）,用于指定如何处理数据包。它能改变TCP头中的QoS位。
- security 用于 强制访问控制 网络规则（例如： SELinux）
- iptables的链（chains）
  表是由链构成的
  下面是各个表包含的链:

### Filter
INPUT链 – 处理来自外部的数据。
OUTPUT链 – 处理向外发送的数据。
FORWARD链 – 将数据转发到本机的其他网卡设备上。
NAT
INPUT链 – 处理来自外部的数据。
OUTPUT链 – 处理向外发送的数据。
FORWARD链 – 将数据转发到本机的其他网卡设备上。
Mangle
它由上述Filter和NAT的所有链：

PREROUTING：对数据包作路由选择前应用此链中的规则
            （记住！所有的数据包进来的时侯都先由这个链处理）
OUTPUT
FORWARD
INPUT
POSTROUTING：对数据包作路由选择后应用此链中的规则
            （所有的数据包出来的时侯都先由这个链处理）
PREROUTING 和 POSTROUTING 的简单关系:

源地址发送数据--> {PREROUTING-->路由规则-->POSTROUTING} -->目的地址接收到数据
当你使用：
```
iptables -t nat -A PREROUTING -i eth1 -d 1.2.3.4 -j DNAT --to 192.168.1.40
```
你访问1.2.3.4，linux路由器会在“路由规则”之前将目的地址改为192.168.1.40，并且Linux路由器（iptables）会同时记录下这个连接，并在数据从192.168.1.40返回时，经过linux路由器将数据发送到那台发出请求的机器。所以你的"POSTROUTING"规则没有起作用。
而"POSTROUTING"是“路由规则”之后的动作。

## PREROUTING的应用
一般情况下，PREROUTING应用在普通的NAT中（也就是SNAT），如：你用ADSL上网，这样你的网络中只有一个公网IP地址（如：`61.129.66.5`），但你的局域网中的用户还要上网（局域网IP地址为：192.168.1.0/24），这时你可以使用PREROUTING(SNAT)来将局域网中用户的IP地址转换成61.129.66.5，使他们也可以上网：
```
iptables -t nat -A PREROUTING -s 192.168.1.0/24 -j SNAT 61.129.66.5
```

## POSTROUTING的应用
POSTROUTING用于将你的服务器放在防火墙之后，作为保护服务器使用，例如：
A.你的服务器IP地址为：192.168.1.2；
B.你的防火墙（Linux & iptables）地址为192.168.1.1和202.96.129.5

Internet上的用户可以正常的访问202.96.129.5,但他们无法访问192.168.1.2，这时在Linux防火墙里可以做这样的设置：
iptables -t nat -A POSTROUTING -d 202.96.129.5 -j DNAT 192.168.1.2

结：最要紧的是我们要记住PREROUTING是“路由规则”之前的动作，POSTROUTING是“路由规则”之后的动作！
Raw
含有两条内建链
```
PREROUTING chain
OUTPUT chain
```
# iptables规则（Rules）
一般而言，规则由3条关键点：
1. 包含一个条件和一个目标target
2. 若满足条件，就执行目标的规则或特定值
3. 若不满足，就判断下一条规则

目标target
指定-j后的规则动作
ACCEPT – 允许防火墙接收数据包
DROP – 防火墙丢弃包
QUEUE – 防火墙将数据包移交到用户空间
RETURN – 防火墙停止执行当前链中的后续Rules，并返回到调用链(the calling chain)中。
REDIRECT：重定向、映射、透明代理。
SNAT：源地址转换。
DNAT：目标地址转换。
MASQUERADE：IP伪装（NAT），用于ADSL。
LOG：日志记录。
保存规则
- Ubuntu
```
iptables-save > /etc/iptables.up.rules #文件名自定义
编辑/etc/network/interfaces
并且在 iface lo inet loopback 后增加一行 pre-up iptables-restore < /etc/iptables.up.rules 应用防火墙规则

添加自启动：
sudo apt-get install sysv-rc-conf  # ubuntu
$ sudo sysv-rc-conf iptables on    # ubuntu
```
- Centos
```
# 保存iptables规则
service iptables save #centos5,6
systemctl enable iptables.service #centos7，以下类似

# 重启iptables服务
systemctl restart iptables
service iptables restart

#查看规则
cat  /etc/sysconfig/iptables

#添加自启动
chkconfig iptables on  # redhat/centos  5, 6
systemctl enable iptables.service  #centos7
```
上述内容就是iptables的基本组成部分，下面是配置详解

# iptables的选项配置

基本的配置格式
```
iptables -t 表名 <-A/I/D/R> 规则链名 [规则号] <-i/o 网卡名> -p 协议名 <-s 源IP/源子网> --sport 源端口 <-d 目标IP/目标子网> --dport 目标端口 -j 动作
```

参数说明：
```
-t<表>：指定要操纵的表；
-A：向规则链中添加条目；
-D：从规则链中删除条目；
-i：向规则链中插入条目；
-R：替换规则链中的条目；
-L/--list：显示规则链中已有的条目；
-F：清除规则链中已有的条目；
-X: 杀掉所有使用者 "自定义" 的 chain ;
-Z：清空规则链中的数据包计算器和字节计数器；
-N：创建新的用户自定义规则链；
-P：定义规则链中的默认目标；
-h：显示帮助信息；
-d 目标 IP/网域：同 -s ，只不过这里指的是目标的 IP 或网域；
-p：指定要匹配的数据包协议类型,主要的封包格式有： tcp, udp, icmp 及 all ；
-s：指定要匹配的数据包源ip地址 \
       IP  ：192.168.0.100
       网域：192.168.0.0/24, 192.168.0.0/255.255.255.0 均可。
       若规范为『不许』时，则加上 ! 即可，例如：
       -s ! 192.168.100.0/24 表示不许 192.168.100.0/24 之封包来源；
-j<目标>：指定要跳转的目标，主要的动作有接受(ACCEPT)、丢弃(DROP)、拒绝(REJECT)及记录(LOG)；
-i<网络接口>：指定数据包进入本机的网络接口；
-o<网络接口>：指定数据包要离开本机所使用的网络接口。
-v ：列出更多的信息，包括通过该规则的封包总位数、相关的网络接口等
iptables配置实例讲解
清空已有的规则 谨慎用
iptables -F #清空所有
iptables -X #清空所有用户自定义的

```
查看表的规则
比如NAT表：
```
iptables -t nat -L
```

查看用户自定义的规则
```
iptables -L -n -v
```
删除用户添加的某条规则
先显示出所有用户添加的规则
```
iptables -L -n --line-numbers
```
然后删除指定规则：
```
iptables -D INPUT/OUTPUT/...(链) number
```
开启SSH端口
```
iptables -A INPUT -i eth0 -p tcp --dport 22 -j ACCEPT
iptables -A OUTPUT -i eth0 -p tcp --dport 22 -j ACCEPT
```
*上述的-i可以自己设定*
关闭一部分端口
```
iptables -A OUTPUT -p tcp --sport 31337 -j DROP
iptables -A OUTPUT -p tcp --dport 31337 -j DROP
```
注：
有些特洛伊木马会扫描端口31337到31340(即黑客语言中的 elite 端口)上的服务。既然合法服务都不使用这些非标准端口来通信,阻塞这些端口能够有效地减少你的网络上可能被感染的机器和它们的远程主服务器进行独立通信的机会。此外，其他端口也一样,像:31335、27444、27665、20034 NetBus、9704、137-139（smb）,2049(NFS)端口也应被禁止

设置仅允许特定IP的主机进行ssh连接
这个做法较为偏激
```
iptables -A INPUT -s 192.168.2.10 -p tcp --dport 22 -j ACCEPT
```
如果是指定ip段：
```
iptables -A INPUT -s 192.168.0.0/24 -p tcp --dport 22 -j ACCEPT
```
然后删除原规则的
```
iptables -A INPUT -p tcp -m tcp --dport 22 -j ACCEPT
```
如果是单独屏蔽某个IP不能ssh连接
```
iptables -A INPUT -s !192.168.2.10 -p tcp --dport 22 -j ACCEPT
```
拒绝所有其他的数据包
```
iptables -A INPUT -j DROP
```
屏蔽指定IP
```
BLOCK_IP="x.x.x.x"iptables -A INPUT -i eth0 -p tcp -s "$BLOCK_IP" -j DROP
```
防DDOS
```
iptables -A INPUT -p tcp --dport 80 -m limit --limit 25/minute --limit-burst 100 -j ACCEPT
```
`--litmit 25/minute` 指示每分钟限制最大连接数为25
`--litmit-burst 100` 指示当总连接数超过100时，启动 `litmit/minute` 限制

禁止长时间ping
设置ICMP包过滤, 允许每秒1个包, 限制触发条件是10个包
```
iptables -A FORWARD -p icmp -m limit --limit 1/s --limit-burst 10 -j ACCEPT
```
网口转发
对于用作防火墙或网关的服务器，一个网口连接到公网，其他网口的包转发到该网口实现内网向公网通信，假设eth0连接内网，eth1连接公网
```
iptables -A FORWARD -i eth0 -o eth1 -j ACCEPT
```
