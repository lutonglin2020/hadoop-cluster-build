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
- [x] 下载 CentOS-Stream-9-20250728.1-x86_64-dvd1.iso(清华源最新版)
- [x] 下载 `hadoop-3.3.6.tar.gz` 与 `jdk-8u381-linux-x64.tar.gz`

---

## 步骤总览
| 序号 | 关键操作            | 预计耗时 |
|------|---------------------|----------|
| 1    | 创建 1 台虚拟机     | 30 min   |
| 2    | 配置 hostsIP 映射   | 2 min   |
| 3    | 克隆虚拟机          | 2 min|
|4|配置主机名|1min|
|5|配置网络参数与重新生成uuid|10min|
|6|配置SSH远程登陆|10min|
|7|配置虚拟机之间SSH免密登录|5min|
| 8| 安装 JDK & Hadoop   | 10 min   |
| 9    | 修改 5 大配置文件   | 20 min   |
| 10    | 格式化 & 启动集群   | 5 min    |

---

## 详细步骤

### 1. 虚拟机网络配置
```bash
# /etc/sysconfig/network-scripts/ifcfg-ens33
BOOTPROTO=static
IPADDR=192.168.121.221
NETMASK=255.255.255.0
GATEWAY=192.168.121.2
DNS1=192.168.121.2
