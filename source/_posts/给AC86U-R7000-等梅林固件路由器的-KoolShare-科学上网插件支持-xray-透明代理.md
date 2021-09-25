---
title: 给 AC86U/R7000 等梅林固件路由器的 KoolShare 科学上网插件支持 Xray 透明代理
date: 2021-09-25 16:53:02
tags:
---

之前在 AC86U 上的 KoolShare 改版固件安装了科学上网插件，并开启了 ss + v2ray 插件的方式；但是由于这种方式性能太低，跑到 50M 带宽就把路由器的 CPU 跑满了。

最近看到 Xray 的 Vless + XTLS 的方案，尝试了下能在 AC86U 上把 300 M 带宽跑满。

这里分享一下如何复用原本的科学上网插件的情况下使用 Xray 做透明代理。
![](/images/ss-tool.png)

先决条件：
- 这里假设已经完成了科学上网的设置，并使用 ss 方式能跑通的前提下。
- 路由器已经开启了 ssh 。
  
# 1. 安装配置 Xray 并设置开启启动

## 下载 Xray 二进制
在 /jffs/ 目录下新建一个 xray 的目录，比如 /jffs/xray/
从 Xray github [release 页面](https://github.com/XTLS/Xray-core/releases) 下载 Xray 最新版的 二进制文件：

AC86U 是 armv8 架构的，则下载 [Xray-linux-arm64-v8a.zip](https://github.com/XTLS/Xray-core/releases/download/v1.4.5/Xray-linux-arm64-v8a.zip)

如果是 R7000 则是 v5 架构的，请选择 [Xray-linux-arm32-v5.zip](https://github.com/XTLS/Xray-core/releases/download/v1.4.5/Xray-linux-arm32-v5.zip)

总之根据自己路由器平台选择二进制文件。

## 上传并配置到路由器中
解压并把 xray 文件上传到 /jffs/xray/ 目录 
可以使用 scp 命令复制
```
scp xray admin@192.168.50.1 /jffs/xray/
```

（记得在路由器内给 xray 文件加上可执行权限 chmod +x /jffs/xray/xray）

然后 ssh 进入路由器，在 /jffs/xray/目录下 创建 conf.json 的 Xray 配置:
```
{
    "log": {
        "loglevel": "none"
    },
    "inbounds": [
        {
            "port": 23456,
            "protocol": "socks",
            "settings": {
                "auth": "noauth",
                "udp": true,
                "ip": "127.0.0.1",
                "clients": null
            }
        },
        {
            "listen": "0.0.0.0",
            "port": 3333,
            "protocol": "dokodemo-door",
            "settings": {
                "network": "tcp,udp",
                "followRedirect": true
            }
        }
    ],
    "outbounds": [
        {
            "protocol": "vless",
            "settings": {
                "vnext": [
                    {
                        "address": "你的地址",
                        "port": 443,
                        "users": [
                            {
                                "id": "xxxx",
                                "flow": "xtls-rprx-splice",
                                "encryption": "none",
                                "level": 0
                            }
                        ]
                    }
                ]
            },
            "streamSettings": {
                "network": "tcp",
                "security": "xtls",
                "xtlsSettings": {
                    "serverName": "你的地址域名"
                }
            }
        }
    ]
}
```

## 配置开机启动脚本
/jffs/scripts/ 目录下新建开机启动脚本 xray-start.sh 

```
# vi  /jffs/scripts/xray-start.sh 
```

填入：
```
#!/bin/sh
nohup /jffs/xray/xray run -c /jffs/xray/conf.json > /jffs/xray/out.log 2>&1 &
```

并给到执行权限：
```
chmod +x /jffs/scripts/xray-start.sh 
```

此时就可以执行以下命令启动 Xary 了（注意此时如果开启了科学上网插件的话会启动失败，因为我们上面配置的 Xary 3333 和 23456 端口都被科学上网插件启动的 ss-local 和 ss-redir 给占用了，下面会介绍如何禁用这两个进程，用我们的 Xray 来替代）

```
/jffs/scripts/xray-start.sh 
```

如果想关闭 Xray 进程，则可以执行：
```
ps | grep xray | grep -v grep | awk '{print $1}' | xargs kill
```

最后配置一下开机启动：
```
vi /jffs/scripts/services-start
```

在 /jffs/scripts/services-start 最后加入 /jffs/scripts/xray-start.sh ,最后 xray-start.sh 内容如下
```
#!/bin/sh
/koolshare/bin/ks-services-start.sh
/jffs/scripts/xray-start.sh # 新增这行
```

此时就可以开机自动启动 Xray 啦～

注意如果使用科学上网的大陆白名单策略的话需要修改梅林的 ntp 服务器为国内服务器（如：cn.ntp.org.cn），否则路由器启动时同步时间失败导致 tls 握手会失败。
在梅林的 系统管理-〉系统设置下找到 ntp 服务并修改：
![](/images/ntp.png)

## 修改科学上网脚本
最后一步，修改科学上网脚本，让它不启动 ss-local 和 ss-redir 进程，改成 Xray 代替。


```
vi /jffs/.koolshare/ss/ssconfig.sh
```

找到 start_sslocal 函数，在该函数第一行添加 return 即可：
```
start_sslocal() {                           
        return   #直接返回                                             
        if [ "$ss_basic_type" == "1" ]; then
                echo_date 开启ssr-local，提?.socks5代理端口：23456
                rss-local -l 23456 -c $CONFIG_FILE -u -f /var/run/sslocal1.pid >/dev/null 2>&
        elif [ "$ss_basic_type" == "0" ]; then                                               
                echo_date 开启ss-local，提?.socks5代理端口：23456                            
                if [ "$ss_basic_ss_obfs" == "0" ] && [ "$ss_basic_ss_v2ray" == "0" ]; then   
                        ss-local -l 23456 -c $CONFIG_FILE -u -f /var/run/sslocal1.pid >/dev/n
                else                                                                         
                        ss-local -l 23456 -c $CONFIG_FILE $ARG_OBFS -u -f /var/run/sslocal1.p
                fi                                                                           
        fi                                                                                   
}
```

然后再找到 start_ss_redir 函数，同样的在该函数第一行添加 return 即可：
```
start_ss_redir() {
        return                                                                                       
        if [ "$ss_basic_type" == "1" ]; then                                                 
                echo_date 开启ssr-redir?.程，用于透明代理.                                   
                BIN=rss-redir                                                                
                ARG_OBFS=""  
...       
```

然后回到科学上网插件，直接点击保存就可以啦（科学上网页面上显示的 ss 配置现在已经完全不会被用到了，可以随便填，真正起到效果的是 /jffs/xray/conf.json 这个 xray 的配置），享受 xray 的飞快速度吧～