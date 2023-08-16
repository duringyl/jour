### 一、原理

1. 将本机的22端口转发到远程机的6660端口，从而远程机使用127.0.0.1:6660可访问本机   
> `ssh -NfR 6660:localhost:22 proxy@10.0.20.6`


2. 将本机外部的6660端口转发到本机的内部6660端口  
> `ssh -NfL 10.0.20.6:6660:127.0.0.1:6660 proxy@10.0.20.6`



### 二、场景(Linux操作系统已测)

1. 家里局域网有一台`Linux`，上面有一个`hello`用户，这个机器能联网，这个机器需要被代理
2. 互联网上有一个`proxy@123.123.123.123`的服务器(买的云服)(有公网IP)，作为跳板服务器

### 三、设置方式

1. 登录被代理机器，任何用户都行，执行: 
> `ssh -NfR 6660:localhost:22 proxy@123.123.123.123` 
> #将本地的22端口转发到远程服务器的6660端口

2. 登录互联网的代理服务器, 任意用户都行，执行: 
> `ssh -NfL 123.123.123.123:6660:127.0.0.1:6660 proxy@123.123.123.123` 
> #将本机外部IP的6660端口的流量转发到本机127.0.0.1的6660端口上，而127.0.0.1:6660恰好是SSH反向代理监听的端口，于是会将这些流量转发到被代理机器上

3. 上面的两个命令执行之后都会在后台运行，退出登录设置依然有效

4. 互联网代理服务器设置好防火墙，将其6660端口开放

5. 任意找一台能联网的机器，执行以下命令可以登录被代理的机器: 
> `ssh hello@123.123.123.213 -p6660`  

6. 另外可以设置ssh连接不超时，避免超时而导致反向代理失效

### 四、警告

1. 互联网上的代理服务器的proxy用户和被代理的内网登录用户一定不要使用弱密码!!!
2. 被代理的Linux最好关闭ssh远程root登录

### 五、被代理服务器自动脚本案例

##### 脚本 `~/auto_reverse_proxy.sh`:

```sh
#!/bin/bash

# 代理服务器信息
proxyAddr=121.37.241.41
proxyUser=proxy
proxyPass='xxxxxx'
proxyPort=6660

# 被代理服务器信息(运行脚本的本机)
localAddr=localhost
localPort=22

# 日志文件
logf=~/.auto_reverse_proxy.log
still=`ps -ef | grep "$proxyAddr" | grep -v grep | wc -l`
#echo "    `ps -ef | grep "$proxyAddr" | grep -v grep`" >> $logf

# 本机需要安装expect工具, 
# $still小于1说明与代理服务器的连接没有建立, 执行expect脚本尝试进行连接
if [ $still -lt 1 ]; then
    expect << EOF
spawn ssh -NfR $proxyPort:$localAddr:$localPort $proxyUser@$proxyAddr
set timeout 1
expect "yes/no" {send "yes\n"}
set timeout 10
expect "*password:" {send "${proxyPass}\n"}

set timeout 1
expect eof
EOF
    # 再次检查是否有正在后台运行的代理任务, 小于1说明前面的连接执行失败
    still=`ps -ef | grep "$proxyAddr" | grep -v grep | wc -l`
    if [ $still -lt 1 ]; then
        result='failed'
    else
        result='done'
    fi
    echo "`date +%F\ %T`: auto reconf reverse proxy $result" >> $logf
else
    echo "`date +%F\ %T`: session still on" >> $logf
fi

```

##### cron任务
    crontab -e                              # 命令打开cron任务编辑界面, 在其中加入下方内容
    */5 * * * * ~/auto_reverse_proxy.sh     # 设置每5分钟检测一次
