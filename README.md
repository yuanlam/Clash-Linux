# Clash-Linux-折腾笔记
#### 在这里记录下自己在LINUX下折腾Clash的基本过程和踩过得坑
###### 以下内容全部是在网络上面搜集的教程，然后整合，反复折腾出来的，亲测可用 

## 系统：Debian 10.3 （其他任何Linux系统都可以，或者更好）
官网下载的ISO镜像，Proxmox新建虚拟机安装，安装过程就不累述了，网上大把教程。新建一个普通用户，以下用本人惯用的yuanlam为例

## 一.修改Debian IP获取方式为静态IP
*因为Debian新系统IP获取方式默认是DHCP，这里得修改为静态，这样才能用SSH进行后续操作*

root用户登录，如果没法直接用root用户登录的话，就用yuanlam登录，然后用su - root输入密码切换。

nano /etc/network/interfaces        #打开文件

参照以下内容根据实际情况修改：
、、、
auto lo
auto eth0                      #eth0为网卡，根据实际情况修改    
iface lo inet loopback
iface eth0 inet static         #static为静态，dhcp为动态
address 10.10.10.97            #本机IP
netmask 255.255.255.0          #子网掩码
gateway 10.10.10.77            #科学上网的IP，我的科学上网方式是openwrt旁路由用的openclash，
dns-nameservers 198.18.0.1     #DNS
mtu 1492                       #MTU
mss 1452                       #MSS
、、、
