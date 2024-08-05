在 Centos Stream 8 中设置静态IP如下：
## 1. 先查看当前网络使用的网络接口
```bash
ip addr
```
## 2. 修改当前网卡配置信息

```bash
vim /etc/sysconfig/network-scripts/ifcfg-ens160
```
内容如下
```ini
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
NAME=ens160
UUID=4d6ee2f5-6216-4f92-aa8f-c50ec78404a1
DEVICE=ens160
ONBOOT=yes
IPADDR=192.168.157.50
PREFIX=24
GATEWAY=192.168.157.2
DNS1=192.168.157.20
```
`BOOTPROTO` 设置为 `static` 然后根据自己的网段设置对应的网关、IP、子网掩码等。`PREFIX=24` 这就是子网掩码（表示网络号有24位，即：`255.255.255.0`）

## 3. 应用配置
```bash
# 刷新配置
nmcli c reload
# 重新激活网卡
nmcli c up ens160
# 再次查看
ip addr
```
> 注：ssh 会断开，需要重新用新的IP链接