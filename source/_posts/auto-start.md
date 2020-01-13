---
title: 配置开机自启动脚本
date: 2019-12-23 14:51:09
tags: 
  - 开机自启动
  - 树莓派
categories:
  - linux
---

我的树莓派现在 wifi 能自动连上了，frp 配置也做好了，阿里云服务器上的 frps 已经启动等待着客户端接入了，我的树莓派上的 frpc 也随时可以手动启动，内网穿透已经可以正常进行，但是每次都要开机联网之后找到树莓派的 ip ssh 上去**手动启动** frpc，太蠢了。所以这篇要讲的就是如何做到让树莓派开机网络之后自动启动 frpc，彻底解决我随时随地方便登陆的需求。

linux 开机自启动方案有很多，我采用的是在 **/etc/init.d** 目录下进行一系列操作将启动脚本作为一个服务来管理。

利用 /etc/network/if-up.d 网卡启动执行脚本的方式也尝试过，但因为网卡不止一张，所以脚本会被重复执行多次，是有点问题的，这里就不多说了。

<!--more-->

## /etc/init.d

1. 需要开机自动执行的脚本开头必须加上以下内容：

   ```bash
   #!/bin/sh
   # Start/stop the <脚本/服务名称> daemon.
   #
   ### BEGIN INIT INFO
   # Provides:          <脚本/服务名称>
   # Required-Start:    $all
   # Required-Stop:
   # Default-Start:     2 3 4 5
   # Default-Stop:
   # Short-Description: <简短介绍>
   ### END INIT INFO
   ```

2. 将脚本 xxx 放到 /etc/init.d 目录下

3. 执行命令：`update-rc.d /etc/init.d/xxx defaults`，你会看到 /run/systemd/generator.late 目录下会多出一个 xxx.service 文件，说明你的脚本已经变成了一个系统服务

4. 执行命令：`update-rc.d /etc/init.d/xxx enable`，开启服务

5. 使用 `sudo service xxx status` 可以查看该服务的状态

 {% asset_img auto-start01.png %}

### 可访问外网后再自动执行脚本

可以通过配置 systemd 来完成，但也可以在脚本里自己写条件判断。这里我需要在可访问外网后再做内网穿透，脚本如下：

```bash
#!/bin/sh
# Start/stop the frp-setup daemon.
#
### BEGIN INIT INFO
# Provides:          frp-setup
# Required-Start:    $all
# Required-Stop:
# Default-Start:     2 3 4 5
# Default-Stop:
# Short-Description: 用于内网穿透
### END INIT INFO

ip="xx.xx.xx.xx"
port=123
frpPath="/usr/local/frp/frp_0.30.0_linux_arm64"

while :
do
	/bin/nc $ip $port -w 1 -z
	res=$?
	# 这里注意不能用 [[]] 双方括号，因为在 sh 中会有问题，实验结果是会永远走到 else 逻辑
	# 有些脚本系统在执行时可能会强制使用 sh 而非 bash，所以尽量用在 sh 中兼容的语法，同样还可能会有问题的是单/双/三等号的用法
	if [ $res -ne 0 ];then
		sleep 5
	else
    # 重定向 nohup 生成的日志到 /var/log/frp.log
		nohup $frpPath/frpc -c $frpPath/frpc.ini > /var/log/frp.log 2>&1 &
		break
	fi
done
```



## /etc/network/if-up.d

/etc/network/ 目录下的子目录都与网络配置相关。

 {% asset_img auto-start02.png %}

| 目录名（/etc/network/ 下） |        说明        |
| :------------------------: | :----------------: |
|         if-down.d          | 网卡关闭时执行动作 |
|       if-post-down.d       | 网卡关闭后执行动作 |
|        if-pre-up.d         | 网卡启动前执行动作 |
|          if-up.d           | 网卡启动时执行动作 |
|         interface          |    网卡配置信息    |
|        interface.d         |    网卡扩展配置    |

将脚本放到 if-down.d、if-post-down.d、if-pre-up.d、if-up.d 下会在网卡相应状态变化时执行。这里要注意，如果网卡有多张，那么脚本就可能执行多次。