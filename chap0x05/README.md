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

  - 攻击者主机 Attacker-kali ( 172.16.111.125 )
  - 网关 Gateway-debian ( 172.16.111.1 )
  - 靶机 Victim-kali ( 172.16.111.119)
  
  ![网络拓扑](img/网络拓扑.jpg)



## 实验要求

- [x] 禁止探测互联网上的 IP ，严格遵守网络安全相关法律法规

- 完成以下扫描技术的编程实现
  - [x] TCP connect scan / TCP stealth scan
  - [x] TCP Xmas scan / TCP fin scan / TCP null scan
  - [x] UDP scan

- [x] 上述每种扫描技术的实现测试均需要测试端口状态为：`开放`、`关闭` 和 `过滤` 状态时的程序执行结果
- [x] 提供每一次扫描测试的抓包结果并分析与课本中的扫描方法原理是否相符？如果不同，试分析原因；
- [x] 在实验报告中详细说明实验网络环境拓扑、被测试 IP 的端口状态是如何模拟的

- （可选）复刻 `nmap` 的上述扫描技术实现的命令行参数开关 【对比我发的包和nmap发的包】



## 实验准备

- Scapy 安装

  ```bash
  sudo apt-get install scapy
  ```
  
- ufw 安装

  ```bash
  sudo apt-get install ufw
  ```



## 实验过程



### 端口状态

- 端口状态查询

  ```python
  ufw status
  ```

- 端口关闭状态：对应端口未开启监听，防火墙未启用

  ```python
  ufw disable
  ```

- 端口开启状态：对应端口开启监听，防火墙处于开启状态

  ```bash
  ## 开启端口
  sudo ufw enable && sudo ufw allow 80/tcp
  sudo ufw enable && sudo ufw allow 53/udp
  
  ## apache2在80端口提供基于TCP的服务
  systemctl start apache2 # port 80 
  systemctl start dnsmasp # port 53
  
  ## 开启端口
  iptables -I INPUT -p tcp --dport 80 -j ACCEPT
  iptables -I INPUT -p ucp --dport 53 -j ACCEPT
  
  ## 查看端口信息
  iptables -L -n
  
  ```

- 端口过滤状态：对应端口开启监听, 防火墙处于开启状态。

  ```bash
  ufw enable && ufw deny 80/tcp
  ufw enable && ufw deny 53/udp
  
  ## 端口过滤
  iptables -I INPUT -p tcp --dport 80 -j REJECT --reject-with tcp-reset
  ```
  
- 有关 `iptables` 的操作

  ```bash
  # 保存 iptables 信息到文件
  sudo iptables-save -c > iptables.rules
  # 修改后，保存重启
  iptables-restore < iptables.rules
  ```

  

### TCP connect scan

> 先发送一个S，然后等待回应。如果有回应且标识为RA，说明目标端口处于关闭状态；如果有回应且标识为SA，说明目标端口处于开放状态。这时TCP connect scan会回复一个RA，在完成三次握手的同时断开连接.
>

**TCP-connect-scan code**

```python
from scapy.all import *


def tcpconnect(dst_ip, dst_port, timeout=10):
    pkts = sr1(IP(dst=dst_ip)/TCP(dport=dst_port,flags="S"),timeout=timeout)
    if pkts is None:
        print("Filtered")
    elif(pkts.haslayer(TCP)):
        if(pkts.getlayer(TCP).flags == 0x12):  #Flags: 0x012 (SYN, ACK)
            send_rst = sr(IP(dst=dst_ip)/TCP(dport=dst_port,flags="AR"),timeout=timeout)
            print("Open")
        elif (pkts.getlayer(TCP).flags == 0x14):   #Flags: 0x014 (RST, ACK)
            print("Closed")

tcpconnect('172.16.111.119', 80)

```

**nmap 复刻**

```bash
nmap -sT -p 80 172.16.111.119
```

- 端口关闭：
  - code
  
    ![tcp-connect-closed](img/tcp-connect-closed.jpg)
  
    ![tcp-connect-wireshark-1](img/tcp-connect-wireshark-1.jpg)
  
  - nmap
  
    ![tcp-connect-wireshark-1](img/tcp-connect-wireshark-1.jpg)
  
- 端口开启：
  - code
  
    ![tcp-connect-open-code](img/tcp-connect-open-code.jpg)
  
    ![tcp-connect-open-ws](img/tcp-connect-open-ws.jpg)
  
  - nmap
  
    ![tcp-connect-open-nmap](img/tcp-connect-open-nmap.jpg)
  
- 端口过滤：
  - code
  
    ![tcp-connect-filterd](img/tcp-connect-filterd.jpg)
  
    ![tcp-connect-filterd-ws](img/tcp-connect-filterd-ws.jpg)
  
  - nmap
  
    ![tcp-connect-filterd-nmap](img/tcp-connect-filterd-nmap.jpg)

### TCP stealth scan

> 先发送一个S，然后等待回应。如果有回应且标识为RA，说明目标端口处于关闭状态；如果有回应且标识为SA，说明目标端口处于开放状态。这时TCP stealth scan只回复一个R，不完成三次握手，直接取消建立连接。

**TCP-stealth-scan code**

```python
from scapy.all import *


def tcpstealthscan(dst_ip, dst_port, timeout=10):
    pkts = sr1(IP(dst=dst_ip)/TCP(dport=dst_port, flags="S"), timeout=10)
    if (pkts is None):
        print("Filtered")
    elif(pkts.haslayer(TCP)):
        if(pkts.getlayer(TCP).flags == 0x12):
            send_rst = sr(IP(dst=dst_ip) /
                          TCP(dport=dst_port, flags="R"), timeout=10)
            print("Open")
        elif (pkts.getlayer(TCP).flags == 0x14):
            print("Closed")
        elif(pkts.haslayer(ICMP)):
            if(int(pkts.getlayer(ICMP).type) == 3 and int(stealth_scan_resp.getlayer(ICMP).code) in [1, 2, 3, 9, 10, 13]):
                print("Filtered")


tcpstealthscan('172.16.111.119', 80)
```

**nmap**

```bash
nmap -sS -p 80 172.16.111.119
```

- 端口关闭：
  - code
  
    ![stealth-close-code](img/stealth-close-code.jpg)
  
  - nmap
  
    ![stealth-close-nmap](img/stealth-close-nmap.jpg)
  
- 端口开启：
  - code
  
    ![stealth-open-code](img/stealth-open-code.jpg)
  
  - nmap
  
    ![stealth-open-nmap](img/stealth-open-nmap.jpg)
  
- 端口过滤：
  - code
  
    ![tcp-stealth-code](img/tcp-stealth-code.jpg)
  
    ![tcp-stealth-code-ws](img/tcp-stealth-code-ws.jpg)
  
  - nmap
  
    ![tcp-stealth-code-nmap](img/tcp-stealth-code-nmap.jpg)



### TCP Xmas scan

> 一种隐蔽性扫描，当处于端口处于关闭状态时，会回复一个RST包；其余所有状态都将不回复。

**TCP-Xmas-scan code**

```python
from scapy.all import *


def Xmasscan(dst_ip, dst_port, timeout=10):
    pkts = sr1(IP(dst=dst_ip)/TCP(dport=dst_port, flags="FPU"), timeout=10)
    if (pkts is None):
        print("Open|Filtered")
    elif(pkts.haslayer(TCP)):
        if(pkts.getlayer(TCP).flags == 0x14):
            print("Closed")
    elif(pkts.haslayer(ICMP)):
        if(int(pkts.getlayer(ICMP).type) == 3 and int(pkts.getlayer(ICMP).code) in [1, 2, 3, 9, 10, 13]):
            print("Filtered")


Xmasscan('172.16.111.119', 80)
```

**nmap**

```bash
nmap -sX -p 80 172.16.111.119
```

- 端口关闭：
  - code
  
    ![xmas-close-code](img/xmas-close-code.jpg)
  
  - nmap
  
    ![xmas-close-nmap](img/xmas-close-nmap.jpg)
  
- 端口开启：
  - code
  
    ![xmas-open-code](img/xmas-open-code.jpg)
  
  - nmap
  
    ![xmas-close-nmap](img/xmas-close-nmap.jpg)
  
- 端口过滤：
  - code
  
    ![xmas-f-code](img/xmas-f-code.jpg)
  
  - nmap
  
    ![xmas-f-nmap](img/xmas-f-nmap.jpg)



### TCP FIN scan

> 仅发送FIN包，FIN数据包能够通过只监测SYN包的包过滤器，隐蔽性较SYN扫描更⾼，此扫描与Xmas扫描也较为相似，只是发送的包未FIN包，同理，收到RST包说明端口处于关闭状态；反之说明为开启/过滤状态。

**TCP-FIN-scan code**

```python
from scapy.all import *


def finscan(dst_ip, dst_port, timeout=10):
    pkts = sr1(IP(dst=dst_ip)/TCP(dport=dst_port, flags="F"), timeout=10)
    if (pkts is None):
        print("Open|Filtered")
    elif(pkts.haslayer(TCP)):
        if(pkts.getlayer(TCP).flags == 0x14):
            print("Closed")
    elif(pkts.haslayer(ICMP)):
        if(int(pkts.getlayer(ICMP).type) == 3 and int(pkts.getlayer(ICMP).code) in [1, 2, 3, 9, 10, 13]):
            print("Filtered")


finscan('172.16.111.119', 80)
```

**nmap**

```bash
nmap -sF -p 80 172.16.111.119
```

- 端口关闭：
  - code
  
    ![fin-close-code](img/fin-close-code.jpg)
  
  - nmap
  
    ![fin-close-nmap](img/fin-close-nmap.jpg)
  
- 端口开启：
  - code
  
    ![fin-open-code](img/fin-open-code.jpg)
  
  - nmap
  
    ![fin-open-nmap](img/fin-open-nmap.jpg)
  
- 端口过滤：
  - code
  
    ![fin-code](img/fin-code.jpg)
  
  - nmap
  
    ![fin-nmap](img/fin-nmap.jpg)



### TCP NULL scan

> 发送的包中关闭所有TCP报⽂头标记，实验结果预期还是同理：收到RST包说明端口为关闭状态，未收到包即为开启/过滤状态.

**TCP-NULL-scan code**

```python
#! /usr/bin/python
from scapy.all import *


def nullscan(dst_ip, dst_port, timeout=10):
    pkts = sr1(IP(dst=dst_ip)/TCP(dport=dst_port, flags=""), timeout=10)
    if (pkts is None):
        print("Open|Filtered")
    elif(pkts.haslayer(TCP)):
        if(pkts.getlayer(TCP).flags == 0x14):
            print("Closed")
    elif(pkts.haslayer(ICMP)):
        if(int(pkts.getlayer(ICMP).type) == 3 and int(pkts.getlayer(ICMP).code) in [1, 2, 3, 9, 10, 13]):
            print("Filtered")


nullscan('172.16.111.119', 80)
```

**nmap**

```bash
nmap -sN -p 80 172.16.111.119
```

- 端口关闭：
  - code
  
    ![null-close-code](img/null-close-code.jpg)
  
  - nmap
  
    ![null-close-nmap](img/null-close-nmap.jpg)
  
- 端口开启：
  - code
  
    ![null-code-open](img/null-code-open.jpg)
  
  - nmap
  
    ![null-open-nmap](img/null-open-nmap.jpg)
  
- 端口过滤：
  - code
  
    ![null-code](img/null-code.jpg)
  
  - nmap
  
    ![null-nmap](img/null-nmap.jpg)



### UDP scan

> 一种开放式扫描，通过发送UDP包进行扫描。当收到UDP回复时，该端口为开启状态；否则即为关闭/过滤状态.

**UDP-scan code**

```python
from scapy.all import *
def udpscan(dst_ip, dst_port, dst_timeout=10):
    resp = sr1(IP(dst=dst_ip)/UDP(dport=dst_port), timeout=dst_timeout)
    if (resp is None):
        print("Open|Filtered")
    elif (resp.haslayer(UDP)):
        print("Open")
    elif(resp.haslayer(ICMP)):
        if(int(resp.getlayer(ICMP).type) == 3 and int(resp.getlayer(ICMP).code) == 3):
            print("Closed")
        elif(int(resp.getlayer(ICMP).type) == 3 and int(resp.getlayer(ICMP).code) in [1, 2, 9, 10, 13]):
            print("Filtered")
        elif(resp.haslayer(IP) and resp.getlayer(IP).proto == IP_PROTOS.udp):
            print("Open")
udpscan('172.16.111.119', 53)
```

##### nmap

```bash
nmap -sU -p 53 172.16.111.119
```

- 端口关闭：
  - code
  
    ![udp-closed-code](img/udp-closed-code.jpg)
  
  - nmap
  
    ![udp-close-nmap](img/udp-close-nmap.jpg)
  
- 端口开启：
  - code
  - nmap
  
- 端口过滤：
  - code
  - nmap

## 课后思考题

- 扫描方式与端口状态的对应关系：

  | 扫描方式/端口状态             | 开放                            | 关闭            | 过滤            |
  | ----------------------------- | ------------------------------- | --------------- | --------------- |
  | TCP connect / TCP stealth     | 完整的三次握手，能抓到ACK&RST包 | 只收到一个RST包 | 收不到任何TCP包 |
  | TCP Xmas / TCP FIN / TCP NULL | 收不到TCP回复包                 | 收到一个RST包   | 收不到TCP回复包 |
  | UDP                           | 收到UDP回复包                   | 收不到UDP回复包 | 收不到UDP回复包 |

- 提供每一次扫描测试的抓包结果并分析与课本中的扫描方法原理是否相符？如果不同，试分析原因； 

  结果是完全符合的。

## 参考资料

- [alpine 常用命令](https://blog.csdn.net/liumiaocn/article/details/87603628)
- [自己动手编程实现并讲解TCP connect scan/TCP stealth scan/TCP XMAS scan/UDP scan](https://www.likecs.com/show-203803956.html)

