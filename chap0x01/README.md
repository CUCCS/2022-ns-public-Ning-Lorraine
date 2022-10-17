# 基于 VirtualBox 的网络攻防基础环境搭建



## 实验目的

- 掌握 VirtualBox 虚拟机的安装与使用；
- 掌握 VirtualBox 的虚拟网络类型和按需配置；
- 掌握 VirtualBox 的虚拟硬盘多重加载；



## 实验环境

以下是本次实验需要使用的网络节点说明和主要软件举例：

- VirtualBox 虚拟机
- 攻击者主机（Attacker）：Kali-linux-2022.3
- 网关（Gateway, GW）：Debian Buster
- 靶机（Victim）：From Sqli to shell / xp-sp3 / Kali



## 实验过程

1. 虚拟硬盘配置成多重加载，效果如下图所示；

   ![多重加载](img/修改虚拟硬盘类型.jpg)



2.  VirtualBox 虚拟机的安装，结果如图所示：

   ![虚拟机完成配置](img/虚拟机完成配置.jpg)

​	

3. 虚拟机虚拟网络类型和按需配置

   | 虚拟机          | 网络类型         | IP地址            |
   | --------------- | ---------------- | ----------------- |
   | Attacker-kali   | NATNetwork       | 10.0.2.4/24       |
   | Victim-kali-1   | 内部网络 intnet1 | 172.16.111.124/24 |
   | Victim-XP-1     | 内部网络 intnet1 | 172.16.111.131/24 |
   | Victim-debian-2 | 内部网络 intnet2 | 172.16.222.131/24 |
   | Victim-XP-2     | 内部网络 intnet2 | 172.16.222.147/24 |
   | Gateway-debian  | NATNetwork       | 10.0.2.15/24      |
   |                 | Host Only        | 192.168.56.113/24 |
   |                 | 内部网络 intnet1 | 172.16.111.1/24   |
   |                 | 内部网络 intnet2 | 172.16.222.1/24   |

- 网络类型配置

  - NAT 网络类型配置

    在 管理 --> 全局设定 --> 网路 中可以修改配置，如图所示：

    ![NAT网络配置](img/NAT网络配置.jpg)

    

  - Host-only 网络类型配置

    在 管理 --> 主机网络管理器 中修改配置，如图所示：

    ![Host-only网络配置](img/Host-only网络配置.jpg)

    

  - 内部网络配置

    直接在需要修改的对象的设置-->网络中，将内部网络修改为相应的名称即可，注意在同一内部网络中名称相同，配置界面如图所示：

    ![内部网络配置](img/内部网络配置.jpg)

    

- 网络配置结果：

  - 网关 Gateway-debian :

    ![gw-ipaddr](img/gw-ipaddr.jpg)

  - 攻击者 Attacker-kali :

    ![Attacker-kali-ip配置](img/Attacker-kali-ip配置.jpg)

  - 内部网络1靶机 :

    ![内部网络1-靶机xp-ip配置](img/内部网络1-靶机xp-ip配置.jpg)

    ![靶机kali-ip配置](img/靶机kali-ip配置.jpg)

  - 内部网络2靶机：

    ![内部网络2-靶机xp-ip配置](img/内部网络2-靶机xp-ip配置.jpg)

    ![靶机debian-2-ip配置](img/靶机debian-2-ip配置.jpg)

    

- 搭建满足如下[黄Sir的拓扑图](https://c4pr1c3.github.io/cuc-ns/chap0x01/exp.html)所示的虚拟机网络拓扑

  > ![网络拓扑图](img/拓扑图.jpg)



### 完成网络连通性测试

- [x] 靶机可以直接访问攻击者主机

  ![靶机访问攻击者主机](img/靶机访问攻击者主机.png)

  

- [x] 攻击者主机无法直接访问靶机

  ![攻击者无法直接访问靶机](img/攻击者无法直接访问靶机.jpg)

  

- [x] 网关可以直接访问攻击者主机和靶机

  ![网关访问靶机和攻击者1](img/网关访问靶机和攻击者主机1.jpg)

  ![网关访问靶机2](img/网关访问靶机2.jpg)

  

- [x] 靶机的所有对外上下行流量必须经过网关

  ```bash
  # 网关安装tcpdump
  apt intstall tcpdump
  
  # 对对应网卡进行监控
  /usr/sbin/tcpdump -i <网卡名称>
  ```

  ![网关检测内部网络1](img/网关检测内部网络1.jpg)

  ![网关检测内部网络2](img/网关检测内部网络2.jpg)

  关闭网关虚拟机后，各靶机无法访问互联网，结果如下图所示：

  ![关闭网关后靶机无法访问外网](img/关闭网关后靶机无法访问外网.png)

  

- [x] 所有节点均可以访问互联网

  ```bash
  # 访问baidu.com
  ping baidu.com
  ```

  各靶机访问结果如下图所示：

  ![测试靶机访问外网](img/测试靶机访问外网.png)



### 参考材料

- [网络安全-线上课本-第一章实验](https://c4pr1c3.github.io/cuc-ns/chap0x01/exp.html)
- [师哥作业](https://github.com/CUCCS/2020-ns-public-LyuLumos/blob/ch0x01/ch0x01/%E5%9F%BA%E4%BA%8E%20VirtualBox%20%E7%9A%84%E7%BD%91%E7%BB%9C%E6%94%BB%E9%98%B2%E5%9F%BA%E7%A1%80%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA.md)

- [多重加载功能介绍](https://blog.csdn.net/jeanphorn/article/details/45056251)