# 翻墙正确的姿势 frp逆向网络通信实现反监测翻墙
### 原理
	不了解frp是什么东西的恶补下https://www.zhihu.com/question/276309837/answer/386270615， https://www.appinn.com/frp/ 如果你用过花生壳那应该比较好理解。
	首先你要明白中国的网络防火墙只能劫持和控制中国内地发出去的tcp package，而一般来讲国外的ip往中国内地发tcp package长城是不会去检测的。举个通俗的例子，你在内地不能直接访问google，但是人家老外可是随便来访问咱家baidu。你再想想frp是什么东西，人家原本是做内网穿透的。我们平常理解的局域网不就是咱可以往外发tcp package，而由于种种原因外部不可以给内部发tcp package吗？
	类比下我们发现站在老外的角度，整个除中国以外的机器可以比喻为局域网，而中国的网络可以比喻为公网。其实没有绝对的局域网和公网，局域网和公网永远是相对而言的。“相对论”明白不？呵呵。


##### 准备两台服务器，一台海外的（非必须有独立公网ip），一台内地的(必须有独立公网ip)
###### 我这里有两台
	xxx.xxx.180.175   #香港 你可理解为是局域网用来搭建frpc
	xxx.xxx.122.101   #大陆 你可理解为是公网用来搭建frps

###### xxx.xxx.122.101 frps采用默认配置就可以
```shell
[common]
bind_port = 7000
```
###### 配置完frpserver 之后启动frpserver
```shell
./frps -c frps.ini #简单粗暴喜欢可以加nohup变为后台启动
```
###### xxx.xxx.180.175 配置http代理，我喜欢用goproxy。你用你喜欢的姿势
	安装golang环境。
	go get -v github.com/elazarl/goproxy
	vim goproxy.go
```golang
package main

import (
    "github.com/elazarl/goproxy"
    "log"
    "net/http"
)

func main() {
    proxy := goproxy.NewProxyHttpServer()
    proxy.Verbose = true
    //为了防止阿里云检测海外主机是否有翻墙行为我们把服务开在127.0.0.1,这样外网是检测不到你开了httpproxy的
    log.Fatal(http.ListenAndServe("127.0.0.1:5269", proxy))
}
```
	go run goproxy.go &
###### 检测本地代理是否ok
```shell
curl -x 127.0.0.1:5269 "http://www.google.com"  
#如果输出一个html文本说明你本地搭建的http代理是ok的
```

###### xxx.xxx.180.175 frpc配置如下
```shell
[common]
#对接公网的frpserver
server_addr = xxx.xxx.122.101
server_port = 7000
#你本地启动的需要frpserver帮你映射的http代理或者socks5或者你喜欢的都行，这里为了你好理解我就开http代理。
[http proxy]
type = tcp
local_ip = 127.0.0.1  #本地http代理ip
local_port = 5269 #本地http代理端口
remote_port = 5269  #你想让将http代理服务映射到frpserver的哪个端口可以和local_port不一样。但是使用的时候请注意变更。
```
###### 配置完frpclient 之后启动frpclient
```shell
./frpc -c frpc.ini #简单粗暴喜欢可以加nohup变为后台启动
```
###### 如果以上步骤全部OK，那么你的海外机器在本地开启的http代理服务已经被映射到了内地的“公网”服务器上面。
###### 验证下
```shell
curl -x xxx.xxx.122.101:5269 "http://www.google.com"
#我看到了一串html文本.....
```
###### 为什么要搞两台服务器，我只搞一台海外的，然后用花生壳或者其他的内网穿透服务行不行。
	不行，这些商用服务的内网穿透是会检测你是否用来做翻墙行为的。千万不要用。自己内地搞一台服务器，比较稳。
###### 有多稳定？
	每年6月4日，别人家的全挂了。我的还是正常工作，而且是爬虫在使用。
###### 爬虫场景多ip这个怎么实现？
	proxychains4，tcp forward各种姿势玩，你想搞什么样的都可以。网络套字节的东西很简单，工具也非常多。师傅领进门，成败在个人。
