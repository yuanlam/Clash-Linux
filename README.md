# Clash-Linux-折腾笔记
#### 在这里记录下自己在LINUX下折腾Clash的基本过程和踩过得坑
#### 以下内容全部是在网络上面搜集的教程，然后整合，反复折腾出来的，亲测可用 

## 系统：Debian 10.3 （其他任何Linux系统都可以，或者更好）
官网下载的ISO镜像，Proxmox新建虚拟机安装，安装过程就不累述了，网上大把教程。新建一个普通用户，以下用本人惯用的yuanlam为例

## 一.修改Debian IP获取方式为静态IP（root用户）
*因为Debian新系统IP获取方式默认是DHCP，这里得修改为静态，这样才能用SSH进行后续操作*
*root用户登录，如果没法直接用root用户登录的话，就用yuanlam登录，然后用su - root输入密码切换。*
1.打开文件，用nano用vim都可以，看个人习惯
```
nano /etc/network/interfaces       
```
2.参照以下内容根据实际情况修改：
```
auto lo
auto eth0                      #eth0为网卡，根据实际情况修改    
iface lo inet loopback
iface eth0 inet static         #static为静态，dhcp为动态
address 10.10.10.97            #本机IP
netmask 255.255.255.0          #子网掩码
gateway 10.10.10.77            #网关（科学上网的IP）
dns-nameservers 198.18.0.1     #DNS，我的科学上网方式是openwrt旁路由用的openclash，用的fake-ip，所以这里填的是198.18.0.1
mtu 1492                       #MTU
mss 1452                       #MSS
```
3.重启网络
```
/etc/init.d/networking restart
```
4.重启系统
```
reboot
```

## 二.系统调优（root用户，此步骤可做可不做）
#### ~~所有东西都是参照网上修改的，酒精有没有效我也不知道~~
1.升级
```
apt update && apt -y upgrade
```
2.安装必要组件
```
apt -y install net-tools curl vim zip unzip yum supervisor wget nano gnupg gnupg2 gnupg1
apt -y install sudo
sudo apt-get install libcap2-bin
```
3.切换内核，开启BBR
