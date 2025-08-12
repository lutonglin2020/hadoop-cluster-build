# 在虚拟机上搭建 Hadoop 完全分布式集群

> 日期：2025-08-11  
> 环境：VMware + CentOS 7 + Hadoop 3.3.6  
> 节点：1×NameNode + 2×DataNode

## 目录
- [前置准备](#前置准备)
- [步骤总览](#步骤总览)
- [详细步骤](#详细步骤)
- [踩坑记录](#踩坑记录)
- [参考链接](#参考链接)

---

## 前置准备
- [x] 已安装 VMware Workstation 17.5.2(最新版)  
- [x] 下载 CentOS-Stream-9-20250728.1-x86_64-dvd1.iso(清华源最新版https://mirrors.tuna.tsinghua.edu.cn/apache/hadoop/common/h)
- [x] 下载 `hadoop-3.3.6.tar.gz` 与 `jdk-8u381-linux-x64.tar.gz`

---

## 步骤总览
| 序号 | 关键操作            | 预计耗时 |
|------|---------------------|----------|
| 1 | 创建 1 台虚拟机     | 30 min   |
| 2  | 配置 hostsIP 映射   | 2 min   |
| 3 | 克隆虚拟机          | 2 min|
|4|配置主机名|1min|
|5|配置网络参数与重新生成uuid|10min|
|6|配置SSH远程登陆|10min|
|7|配置虚拟机之间SSH免密登录|5min|
| 8| 安装 JDK & Hadoop   | 10 min   |
| 9  | 修改 5 大配置文件   | 20 min   |
| 10 | 格式化 & 启动集群   | 5 min    |

---

## 详细步骤

### 1. 创建一台虚拟机
要点是稍后安装操作系统，客户机系统选centos8，处理器数量1，内核2，内存6GB，磁盘大小30GB。


挂载镜像后进行安装系统，首先打开以太网，更改主机名，确定时间日期在上海，磁盘分区设置自动，软件选择勾选最小化就行，配置root密码（简单密码确认两遍），等待安装完成。
### 2. 配置hosts文件&克隆虚拟机
vi /etc/hosts 

追加
```vi
192.168.121.160 hadoop1
192.168.121.161 hadoop2
192.168.121.162 hadoop3
```

完整克隆两台虚拟机当节点，并命名为hadoop2、3。
```base
hostnamectl set-hostname hadoop2
hostnamectl set-hostname hadoop3
```
### 3. 配置网络参数与重新生成uuid
打开虚拟网络编辑器，选择VMent8，点击更改设置，重新选择VMent8，子网地址改为

192.168.121.0

进入系统，编辑网络配置文件（三个虚拟机都要干）
```bash
vi /etc/NetworkManagersystem-connections/ens33.nmconnection
```
修改ipv4的设置内容

method参数指定为manual，表示使用静态ip

添加
```vi
address1=192.168.121.160/24，192.168.121.2
dns=114.114.114.114
```
其他的虚拟机ip地址不同，根据hosts文件修改就行。

重新生成uuid的命令以及使生效
```bash
sed -i '/uuid=/c\uuid='`uuidgen`'' /etc/NetworkManagersystem-connections/ens33.nmconnection
nmcli c reload
nmcli c up ens33
ip addr
```
查看ip地址是否已改变，并ping百度测试网络是否联通。
### 4. 配置SSH远程登陆
为了各种原因的方便，选择xshell进行远程登陆比较方便。
```vi
sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
systemctl restart sshd
```
并关闭防火墙
```bash
systemctl stop firewalld
systemctl disable firewalld
```
打开xshell，最新的版本已经免费，无需登录，直接选择后来，进入主界面，起名为hadoop1，hostname输入IP地址，选择接受并保存密钥，输入root和密码，建立链接成功，如不成功，多试几次。

连接成功会进入终端，自此开始可以自由的复制黏贴和上传文件了。
### 5. 配置虚拟机SSH免密登录
```vi
ssh-keygen -r rsa
```
根据提示操作
```vi
ssh-copy-id hadoop1
ssh-copy-id hadoop2
ssh-copy-id hadoop3
```
根据要求输入，并输入密码，执行ssh hadoop2测试是否成功。退出命令为exit。
### 6. 安装jdk
为了统一资源，创建三个文件夹
```vi
mkdir -p /export/data/
mkdir -p /export/servers/
mkdir -p /export/software/
```
