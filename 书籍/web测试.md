##### 使用Docker构建并测试Web应用程序

测试一个基于Sinatra的Web应用程序，而不是静态网站。这个应用程序会接受输入参数，并使用JSON散列输出这些参数。

##### 构建Sinatra应用程序

使用Dockerfile构建一个基础镜像，并用这个镜像来开发 Sinatra Web应用程序。

```
[root@vlnx251105 ~]# mkdir sinatra
[root@vlnx251105 ~]# cd sinatra/

[root@vlnx251105 sinatra]# vim Dockerfile

FROM centos
MAINTAINER zhao yonggang zhaoyg@test.com

RUN yum update -y

# install Ruby RubyGem
RUN yum install -y centos-release-scl
RUN yum install -y rh-ruby22 rh-ruby22-ruby-devel build-essential redis-tools
RUN scl enable rh-ruby22 bash
RUN yum install -y gcc gcc-c++ make
RUN ln -s /opt/rh/rh-ruby22/root/usr/lib64/libruby.so.2.2 /lib64/

# use gem command install sinatra json redis
RUN /opt/rh/rh-ruby22/root/usr/bin/gem install --no-rdoc --no-ri sinatra json redis

RUN mkdir -p /opt/webapp

EXPOSE 4567

# web application bootfile 
CMD [ "/opt/webapp/bin/webapp" ]


[root@vlnx251105 sinatra]# docker build -t zhaoyg/sinatra .




# install Ruby RubyGem
RUN yum install -y rubygems ruby-devel
RUN yum install -y gcc gcc-c++ make

# use gem command install sinatra json redis
RUN gem install json
RUN gem install redis --version 3.2
RUN gem install rack --version 1.6
RUN mkdir -p /opt/webapp
```

##### 创建Sinatra容器

Sinatra Web应用程序的源代码，在webapp目录下，有bin和lib两个目录组成。需要使用chmod命令保证 webapp/bin/webapp文件可以执行

```
[root@vlnx251105 sinatra]# vim webapp/bin/webapp 
#!/opt/rh/rh-ruby22/root/usr/bin/ruby

[root@vlnx251105 sinatra]# chmod +x webapp/bin/webapp 


[root@vlnx251105 sinatra]# docker run -d -p 4567 --name webapp -v $PWD/webapp:/opt/webapp zhaoyg/sinatra
5fe8e9ddee9e41320cbee79c9a52e34be840c9c1dedaed683c8cb417ef33e82b
```

指定了一个新卷$PWD/webapp,来存放新的Sinatra Web应用程序，并将这个卷挂载在Dockerfile里创建的目录/opt/webapp。没有在命令行中提供要运行的命令，而是在镜像的Dockerfile中的CMD指令中提供了一条命令

```
CMD [ "/opt/webapp/bin/webapp" ]
```

```
[root@vlnx251105 sinatra]# docker logs -f webapp
[2017-07-16 05:47:41] INFO  WEBrick 1.3.1
[2017-07-16 05:47:41] INFO  ruby 2.2.2 (2015-04-13) [x86_64-linux]
== Sinatra (v2.0.0) has taken the stage on 4567 for development with backup from WEBrick
[2017-07-16 05:47:41] INFO  WEBrick::HTTPServer#start: pid=1 port=4567
```

从日志中可以看出，容器中已经启动了Sinatra，而且WEBrick服务进程正在监听4567端口，等待测试。

查看Docker容器里正在运行的进程

```
[root@vlnx251105 sinatra]# docker top webapp
UID                 PID                 PPID                C                   STIME               TTY                 TIME                CMD
root                6274                6260                0                   13:47               ?                   00:00:00            /opt/rh/rh-ruby22/root/usr/bin/ruby /opt/webapp/bin/webapp
```

查看WEBrick服务监听的4567端口映射到本地宿主机的那个端口。

```
[root@vlnx251105 sinatra]# docker port webapp 4567
0.0.0.0:32774
```

目前，Sinatra应用还很基础，没做什么。它只是接收输入参数，并将输入转化为JSON输出。可以使用curl 命令来测试。

```
[root@vlnx251105 sinatra]# curl -i -H 'Accpet: application/json' -d 'name=Foo&status=Bar' http://localhost:32774/json
HTTP/1.1 200 OK 
Content-Type: text/html;charset=utf-8
Content-Length: 29
X-Xss-Protection: 1; mode=block
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
Server: WEBrick/1.3.1 (Ruby/2.2.2/2015-04-13)
Date: Sun, 16 Jul 2017 06:03:14 GMT
Connection: Keep-Alive

{"name":"Foo","status":"Bar”}
```

可以看到，给Sinatra应用程序传入一些参数，并看到这些参数转化成JSON散列后的输出：

```
{"name":"Foo","status":"Bar”}。成功！
```

##### 构建Redis镜像和容器

现在扩展Sinatra应用程序，加入后端数据库Redis，并在Redis数据库中存储输入的参数。为了达到这个目的，要构建全新的镜像和容器来运行Redis数据库，之后，要利用Docker的特性来关联两个容器。

```
[root@vlnx251105 ~]# mkdir redis
[root@vlnx251105 ~]# cd redis/
[root@vlnx251105 redis]# vim Dockerfile

FROM centos
MAINTAINER zhao yonggang zhaoyg@test.com
RUN yum update -y
RUN yum install -y epel-release
RUN yum install -y redis
EXPOSE 6379
ENTRYPOINT ["/usr/bin/redis-server"]
CMD []

[root@vlnx251105 redis]# docker build -t zhaoyg/redis .

[root@vlnx251105 redis]# docker run -d -p 6379 --name redis zhaoyg/redis

[root@vlnx251105 redis]# docker port redis 6379
0.0.0.0:32776
```

使用Redis客户端 redis-cli 连接到 127.0.0.1 的 32776 端口，验证了 Redis服务器正在正常工作。

```
[root@vlnx251105 redis]# redis-cli -h 127.0.0.1 -p 32776
127.0.0.1:32776>
```

##### 连接到Redis 容器

现在更新Sinatra应用程序，让其连接到Redis并存储传入的参数。

一种方法是，Docker自己的网络栈。Docker容器可以公开端口并绑定到本地网络接口，这样可以把容器里的服务在本地Docker宿主机所在的外部网络上公开。

还有一种就是内部网络。在安装Docker时，会创建一个新的网络接口，名字时docker0。每个Docker容器都会在这个接口上分配一个IP地址。

```
[root@vlnx251105 ~]# ip addr show dev docker0
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP 
    link/ether 02:42:7e:e2:fe:b3 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:7eff:fee2:feb3/64 scope link 
       valid_lft forever preferred_lft forever
```

docker0 接口为私有IP地址，范围时172.16～172.30（Docker会默认使用 172.17.x.x 作为子网地址，除非已经这个子网已经被占用。如果子网被占用，Docker会在172.16～172.30这个范围内尝试创建子网）。接口本身的地址 172.17.0.1是这个Docker网络的网关地址，也是所有Docker容器的网关地址。

docker0 是一个虚拟的以太网桥，用于连接容器和本地宿主网络。进一步查看Docker宿主机的其他网络接口，会发现一系列名字以veth开头的接口。

```
[root@vlnx251105 ~]# ip addr show 

49: vethd427c6f@if48: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP 
    link/ether 8e:cf:1b:c8:a3:12 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::8ccf:1bff:fec8:a312/64 scope link 
       valid_lft forever preferred_lft forever
57: veth2b5ead5@if56: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP 
    link/ether b6:26:f4:0b:67:cb brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet6 fe80::b426:f4ff:fe0b:67cb/64 scope link 
       valid_lft forever preferred_lft forever
```

Docker 每创建一个容器就会创建一组互联的网络接口。这组接口就像管道的两端（从一端发送的数据会在另一端接收到）。这组接口其中一端作为容器里的eth0接口，而另一端统一命名为类似vethd427c6f@if48 这种名字，作为宿主机的一个端口。可以把 veth接口认为是虚拟网线的一端。这条虚拟网线一端插在名为docker0的网桥上，另一端插到容器里。通过把每个veth\*接口绑定到docker0网桥，Docker创建了一个虚拟子网，这个子网由宿主机和所有的Docker容器共享。

```
[root@vlnx251105 ~]# docker run -t -i centos /bin/bash
[root@e774acdcf3de /]# yum install iproute
[root@e774acdcf3de /]# ip addr show dev eth0
58: eth0@if59: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP 
    link/ether 02:42:ac:11:00:04 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.4/16 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe11:4/64 scope link 
       valid_lft forever preferred_lft forever
```

Docker给容器分配了IP地址 172.17.0.4 作为宿主虚拟接口的另一端。这样就可以让宿主机和容器互相通信了。

从容器内跟踪对外通信的路由。

```
[root@e774acdcf3de /]# yum install traceroute

[root@e774acdcf3de /]# traceroute baidu.com
traceroute to baidu.com (123.125.114.144), 30 hops max, 60 byte packets
 1  172.17.0.1 (172.17.0.1)  0.164 ms  0.011 ms  0.009 ms
 2  192.168.251.2 (192.168.251.2)  0.162 ms  0.110 ms  0.155 ms
 3  * * *
```

可以看到容器地址后的下一跳时宿主网络上 docker0 接口的网关IP 172.17.0.1.

不过Docker网络还有另一个部分配置才能允许建立连接：防火墙规则和NAT配置。这些配置允许Docker在宿主网络和容器间路由。

```
[root@vlnx251105 ~]# iptables -t nat -L -n
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination         
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT)
target     prot opt source               destination         

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination         
DOCKER     all  --  0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination         
MASQUERADE  all  --  172.17.0.0/16        0.0.0.0/0           
MASQUERADE  tcp  --  172.17.0.2           172.17.0.2           tcp dpt:4567
MASQUERADE  tcp  --  172.17.0.3           172.17.0.3           tcp dpt:6379

Chain DOCKER (2 references)
target     prot opt source               destination         
RETURN     all  --  0.0.0.0/0            0.0.0.0/0           
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:32774 to:172.17.0.2:4567
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:32776 to:172.17.0.3:6379
```

容器默认时无法访问的。从宿主网络与容器通信时，必须明确指定打开的端口。DNAT这个规则，把容器里的访问路由到Docker宿主机的32776端口。

##### 连接Redis

使用docker inspect命令来查看新的Redis容器的网络配置

```
[root@vlnx251105 ~]# docker inspect redis

   "NetworkSettings": {
            "Bridge": "",
            "SandboxID": "7eabbecc61b79ee57c28d10019b843086bf9a72c6845b69ec057d8d38168c02c",
            "HairpinMode": false,
            "LinkLocalIPv6Address": "",
            "LinkLocalIPv6PrefixLen": 0,
            "Ports": {
                "6379/tcp": [
                    {
                        "HostIp": "0.0.0.0",
                        "HostPort": "32776"
                    }
                ]
            },
            "SandboxKey": "/var/run/docker/netns/7eabbecc61b7",
            "SecondaryIPAddresses": null,
            "SecondaryIPv6Addresses": null,
            "EndpointID": "951c5ab7be4a1621fa3b3dc210056d717126357b1ecf475d465f882fc312c177",
            "Gateway": "172.17.0.1",
            "GlobalIPv6Address": "",
            "GlobalIPv6PrefixLen": 0,
            "IPAddress": "172.17.0.3",
            "IPPrefixLen": 16,
            "IPv6Gateway": "",
            "MacAddress": "02:42:ac:11:00:03",
            "Networks": {
                "bridge": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": null,
                    "NetworkID": "e0d75486322dcc5f5854033c3dba24b40a2eb6612ca44289c1d7aa04d2df22e3",
                    "EndpointID": "951c5ab7be4a1621fa3b3dc210056d717126357b1ecf475d465f882fc312c177",
                    "Gateway": "172.17.0.1",
                    "IPAddress": "172.17.0.3",
                    "IPPrefixLen": 16,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "02:42:ac:11:00:03"
                }
            }


[root@vlnx251105 ~]# docker inspect -f '{{ .NetworkSettings.IPAddress }}' redis
172.17.0.3
```

容器的IP地址为172.17.0.3，并使用docker0接口作为网关地址。还可以看到6379端口被映射到本地宿主机的32776端口。由于运行在本地的Docker宿主机上，不一定要用映射后的端口，也可以直接使用172。17.0.3地址与Redis服务器的6379端口通信。

```
[root@vlnx251105 ~]# redis-cli -h 172.17.0.3
172.17.0.3:6379>
```

但是如果重启容器（docker restart redis），容器的IP可能会改变，要是应用程序对Redis容器的IP地址做了硬编码，就无法连接到Redis数据库了。还好 Docker有个叫做 连接（link）的功能。

##### 让Docker容器互联

```
[root@vlnx251105 ~]# docker run --name redis01 zhaoyg/redis

[root@vlnx251105 sinatra]# docker run -p 4567 --name webapp01 --link redis01:db -t -i -v $PWD/webapp:/opt/webapp zhaoyg/sinatra /bin/bash
[root@715e7c933e9e /]#
```

--link选项创建了两个容器间的父子连接。这个选项需要两个参数：一个是要连接的容器名字（redis01），另一个是连接后容器的别名（db）。别名可以无须关注底层容器名字，来访问公开的信息。连接让父容器可以访问子容器，并且把子容器的一些连接细节分享给父容器，这些细节有属于配置应用程序并使用这个连接。

连接也有一些安全上的好处。启动的Redis容器并没有使用 -p 选项公开Redis的端口。因为不需要那么做。通过把容器连接在一起，可以让父容器直接访问任意子容器的公开端口。只有使用 --link 连接到这个容器的容器才能连接到这个端口。容器不需要对本地宿主机公开，这就可以限制容器化应用程序被攻击的可能，减少应用暴露的网络。

也可以把多个容器连接在一起，比如，Redis服务与多个 Web应用程序。

```
docker run -p 4567 --name webapp02 --link redis01:db -t -i -v $PWD/webapp:/opt/webapp zhaoyg/sinatra /bin/bash
 docker run -p 4567 --name webapp03 --link redis01:db -t -i -v $PWD/webapp:/opt/webapp zhaoyg/sinatra /bin/bash
```

Docker在父容器中在以下两个位置保存连接信息：

```
[root@715e7c933e9e /]# cat /etc/hosts
127.0.0.1localhost
::1localhost ip6-localhost ip6-loopback
fe00::0ip6-localnet
ff00::0ip6-mcastprefix
ff02::1ip6-allnodes
ff02::2ip6-allrouters
172.17.0.5db af2d3cac4ca7 redis01
172.17.0.4715e7c933e9e

[root@715e7c933e9e /]# ping db
PING db (172.17.0.5) 56(84) bytes of data.
64 bytes from db (172.17.0.5): icmp_seq=1 ttl=64 time=0.209 ms
64 bytes from db (172.17.0.5): icmp_seq=2 ttl=64 time=0.072 ms

[root@715e7c933e9e /]# env | grep ^DB
DB_NAME=/webapp01/db
DB_PORT_6379_TCP_PORT=6379
DB_PORT=tcp://172.17.0.5:6379
DB_PORT_6379_TCP=tcp://172.17.0.5:6379
DB_PORT_6379_TCP_ADDR=172.17.0.5
DB_PORT_6379_TCP_PROTO=tcp
```

这些环境变量会随容器不同而变化，取决于容器是如何配置的，例如，Dockerfile中由ENV和EXPOSE指定定义的内容

##### 使用容器连接来通信

第一种方法，使用环境环境变量里的一些连接信息。查看Web应用程序里的 lib/app.rb文件是如何利用这些新的环境变量的。

```
[root@715e7c933e9e /]# vi /opt/webapp/lib/app.rb 

require "rubygems"
require "sinatra"
require "json"
require "uri"
require "redis"

class App < Sinatra::Application

  uri = URI.parse(ENV['DB_PORT'])
  redis = Redis.new(:host => uri.host, :port => uri.port)


  set :bind, '0.0.0.0'

  get '/' do
    "<h1>DockerBook Test Sinatra app</h1>"
  end

  post '/json/?' do
    params.to_json
  end

end
```

使用Ruby的 URI模块来解析 DB\_PORT环境变量。

另一种是使用本地DNS

```
[root@715e7c933e9e /]# vi /opt/webapp/lib/app.rb 

require "rubygems"
require "sinatra"
require "json"
require "redis"

class App < Sinatra::Application

  redis = Redis.new(:host => 'db', :port => '6379')

  set :bind, '0.0.0.0'

  get '/' do
    "<h1>DockerBook Test Sinatra app</h1>"
  end

  post '/json/?' do
    params.to_json
  end

end
```

应用程序会到/etc/hosts文件中查找db。

```
[root@715e7c933e9e /]# nohup /opt/webapp/bin/webapp &



[root@vlnx251105 ~]# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                     NAMES
715e7c933e9e        zhaoyg/sinatra      "/bin/bash"              33 minutes ago      Up 33 minutes       0.0.0.0:32780->4567/tcp   webapp01
af2d3cac4ca7        zhaoyg/redis        "/usr/bin/redis-serve"   35 minutes ago      Up 35 minutes       6379/tcp                  redis01
7f291269ad7a        zhaoyg/redis        "/usr/bin/redis-serve"   About an hour ago   Up 42 minutes       0.0.0.0:32779->6379/tcp   redis
5fe8e9ddee9e        zhaoyg/sinatra      "/opt/webapp/bin/weba"   2 hours ago         Up 2 hours          0.0.0.0:32774->4567/tcp   webapp


[root@vlnx251105 ~]# curl -i -H 'Accept: application/json' -d 'name=Foo&status=Bar' http://localhost:32780/json
HTTP/1.1 200 OK 
Content-Type: text/html;charset=utf-8
Content-Length: 29
X-Xss-Protection: 1; mode=block
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
Server: WEBrick/1.3.1 (Ruby/2.2.2/2015-04-13)
Date: Sun, 16 Jul 2017 08:09:51 GMT
Connection: Keep-Alive

{"name":"Foo","status":"Bar"}
```

这个概念可以扩展到别的应用程序栈，用于本地开发中做复杂的管理，例如

* Wordpress 、HTML、CSS 和 JavaScript
* Ruby on Rails
* Django 和 Flask
* Node.js



