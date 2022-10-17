# 基于 Scapy 编写端口扫描器

## 实验目的

- 掌握网络扫描之端口状态探测的基本原理



## 实验环境

- python + [scapy ](https://scapy.net/) 

- VirtualBox 虚拟机：

  - 攻击者主机（Attacker）：Kali-linux-2022.3

  - 网关（Gateway, GW）：Debian Buster

  - 靶机（Victim）：Kali-linux-2022.3

- 网络拓扑：

  - 攻击者主机 Attacker-kali ( 10.0.2.6/24 )
  - 网关 Gateway-debian
  - 靶机 Victim ( 172.16.111.114 )
  
  ![网络拓扑](img/网络拓扑.jpg)



## 实验要求

- [x] 禁止探测互联网上的 IP ，严格遵守网络安全相关法律法规

- 完成以下扫描技术的编程实现
  - [x] TCP connect scan / TCP stealth scan
  - [x] TCP Xmas scan / TCP fin scan / TCP null scan
  - [x] UDP scan

- [x] 上述每种扫描技术的实现测试均需要测试端口状态为：`开放`、`关闭` 和 `过滤` 状态时的程序执行结果
- [ ] 提供每一次扫描测试的抓包结果并分析与课本中的扫描方法原理是否相符？如果不同，试分析原因；
- [x] 在实验报告中详细说明实验网络环境拓扑、被测试 IP 的端口状态是如何模拟的

- （可选）复刻 `nmap` 的上述扫描技术实现的命令行参数开关 【对比我发的包和nmap发的包】



## 实验准备

- Scapy 安装

  ```bash
  # alpine 安装scapy包
  / # apk add update && apk add scapy
  
  # 查看scapy包信息
  / # apk search scapy
  scapy-doc-2.4.5-r2
  scapy-2.4.5-r2
  ```

- ufw 安装

  ```bash
  / # apk add ufw
  ```



## 实验过程



### 端口状态

- 端口状态查询

  ```python
  ufw status
  ```

- 端口关闭状态：对应端口未开启监听，防火墙未启用

  ```python
  ufw disables
  ```

- 端口开启状态：对应端口开启监听，防火墙处于关闭状态

  ```bash
  systemctl start apache2 # port 80  apache2在80端口提供基于TCO的服务
  systemctl start dnsmasp # port 53  DNS在53端口提供基于UDP的服务
  ```

- 端口过滤状态：对应端口开启监听, 防火墙处于开启状态。

  ```bash
  ufw enable && ufw deny 80/tcp
  ufw enable && ufw deny 53/udp
  ```

  

### TCP connect scan

> 先发送一个S，然后等待回应。如果有回应且标识为RA，说明目标端口处于关闭状态；如果有回应且标识为SA，说明目标端口处于开放状态。这时TCP connect scan会回复一个RA，在完成三次握手的同时断开连接.
>
> ```
> ufw enable && ufw deny 80/tcp
> ufw enable && ufw deny 53/udp
> ```

**TCP-connect-scan code**

```python
from scapy.all import *

src_port = RandShort()
dst_ip = "172.16.111.124"
dst_port = 80

resp = sr1(IP(dst=dst_ip)/TCP(sport=src_port,dport=dst_port,flags="S"),timeout=10)

if(resp.haslayer(TCP)):
    if (resp.getlayer(TCP).flags == 0x14):   
        print "Closed"
    elif(resp.getlayer(TCP).flags == 0x12): 
        send_rst = sr(IP(dst=dst_ip)/TCP(sport=src_port,dport=dst_port,flags="AR"),timeout=10)
        print "Open"


elif resp is None:
    print "Filtered"
```

**nmap 复刻**

```python
nmap -sT -p 80 172.16.111.124
```

- 端口关闭：
  - code
  - nmap
- 端口开启：
  - code
  - nmap
- 端口过滤：
  - code
  - nmap

### TCP stealth scan

> 先发送一个S，然后等待回应。如果有回应且标识为RA，说明目标端口处于关闭状态；如果有回应且标识为SA，说明目标端口处于开放状态。这时TCP stealth scan只回复一个R，不完成三次握手，直接取消建立连接。

**TCP-stealth-scan code**

```python
from scapy.all import *

src_port = RandShort()
dst_ip = "172.16.111.2" 
dst_port = 80

stl_resp = sr1(IP(dst=dst_ip)/TCP(sport=src_port,dport=dst_port,flags="S"),timeout=10)

if(resp.haslayer(TCP)):
    if (resp.getlayer(TCP).flags == 0x14):
        print "Closed"
    elif(resp.getlayer(TCP).flags == 0x12):
        send_rst = sr(IP(dst=dst_ip)/TCP(sport=src_port,dport=dst_port,flags="R"),timeout=10)
        print "Open"

elif stl_resp is None:
    print "Filtered"
```

**nmap**

```bash
nmap -sS -p 80 172.16.111.2
```

- 端口关闭：
  - code
  - nmap
- 端口开启：
  - code
  - nmap
- 端口过滤：
  - code
  - nmap



### TCP Xmas scan

> 一种隐蔽性扫描，当处于端口处于关闭状态时，会回复一个RST包；其余所有状态都将不回复。

**TCP-Xmas-scan code**

```python
from scapy.all import *

dst_ip = "172.16.111.2"
dst_port = 80

resp = sr1(IP(dst=dst_ip)/TCP(dport=dst_port,flags="FPU"),timeout=10)

if(resp.haslayer(TCP)):
    if(resp.getlayer(TCP).flags == 0x14):
        print "Closed"

elif resp is None:
    print "Open|Filtered"

#elif(resp.haslayer(ICMP)):
#    if(int(resp.getlayer(ICMP).type)==3 and int(resp.getlayer(ICMP).code) in [1,2,3,9,10,13]):
#        print "Filtered"
```

**nmap**

```bash
nmap -sX -p 80 172.16.111.2
```

- 端口关闭：
  - code
  - nmap
- 端口开启：
  - code
  - nmap
- 端口过滤：
  - code
  - nmap



### TCP FIN scan

> 仅发送FIN包，FIN数据包能够通过只监测SYN包的包过滤器，隐蔽性较SYN扫描更⾼，此扫描与Xmas扫描也较为相似，只是发送的包未FIN包，同理，收到RST包说明端口处于关闭状态；反之说明为开启/过滤状态。

**TCP-FIN-scan code**

```python
from scapy.all import *

dst_ip = "172.16.111.2" 
dst_port = 80

resp = sr1(IP(dst=dst_ip)/TCP(dport=dst_port,flags="F"),timeout=10)

if(resp.haslayer(TCP)):
    if(resp.getlayer(TCP).flags == 0x14):
        print "Closed"

elif resp is None:
    print "Open|Filtered"

#elif(resp.haslayer(ICMP)):
#    if(int(resp.getlayer(ICMP).type)==3 and int(resp.getlayer(ICMP).code) in [1,2,3,9,10,13]):
#        print "Filtered"
```

**nmap**

```bash
nmap -sF -p 80 172.16.111.2
```

- 端口关闭：
  - code
  - nmap
- 端口开启：
  - code
  - nmap
- 端口过滤：
  - code
  - nmap



### TCP NULL scan

> 发送的包中关闭所有TCP报⽂头标记，实验结果预期还是同理：收到RST包说明端口为关闭状态，未收到包即为开启/过滤状态.

**TCP-NULL-scan code**

```python
from scapy.all import *

dst_ip = "172.16.111.2" 
dst_port = 80

resp = sr1(IP(dst=dst_ip)/TCP(dport=dst_port,flags=""),timeout=10)

if(resp.haslayer(TCP)):
    if(resp.getlayer(TCP).flags == 0x14):
        print "Closed"

elif resp is None:
    print "Open|Filtered"

#elif(resp.haslayer(ICMP)):
#    if(int(resp.getlayer(ICMP).type)==3 and int(resp.getlayer(ICMP).code) in [1,2,3,9,10,13]):
#        print "Filtered"
```

**nmap**

```bash
nmap -sN -p 80 172.16.111.2
```

- 端口关闭：
  - code
  - nmap
- 端口开启：
  - code
  - nmap
- 端口过滤：
  - code
  - nmap



### UDP scan

> 一种开放式扫描，通过发送UDP包进行扫描。当收到UDP回复时，该端口为开启状态；否则即为关闭/过滤状态.

**UDP-scan code**

```python
from scapy.all import *

dst_ip = "172.16.111.2"
dst_port = 53
dst_timeout = 10

def udp_scan(dst_ip,dst_port,dst_timeout):
    resp = sr1(IP(dst=dst_ip)/UDP(dport=dst_port),timeout=dst_timeout)
    if(resp.haslayer(ICMP)):
        if(int(resp.getlayer(ICMP).type)==3 and int(resp.getlayer(ICMP).code)==3):
            return "Closed"
    elif resp is None:
        return "Open|Filtered"
#   elif(int(resp.getlayer(ICMP).type)==3 and int(resp.getlayer(ICMP).code) in [1,2,9,10,13])
#       print "Filtered"

print(udp_scan(dst_ip,dst_port,dst_timeout))
```

##### nmap

```bash
nmap -sU -p 53 172.16.111.2
```

- 端口关闭：
  - code
  - nmap
- 端口开启：
  - code
  - nmap
- 端口过滤：
  - code
  - nmap

## 课后思考题

- 通过本章网络扫描基本原理的学习，试推测应用程序版本信息的扫描原理，和网络漏洞的扫描原理。
- 网络扫描知识库的构建方法有哪些？

## 参考资料

- [alpine 常用命令](https://blog.csdn.net/liumiaocn/article/details/87603628)
- [自己动手编程实现并讲解TCP connect scan/TCP stealth scan/TCP XMAS scan/UDP scan](https://www.likecs.com/show-203803956.html)

- [师哥作业](https://github.com/CUCCS/2020-ns-public-LyuLumos/blob/ch0x01/ch0x05)

