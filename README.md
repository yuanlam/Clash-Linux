# Clash-Linux-折腾笔记
#### 在这里记录下自己在LINUX下折腾Clash的基本过程和踩过得坑
#### 本人并没有正规学过LINUX，以下内容全部是在网络上面搜集的教程和TG各大佬的指导，然后整合，反复折腾出来的，亲测可用，如果有写的不对的地方，大佬别笑 

## 系统：Debian 10.3 （其他任何Linux系统都可以，或者更好）
官网下载的ISO镜像，Proxmox新建虚拟机安装，安装过程就不累述了，网上大把教程。新建一个普通用户，以下用本人惯用的yuanlam为例

## 一.修改Debian IP获取方式为静态IP（root用户）
#### 因为Debian新系统IP获取方式默认是DHCP，这里得修改为静态，这样才能用SSH进行后续操作
##### root用户登录，如果没法直接用root用户登录的话，就用yuanlam登录，然后用su - root输入密码切换。
###### 1.打开文件，用nano用vim都可以，看个人习惯
```
nano /etc/network/interfaces       
```
#### 2.参照以下内容根据实际情况修改：
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
#### 3.重启网络
```
/etc/init.d/networking restart
```
#### 4.重启系统
```
reboot
```

## 二.系统调优（root用户，此步骤可做可不做）
#### ~~所有东西都是参照网上修改的，酒精有没有效我也不知道~~
#### 1.升级
```
apt update && apt -y upgrade
```
#### 2.安装必要组件
```
apt -y install net-tools curl vim zip unzip yum supervisor wget nano gnupg gnupg2 gnupg1
apt -y install sudo
sudo apt-get install libcap2-bin
```
#### 3.切换内核，开启BBR

添加源
```
echo 'deb http://deb.xanmod.org releases main' | sudo tee /etc/apt/sources.list.d/xanmod-kernel.list && wget -qO - https://dl.xanmod.org/gpg.key | sudo apt-key add -
```
安装
```
apt -y update && apt -y install linux-xanmod
```
在systemd（> = 217）的系统中使用CAKE队列规则
```
echo 'net.core.default_qdisc = cake' | tee /etc/sysctl.d/90-override.conf
```
重启
```
reboot
```
查看CAKE是否生效
```
sysctl net.core.default_qdisc
```
查看可用的拥塞控制算法
```
sysctl net.ipv4.tcp_available_congestion_control
```
查看当前的拥塞控制算法，应该是回显BBR，也就是说BBR是直接开启的，不需要去自己改sysctl.conf
```
sysctl net.ipv4.tcp_congestion_control
```
#### 4.优化系统参数
打开配置文件
```
sudo nano /etc/sysctl.conf
```
最下填入以下参数
```
#开启流量转发
**此处是重点，后面那些可以不改，这里必须改，开启了转发，后面的动作才有意义**
net.ipv4.ip_forward=1

#增大打开文件数限制
fs.file-max = 999999

#增大所有类型数据包的缓冲区大小（通用设置，其中default值会被下方具体类型包的设置覆盖）
#最大缓冲区大小为64M，初始大小64K。下同
#此大小适用于一般的使用场景。如果场景偏向于传输大数据包，则可以按倍数扩大该值，去匹配单个包大小
net.core.rmem_max = 67108864
net.core.wmem_max = 67108864
net.core.rmem_default = 6291456
net.core.wmem_default = 6291456
net.core.netdev_max_backlog = 65535
net.core.somaxconn = 262114

#增大TCP数据包的缓冲区大小，并优化连接保持
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_keepalive_time = 1200
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.tcp_max_tw_buckets = 5000
net.ipv4.tcp_mem = 8192 131072 67108864
net.ipv4.tcp_rmem = 10240 87380 12582912
net.ipv4.tcp_wmem = 10240 87380 12582912
net.ipv4.tcp_mtu_probing = 1
net.ipv4.tcp_notsent_lowat = 16384
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_orphans= 262114
net.ipv4.tcp_fastopen = 3

net.ipv4.ip_local_port_range = 1024 65000

#增大UDP数据包的缓冲区大小
net.ipv4.udp_mem = 8192 131072 67108864
net.ipv4.udp_rmem_min = 4096
net.ipv4.udp_wmem_min = 4096
```
保存退出，输入命令使修改生效
```
sysctl --system
```
#### 5.永久关闭ipv6
依次输入命令
```
echo " ">>/etc/sysctl.conf
echo "# made for disabled IPv6 in $(date +%F)">>/etc/sysctl.conf
echo 'net.ipv6.conf.all.disable_ipv6 = 1'>>/etc/sysctl.conf
echo 'net.ipv6.conf.default.disable_ipv6 = 1'>>/etc/sysctl.conf
echo 'net.ipv6.conf.lo.disable_ipv6 = 1'>>/etc/sysctl.conf
tail -5 /etc/sysctl.conf
sysctl -p
netstat -anptl
```
修改ssh配置，只监听IPv4地址
打开文档
```
nano /etc/ssh/sshd_config
```
文件末尾添加以下参数，也可直接文档取消注释
```
ListenAddress 0.0.0.0
AddressFamily inet
```
重启服务
```
service sshd restart
netstat -anptl
```
#### 6.取消53端口被占用的情况
*其实这里可以不做，只要Clash的DNS监听端口不适用53就可以了*
进入文件
```
sudo nano /etc/systemd/resolved.conf
```
取消注释，修改yes为no
```
DNSStubListener=no
```
保存退出
#### 7.创建非root用户，用于安装运行Clash
创建新用户
*此处如果是用IOS镜像安装，可以跳过*
```
sudo adduser yuanlam      #根据提示输入密码，其他保持默认
```
将新用户添加到 sudo 组
```
usermod -aG sudo yuanlam
```
重启系统
```
reboot
```
## 三.安装Clash（使用yuanlam用户登录）
#### 1.下载最新版本
```
wget https://github.com/Dreamacro/clash/releases/download/v0.18.0/clash-linux-amd64-v0.18.0.gz
```
#### 2.解压
```
gzip -d clash-linux-amd64-v0.18.0.gz
```
#### 3.移动至usr/bin/clash并重命名为clash
```
sudo mv clash-linux-amd64-v0.18.0 /usr/bin/clash      #普通用户首次使用sudo需要输入密码，输入正确没有任何反馈
```
#### 4.赋予clash运行权限
```
sudo chmod +x /usr/bin/clash
```
#### 5.检查是否安装成功
```
clash -v      #如返回（Clash v0.18.0 linux amd64 Fri Feb 21 12:42:08 UTC 2020）即成功
```
#### 6.为 clash 添加绑定低位端口的权限，这样运行clash的时候无需root权限
```
sudo setcap cap_net_bind_service=+ep /usr/bin/clash
```
## 四.创建配置文件及安装控制面板（使用yuanlam用户登录）
#### 1.创建配置文件目录
```
mkdir -p ~/.config/clash
```
#### 2.进入目录
```
cd ~/.config/clash
```
#### 3.创建配置文件
```
touch config.yaml
```
#### 4.手动编辑很麻烦，可用winscp上传到/home/用户名/.config/clash (需打开隐藏文件），然后再用命令确认
```
nano config.yaml
```
#### 5.下载前端代码压缩包
```
wget https://github.com/haishanh/yacd/archive/gh-pages.zip
```
#### 6.解压
```
unzip gh-pages.zip
```
#### 7.把目录名改成 dashboard
```
mv yacd-gh-pages/ dashboard/
```
