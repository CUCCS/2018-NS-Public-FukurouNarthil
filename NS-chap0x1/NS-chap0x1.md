# Chap0x01 基于Virtualbox的网络攻防基础 实验报告

## 实验要求

- 节点：靶机、网关、攻击者主机
- 连通性
  - 靶机可以直接访问攻击者主机
  - 攻击者主机无法直接访问靶机
  - 网关可以直接访问攻击者主机和靶机
  - 靶机的所有对外上下行流量必须经过网关
  - 所有节点均可以访问互联网
- 其他要求
  - 所有节点制作成基础镜像（多重夹杂的虚拟硬盘）

## 实验过程

### 环境搭建

#### 1. 靶机（victim/Windows10）

- 安装增强功能
  - 左上角菜单 -> 安装增强功能
  - 文件资源管理器 -> 安装增强功能的CD驱动器 -> VBoxWindowsAdditions.exe
- 关闭防火墙
  - 控制面板 -> 系统和安全 -> Windows防火墙
- 网卡配置
  - 只有一块Internal Network网卡
  - ![](images\victim-netcard.png)
- IP地址、默认网关、DNS服务器配置
  - 右键点击文件资源管理器右侧“网络” ->  属性
    - ![](images\victim-ip.png)
    - ![](images\victim-ip1.png)
    - ![](images\victim-ip2.png)
    - ![](images\victim-ip3.png)
- cmd输入ipconfig
  - ![](images\victim-ipconfig.png)



#### 2. 网关（gateway/ubuntu）

- 以root身份执行`apt update && apt upgrade -y && apt dist-upgrade -y` 
- 网卡配置
  - Internal Network网卡（enp0s8）
    - ![](images\gateway-netcard1.png)
  - NAT Network网卡（enp0s3）
    - ![](images\gateway-netcard2.png)
- IP地址、默认网关、DNS服务器配置
  - 右上角齿轮 -> 系统设置 -> 网络
    - ![](images\gateway-ip.png)
    - ![](images\gateway-ip1.png)
- 终端输入ifconfig
  - ![](images\gateway-ifconfig.png)



#### 3. 攻击者（attacker/Kali）

- 安装增强功能
  - `apt install virtualbox-guest-x11` 
- 以root身份执行`apt update && apt upgrade -y && apt dist-upgrade -y` 
- 网卡配置
  - 只有一块NAT Network网卡
    - ![](images\attacker-netcard.png)
- IP地址、默认网关、DNS服务器配置
  - 右键桌面 -> 设置 -> 网络
  - 之后的步骤和gateway的设置相差无几，此处略
- 终端输入ifconfig
  - ![](images\attacker-ifconfig.png)



#### IP地址配置说明

1. 靶机的IP地址和网关内网网卡的IP地址在同一子网中
2. 靶机默认网关的ip地址为网关内网网卡的IP地址
3. 网关内网端口的默认网关设置为NAT网络端口的IP地址
4. 网关的NAT 网络端口的IP地址和攻击者主机的IP地址在同一个网段中

### 配置转发

- 在IP地址配置完成之后，发现靶机可以ping通网关的外网端口，但是无法ping通攻击者主机，此时需要在网关添加端口转发的配置
  - 首先打开端口转发功能，使用`sysctl net.ipv4.ip_forward=1`命令
  - 添加转发规则，使用`iptables -t nat -A POSTROUTING -s 192.168.56.0/24 -o enp0s3 -j MASQUERADE`命令
  - 输入`iptables-save`保存设置
  - ![](images\gtw-iptables.png)
  - 设置完成之后，靶机可以ping通攻击者，而攻击者无法ping通靶机，因为网关无法将来自攻击者的包转发给靶机

### 连通性测试

- 靶机访问攻击者主机
  - ![](images\vtm-atk.png)
- 攻击者主机无法直接访问靶机
  - ![](images\atk-vtm.png)
- 网关可以直接访问攻击者主机和靶机
  - ![](images\gtw-vtm-atk.png)
- 靶机的所有对外上下行流量必须经过网关
  - ![](images\vtm-gtw-atk-0.png)
  - ![](images\vtm-gtw-atk.png)
- 所有节点均可以访问互联网
  - 靶机
    - ![](images\vtm-internet.png)
  - 网关
    - ![](images\gtw-internet.png)
  - 攻击者
    - ![](images\attacker-internet.png)

### 多重加载

- ![](images\vtm-ma.jpg)
- ![](images\gtw-ma.png)

- ![](images\atk-ma.jpg)

## 实验遇到的问题

1. 网关双网卡无法上网（SOLVED)

   - 参考 

     [askubuntu上的一个提问]: https://askubuntu.com/questions/868942/how-to-configure-2-network-interfaces-with-different-gateways

   - 修改了 /etc/network/interfaces 文件，但是网关重启之后无法连接上网，后来又把这部分删改去掉了，去掉后仍可以上网，猜测不能同时修改该文件和系统设置

2. 虚拟机重启后之前的配置没有生效（SOLVED）

   - 前一天做完实验第二天打开虚拟机发现配置转发的设置没有保存，重新输入一遍之后生效

3. 靶机的DNS服务器使用的是公共服务器（谷歌的8.8.8.8）

4. DNS服务器没有自动和宿主机保持一致

   - 换了一个网络环境之后虚拟机无法上网，报错`name or service not known` ，执行`cat /etc/resolv.conf`发现其中缺少和宿主机相同的DNS服务器，手动添加后可以上网

