# 遇到的问题

## iptables被刷了数十万条重复规则

某天早上查看grafana node exporter整合数据，发现有一台服务器内存占用90%，登入后htop发现一个iptables fillter进程内存占用非常高。

输入

```bash
iptables -L -n --line-numbers
```

发现命令卡住，等了大概2分钟开始刷屏，然后尝试把结果重定向到一个文件里，等待了大概3分钟，得到了一个有35w规则的文件，其中大部分都是K3s创建的FORWARD规则。

#### 排查原因

发现iptables里Libvirtd添加的规则也很多，Libvirtd是虚拟机工具，也需要iptables，由于查看K3s日志也看不出是什么原因。于是先试试停止Libvirtd。

#### 解决

首先清理无用的iptables，让内存占用降下来

1. 驱逐该节点``` kubectl drain xxx --ignore-daemonsets --delete-emptydir-data ```，由于机器上安装了node-exporter，需要添加参数忽略掉，显示pod都被evicted。
2. 然后清掉该节点上的K3s添加的iptables规则和停掉所有容器，直接执行```/usr/bin/k3s-killall.sh```脚本
3. 停止K3s-agent```systemctl stop k3s-agent```

由于使用iptables FORWARD的只有机器上的K3s和Docker。直接``` iptables -F FORWARD```把规则清除掉。

重启K3s-agent，把节点加回来```kubectl uncordon xxx```

之后就没有这个问题了，可能是Libvirtd添加iptables和K3s产生了意外冲突？

## nfs写入的文件仅root可访问

搭建好nfs和部署了分配器后，发现写入的卷没法用其他用户直接访问。

#### 解决

配置nfs，把用户改为nobody即可。
