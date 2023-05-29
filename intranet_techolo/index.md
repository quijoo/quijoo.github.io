# [周末小项目] 内网穿透技术 实践

## 前言
以前了解过 [内网穿透技术](https://blog.xuaii.cc/zhuan-zai-p2p-nei-wang-chuan-tou-ji-zhu/) 是什么， 但是没有实践应用。


## 起因
这周末玩 apex 时发现即使是在 cpu端跑推理引擎还是会导致帧率下降和突然卡顿问题，经过测试排除了 截图和移动鼠标部分带来的问题，只可能是推理端的问题。


## 问题
为了优化这个问题，修改了 [auto-aim](https://github.com/quijoo/auto-aim-yolov8-openvino) 代码（拉了个分支将推理程序放到服务端，将截屏 + 移动鼠标 放到客户端）

这样我的确能在局域网内多放一台机器然后部署 AI，又或者在 云服务器上跑 AI 程序，但是这不够优雅。

我希望在任意两台机器上都能分别独立的部署 AI 端和 执行端。所以有了试试内网穿透的想法。很幸运，一来我就发现了 **[FRP](https://github.com/fatedier/frp)** 工具，它能很容易的对 TCP/UDP/HTTP/HTTPS 进行内网穿透(转发模式 和 p2p模式)

拿到这个工具就开始实践吧！

## 实践

根据 frp 文档的示例 [xtcp](https://gofrp.org/docs/examples/xtcp/)
**frp服务器**
```
// frps.ini
[common]
bind_port = 7000
bind_udp_port = 7000
```

**frp客户端1 推理引擎**
```
// frp_ai.ini
[common]
server_addr = x.x.x.x
server_port = 7000

[p2p_ssh]
type = xtcp
# 只有 sk 一致的用户才能访问到此服务
sk = abcdefg
local_ip = 127.0.0.1
local_port = 22
```

**frp客户端2 自瞄程序**
```
// frp_aim.ini
[common]
server_addr = x.x.x.x
server_port = 7000

[p2p_ssh_visitor]
type = xtcp
# xtcp 的访问者
role = visitor
# 要访问的 xtcp 代理的名字
server_name = p2p_ssh
sk = abcdefg
# 绑定本地端口用于访问 ssh 服务
bind_addr = 127.0.0.1
bind_port = 6000
```

好了各个机器配置文件都写好了，那就启动吧
```
frps -c frps.ini
frpc -c frp_ai.ini
frpc -c frp_aim.ini

```

然后收到了 timeout 报错，去仓库查 issue， 有个老哥说需要路由器开启 DMZ。 DMZ 是什么东西？

    它是为了解决安装防火墙后外部网络不能访问内部网络服务器的问题，而设立的一个位于内部网络与外部网络之间的缓冲区，在这个网络区域内可以放置一些公开的服务器资源。
    
哦，原来路由器的防火墙将我们的数据包都丢掉了，所以需要将我们的 frp_ai.ini 这一端的 局域网 IP 加入移动光猫的 DMZ区即可。

一般来说光猫需要超级管理员账号才能修改该项配置。一般来说百度搜索光猫型号就能搜索到获取管理员账号和密码的教程。

**我的设备的流程**

![](vx_images/408514200230570.png)

好，这下大功告成了！

确实，aim 端已经能移动了，一看fps 14。怎么回事呢？到底是发送的图片数据量太大了还是网络延迟本身太高了？

于是写了一段 python 的 socket 代码来检查延迟，哦原来本机到本机经过内网穿透后都有 50ms 延迟。怎么会这样呢？？？

目前初步的猜测是 数据包发出去后经过了漫长的路由才又回到本机。。贞真费力啊


## 更进一步的想法

既然 frp 是有效的，只是延迟比较高，那么就可以通过这项技术完成更多有趣的事情。

1. 部署 GitLab 服务，暴露到外网
2. 本机搭建博客
3. 部署 NAS 服务器
4. 和好朋友之间玩游戏直接通过内网穿透


