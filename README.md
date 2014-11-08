小米路由器VPN配置
===

基于域名黑名单的翻墙方式，好处是：

* 配置简单可控，只有需要的时候才走VPN
* 对P2P友好，同时也能节约大量的VPN流量

当然该方法也适用于[OpenWrt](https://openwrt.org/)的路由器

## 设置

首先对指定域名下的所有请求进行标记，以区分需要进行翻墙操作的请求，然后让所有被标记的请求使用VPN专用的路由表进行路由

需要用到[ipset](http://ipset.netfilter.org/ipset.man.html)、[dnsmasq](http://www.thekelleys.org.uk/dnsmasq/docs/dnsmasq-man.html) [ip](http://linux.die.net/man/8/ip)及[iptables](http://ipset.netfilter.org/iptables.man.html)等工具

### Step 1

建立需要翻墙的IP集合，名为`vpn`

```sh
$ ipset create vpn hash:ip
```

为了以后能在开机时自动建立，将该代码作为启动脚本加入`/etc/rc.local`文件中

### Step 2

自动解析需要的翻墙的域名加入IP集合

在`/etc/dnsmasq.d/`目录下建立`vpn.conf`文件，并在其中加入一下代码：

```sh
# 对此域名的解析使用Google的公共DNS服务器(防止DNS污染)
server=/google.com/8.8.8.8
# 将此域名对应的所有IP地址加入之前建立的 vpn ip集合中
ipset=/google.com/vpn
```

对于其它需要翻墙的域名都如法炮制，随意添加。

**注意** 一般网站都会使用多个顶级域名来提供服务，如果网站能打开但是页面图片什么的看不到就是还需要补充些域名

### Step 3

标记vpn集合中的网络请求，方便后续的路由操作

```sh
# 符合VPN地址集合的请求都以`8`进行标记
iptables -t mangle -A fwmark -m set --match-set vpn dst -j MARK --set-mark 7
```

为了以后能在开机时自动生效，将该代码作为启动脚本加入`/etc/rc.local`文件中

### Step 4

建立VPN专用的路由表

在`/etc/iproute2/rt_tables`文件中添加一个行记录

```sh
# 增加一个序号为200 名为gfw的路由表
200 gfw
```

新版的Rom(0.8.74)已经默认有一个叫`vpn`的路由表了，所以得取个另外的名字，囧...

### Step 5

在建立VPN连接时让所有已被标记为需要翻墙的请求使用专用的VPN路由表

拷贝仓库中的[vpnup](vpnup)文件到`/etc/ppp/ip-up.d/`目录下，并执行`chmod a+x vpnup`使`vpnup`文件具有可执行权限

然后在VPN断开时不再使用专用的VPN路由表

拷贝仓库中的[vpndown](vpndown)文件到`/etc/ppp/ip-down.d/`目录下，并执行`chmod a+x vpndown`使`vpndown`文件具有可执行权限

### Step 6

如果`MiWiFi`的Rom版本在`0.7.6`以下，可以跳过此步，直接前往[Step 7](#step7)

`0.7.6`以上的版本改变了VPN建立时相关脚本的逻辑，需要额外进行两处设置

* 拷贝仓库中的[70-vpn](70-vpn)文件至`/etc/hotplug.d/iface/`目录覆盖原文件，并执行`chmod a+x 70-vpn`命令
* 执行`chmod a-x /etc/ppp/ppp.d/vpn-up`

### Step 7

打完收工，设置`PPTP/L2TP`为自动开机链接，最后重启路由器 ~
