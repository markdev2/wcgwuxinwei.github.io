---
layout:     post
title:      "MITM Proxy 环境搭建"
subtitle:   "安全"
date:       2015-08-01 13:00::00
author:     "MarkWoo"
header-img: "img/home-bg.jpg"
---

# MITM_Proxy环境搭建

---

# 环境要求
系统环境要求：
>* Ubuntu 14.04 x64，CentOS 7 x64以上版本系统(建议使用xubuntu 14.04 x64，稳定硬件要求低)
>* Python 2.7以上运行环境（MITM用Python写的）
>* Pip 7.1版本以上,这个用于安装**MITM Proxy**

官方安装指南：**http://mitmproxy.org/doc/install.html**

---
# 下面以xubunut x64为例进行环境搭建
- 首先安装一个纯净版的**xubuntu 14.04 x64**系统,手动安装分区方法见下面链接: http://blog.chinaunix.net/uid-7547035-id-60111.html
- 设置好升级**xubuntu**升级服务器选择，建议选择aliyun节点或者香港中文大学节点，速度快，稳定
- 设置完毕后会提示授权以及是否立即进行更新，选择是
## 然后打开终端模拟器(terminial)，准备开始更新系统列表和开始配置网卡,运行命令如下:

```bash
sudo apt-get update #更新远程软件仓库列表
sudo apt-get install vim git openssh-server openssh-client #安装工具vim,git,openssh-server,openssh-client
git clone https://github.com/wuxinwei/MyConfig.git #从github上clone下来我的配置信息，里面有vimrc,tmux,zsh的配置文件，我已经全部配好了，可以覆盖到/home/用户名/就可以
cp .../MyConfig/_vimrc ~/.vimrc #使用vim配置文件
sudo /etc/init.d/ssh start #启动ssh服务，ssh-server配置文件位于/ etc/ssh/sshd_config，在这里可以定义SSH的服务端口，默认端口是22，你可以自己定义成其他端口号，如222
sudo apt-get install bridge-utils #安装桥工具bridge-utils
brctl addbr br0 #添加一个网桥
brctl addif br0 eth0 #将eth0加到网桥中去
brctl addif br0 eth1 #将eth1加到网桥中去
```

---
## 开启IP转发功能以及其他配置`sudo vim /etc/sysctl.conf`:

```bash
# 将全部内容改为如下:
#
# /etc/sysctl.conf - Configuration file for setting system variables
# See /etc/sysctl.d/ for additional system variables.
# See sysctl.conf (5) for information.
#

#kernel.domainname = example.com

# Uncomment the following to stop low-level messages on console
#kernel.printk = 3 4 1 3

##############################################################3
# Functions previously found in netbase
#

# Uncomment the next two lines to enable Spoof protection (reverse-path filter)
# Turn on Source Address Verification in all interfaces to
# prevent some spoofing attacks
#net.ipv4.conf.default.rp_filter=1
#net.ipv4.conf.all.rp_filter=1

# Uncomment the next line to enable TCP/IP SYN cookies
# See http://lwn.net/Articles/277146/
# Note: This may impact IPv6 TCP sessions too
#net.ipv4.tcp_syncookies=1

# Uncomment the next line to enable packet forwarding for IPv4
net.ipv4.ip_forward=1

# Uncomment the next line to enable packet forwarding for IPv6
#  Enabling this option disables Stateless Address Autoconfiguration
#  based on Router Advertisements for this host
net.ipv6.conf.all.forwarding=1


###################################################################
# Additional settings - these settings can improve the network
# security of the host and prevent against some network attacks
# including spoofing attacks and man in the middle attacks through
# redirection. Some network environments, however, require that these
# settings are disabled so review and enable them as needed.
#
# Do not accept ICMP redirects (prevent MITM attacks)
net.ipv4.conf.all.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
# _or_
# Accept ICMP redirects only for gateways listed in our default
# gateway list (enabled by default)
# net.ipv4.conf.all.secure_redirects = 1
net.ipv4.conf.all.secure_redirects = 0
#
# Do not send ICMP redirects (we are not a router)
#net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.send_redirects = 0
#
# Do not accept IP source route packets (we are not a router)
#net.ipv4.conf.all.accept_source_route = 0
#net.ipv6.conf.all.accept_source_route = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0
#
# Log Martian Packets
#net.ipv4.conf.all.log_martians = 1
#
```

---
## 配置网桥`sudo vim /etc/network/interfaces//以管理员权限打开网络配置文件`:

```bash
# 把全部文件内容改为如下：
# interfaces(5) file used by ifup(8) and ifdown(8)
auto lo
iface lo inet loopback

auto eth0 #网卡eth0
iface eth0 inet manual #设置eth0网卡为手动配置，这样不会自动获取ip配置等

auto eth1 #网卡eth1
iface eth1 inet manual #设置eth0网卡为手动配置，这样不会自动获取ip配置等

auto br0 #网桥0
iface br0 inet static #设置网桥br0为静态，这样不会变化ip
address 192.168.4.144 #这里设置网桥IP
netmask 255.255.255.0 #设置网桥子网掩码
broadcast 192.168.4.255 #设置网桥广播地址
gateway 192.168.4.2 #这是网桥网关
dns-nameservers 202.96.128.86 #设置网桥DNS
bridge_ports eth0 eth1 //将eth0 和eth1网卡添加如网桥中去
bridge_stp off
brideg_hello 2
bridge_fd 9
bridge_maxwait 0
```
---
## 设置防火墙(注意，这里的`iptables`设置在重启系统后会失效,所以如果重启过机器这里需要重新设置)：

```bash
echo 0 | sudo tee /proc/sys/net/ipv4/conf/*/send_redirects
*/
service iptables start #开启iptables过滤服务
service iptables save
iptables -t nat -A PREROUTING -i br0 -p tcp --dport 80 -j REDIRECT --to-port 8080 #将80端口转发给8080
iptables -t nat -A PREROUTING -i br0 -p tcp --dport 443 -j REDIRECT --to-port 8080 #将443端口数据转发给8080
service iptables save
```

官方透明代理设置教程: 
http://mitmproxy.org/doc/transparent/linux.html(物理机设置)
http://mitmproxy.org/doc/tutorials/transparent-dhcp.html(虚拟机设置)
经过以上配置，实现了建立网桥，打开路由转发功能（用于MITM的透明代理）

---
## 安装**MITM Proxy**运行环境以及**MITM Proxy**

```bash
sudo apt-get install python-pip python-dev libffi-dev libssl-dev libxml2-dev libxslt1-dev #安装必要的运行环境
sudo pip install mitmproxy #安装mitmproxy,安装成功后会在生成两个工具/usr/local/bin/mitmproxy与/usr/local/bin/mitmdump
```

---
## 到这里为mitmproxy的环境安装完毕，接下来进行目标机方面的配置

>* CA证书的安装
　　要捕获https证书，就得解决证书认证的问题，因此需要在通信发生的客户端安装证书，并且设置为受信任的根证书颁布机构。下面介绍6种客户端的安装方法。
　　当我们初次运行mitmproxy或mitmdump时，
　　会在当前目录下生成 ~/.mitmproxy文件夹，其中该文件下包含4个文件，这就是我们要的证书了。
　　mitmproxy-ca.pem 私钥
　　mitmproxy-ca-cert.pem 非windows平台使用
　　mitmproxy-ca-cert.p12 windows上使用
　　mitmproxy-ca-cert.cer 与mitmproxy-ca-cert.pem相同，android上使用
　　1. Firefox上安装
　　preferences-Advanced-Encryption-View Certificates-Import (mitmproxy-ca-cert.pem)-trust this CA to identify web sites
　　2. chrome上安装
　　设置-高级设置-HTTPS/SSL-管理证书-受信任的根证书颁发机构-导入mitmproxy-ca-cert.pem
　　2. osx上安装
　　双击mitmproxy-ca-cert.pem - always trust
　　3.windows7上安装
　　双击mitmproxy-ca-cert.p12-next-next-将所有的证书放入下列存储-受信任的根证书发布机构
　　4.iOS上安装
　　将mitmproxy-ca-cert.pem发送到iphone邮箱里，通过浏览器访问/邮件附件
　　6.Android上安装
　　将mitmproxy-ca-cert.cer 放到sdcard根目录下
　　选择设置-安全和隐私-从存储设备安装证书

---
一些额外的资料：
>* 官方教程: http://mitmproxy.org/doc/index.html
>* [win7、linux安装使用pip、mitmproxy][1]
>* [推荐给开发人员的6个实用命令行工具][2]
>* [使用mitmproxy进行Android的http抓包][3]
>* [mitmproxy 入门案例 -『抱抱』24小时销毁的真相(iOS端)][4]
>* [mitmproxy实践教程之调试 Android 上 HTTP流量][5]

  [1]: http://www.cnblogs.com/ShepherdIsland/p/4239052.html
  [2]: http://blog.jobbole.com/30251/
  [3]: http://hello1010.com/mitmproxy-android/
  [4]: http://liujin.me/blog/2015/05/27/mitmproxy-for-beginner/
  [5]: https://greenrobot.me/devpost/how-to-debug-android-http-get-started/
