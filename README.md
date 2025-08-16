# 帮助新手在虚拟机上搭建 Hadoop 完全分布式集群

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
安装传输工具
```
yum install lrzsz -y
```
进入software内，使用rz命令上传jdk安装包，如果出现乱码，使用rz -be命令，反复切换使用试试看。

使用tar -zxvf 命令解压jdk安装包到servers下即可。

配置环境变量
```
vi /etc/profile

export JAVA_HOME=/export/servers/jdk1.8.0_152
export PATH=$PATH:$JAVA_HOME/bin
export HADOOP_HOME=/export/servers/wfb-hadoop/hadoop-3.3.6
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin

:wq
source /etc/profile
java -version
```
出现Java的信息即可。

分发jdk目录和环境变量文件
```
scp -r /export/servers/jdkjdk1.8.0_152 root@hadoop2:/export/servers/
scp -r /export/servers/jdkjdk1.8.0_152 root@hadoop3:/export/servers/
scp -r /etc/profile root@hadoop2:/etc/
scp -r /etc/profile root@hadoop3:/etc/
```
### 7. 部署hadoop
先进入software上传hadoop安装包，用tar解压到servers的wfb-hadoop文件夹内，执行hadoop version,显示hadoop版本号。

进入hadoop配置文件目录，修改配置文件内容
```
cd etc/hadoop

vi hadoop-enx.sh
#添加在底部
export JAVA_HOME=/export/servers/jdk1.8.0_152
export HDFS_NAMENODE_USER=root
export HDFS_DATANODE_USER=root
export HDFS_SECONDARYNAMENODE_USER=root
export YARN_RESOURCEMANAGER_USER=root
export YARN_NODEMANAGER_USER=root

vi core-site.xml
#添加在标签内
<property>
<name>fs.defaultFS</name>
<value>hdfs://hadoop1:9000</value>
</property>
<property>
<name>hadoop.tmp.dir</name>
<value>/export/data/hadoop-wfb-3.3.6</value>
</property>
<property>
<name>hadoop.http.staticuser.user</name>
<value>root</value>
</property>
<property>
<name>hadoop.proxyuser.root.groups</name>
<value>*</value>
</property>
<property>
<name>fs.trash.interval</name>
<value>1440</value>
</property>
<property>
<name>hadoop.proxyuser.root.hosts</name>
<value>*</value>
</property>

vi hdfs_site.xml
#添加在标签内
<property>
<name>dfs.replication</name>
<value>2</value>
</property>
<property>
<name>dfs.namenode.secondary.http-address</name>
<value>hadoop2:9868</value>
</property>

vi mapred-site.xml
<property>
<name>mapreduce.framework.name</name>
<value>yarn</value>
</property>
<property>
<name>mapreduce.jobhistory.address</name>
<value>hadoop1:10020</value>
</property>
<property>
<name>mapreduce.jobhistory.webapp.address</name>
<value>hadoop1:19888</value>
</property>
<property>
<name>yarn.app.mapreduce.am.env</name>
<value>HADOOP_MAPRED_HOME=/export/server/wfb-hadoop/hadoop-3.3.6</value>
</property>
<property>
<name>mapreduce.map.env</name>
<value>HADOOP_MAPRED_HOME=/export/server/wfb-hadoop/hadoop-3.3.6</value>
</property>
<property>
<name>mapreduce.reduce.env</name>
<value>HADOOP_MAPRED_HOME=/export/server/wfb-hadoop/hadoop-3.3.6</value>
</property>

vi yarn-site.xml
<property>
<name>yarn.resourcemanager.hostname</name>
<value>hadoop1</value>
</property>
<property>
<name>yarn.nodemanager.aux-services</name>
<value>mapreduce_shuffle</value>
</property>
<property>
<name>yarn.nodemanager.peme-check-enabled</name>
<value>false</value>
</property>
<property>
<name>yarn.nodemanager.veme-check-enabled</name>
<value>false</value>
</property>
<property>
<name>yarn.log-aggregation-enable</name>
<value>true</value>
</property>
<property>
<name>yarn.log.server.url</name>
<value>http://hadoop1:19888/jobhistory/logs</value>
</property>
<property>
<name>yarn.log-aggregation.retain-seconds</name>
<value>604800</value>
</property>

vi workers
#更改内容为
hadoop2
hadoop3

#分发hadoop安装目录
scp -r /export/servers/wfb-hadoop root@hadoop2:/export/servers/
scp -r /export/servers/wfb-hadoop root@hadoop3:/export/servers/
#格式化hdfs
hdfs namenode -format
```
接下来使用jps在各个主机上查看hadoop的守护进程，hadoop1为主节点，应有namenode、resourcemanager两个进程，hadoop2为从节点，应有datanode、secondarynamenode、nodemanager三个进程，hadoop3为从节点，应有datanode、nodemanager。

自此，hadoop集群的基础安装就好了，具体的各个节点的功能请详读配置文件。

没放图片是因为图片上传比较麻烦，但是该有的内容已经写的极为详尽，需要具备基本的Linux操作基础，本文纯手打，如有错漏之处还请见谅，可以查看错误日志报错位置逐行修改，我自己也是这样一点点改动的。

## 踩坑记录
### 1. 环境变量问题
有的时候从ssh访问的非交互式链接就会容易出现环境变量的配置文件出现无法读取的问题，这一点根据ai给出的解决方案和重启得到了解决，如果出现了无法同时启动三台机器中的hadoop进程的情况，请逐步排查原因。

另外，更改了环境变量的配置文件之后请一定要记住source一下。
### 2. 防火墙问题
有时候启动不成功就是防火墙的问题，本文只是简单的搭建了测试环境，并不适合生产环境，于是我的解决方案直接把三台虚拟机的防火墙ban了，减少操作成本。

有需要的可以针对性的开放对应端口。
### 3. rz上传乱码问题
有时候使用-be参数可以解决，有时候不行，只能用原来的rz，多试几次就能成功，不太确定是什么原因。

### 4. 下机时记得先stop hadoop集群
不要随便挂起，容易损坏虚拟机文件，谁知道什么时候就打不开了，而且最新的VMware不知道为什么卡卡的，用了Xshell之后才舒服一些。正好锻炼了我的敲代码能力。

## 参考链接
jdk安装链接

https://blog.csdn.net/gaobing1/article/details/122112871

hadoop下载链接（清华源）

https://mirrors.tuna.tsinghua.edu.cn/apache/hadoop/common/hadoop-3.3.6/

学校用的黑马的教材，根据上面写写改改的，挺好懂的。
