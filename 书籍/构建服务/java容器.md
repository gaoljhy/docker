使用Dcoker构建一个Java应用服务

* 构建一个镜像从URL拉取指定的WAR文件并将其保存到卷里。
* 构建一个含有Tomcat服务器的镜像于行这些下载的WAR文件。

```
[root@vlnx251105 ~]# mkdir fetcher
[root@vlnx251105 ~]# cd fetcher/
[root@vlnx251105 fetcher]# vim Dockerfile

FROM centos
MAINTAINER zhao yonggang <zhaoyg@test.com>

RUN yum update -y
RUN yum install -y wget

VOLUME [ "/usr/local/tomcat8/webapps/" ]
WORKDIR /usr/local/tomcat8/webapps/

ENTRYPOINT [ "wget" ]
CMD [ "-?" ]
```

容器执行时，使用wget从指定的URL获取文件并把问价保存在 /var/lib/tomcat8/webapps/目录。这个目录也是一个卷，并且是所有容器的工作目录。然后将这个卷共享给Tomcat服务器并且运行里面的内容。

如果没有指定URL，ENTRYPOINT和CMD指令会让容器运行，在容器不带URL运行的时候，这两条指令通过返回wget帮助来做到这一点。

```
[root@vlnx251105 fetcher]# docker build -t zhaoyg/fetcher .
```

##### WAR文件的获取器

从Tomcat官网下载Apache Tomcat 实例应用来启动新镜像。

```
[root@vlnx251105 fetcher]# docker run -t -i --name sample zhaoyg/fetcher https://tomcat.apache.org/tomcat-8.0-doc/appdev/sample/sample.war
--2017-07-17 13:48:08--  https://tomcat.apache.org/tomcat-8.0-doc/appdev/sample/sample.war
Resolving tomcat.apache.org (tomcat.apache.org)... 62.210.60.236, 88.198.26.2, 140.211.11.105, ...
Connecting to tomcat.apache.org (tomcat.apache.org)|62.210.60.236|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 4606 (4.5K)
Saving to: 'sample.war'

100%[======================================>] 4,606       --.-K/s   in 0s      

2017-07-17 13:48:10 (147 MB/s) - 'sample.war' saved [4606/4606]
```

从输出结果看不出最终的保存路径，但是因为设置了容器的工作目录，sample.war 文件最终会保存到/var/lib/tomcat7/webapps/目录中。

可以在/var/lib/docker目录找到这个war文件，先用docker inspect命令查找卷的存储位置。

```
[root@vlnx251105 fetcher]# docker inspect -f "{{ .Mounts }}" sample
[{2b99a29ae183f1278c973ab114f8332a2b26e3c8728a702a8a189db80694b34e /var/lib/docker/volumes/2b99a29ae183f1278c973ab114f8332a2b26e3c8728a702a8a189db80694b34e/_data /usr/local/tomcat8/webapps local  true }]


[root@vlnx251105 fetcher]# ls -l /var/lib/docker/volumes/2b99a29ae183f1278c973ab114f8332a2b26e3c8728a702a8a189db80694b34e/_data
总用量 8
-rw-r--r--. 1 root root 4606 8月   6 2013 sample.war
```

##### Tomcat8 应用服务器

构建tomcat应用服务器的镜像来运行这个WAR文件。

```
[root@vlnx251105 ~]# mkdir tomcat
[root@vlnx251105 ~]# cd tomcat/
[root@vlnx251105 tomcat]# vim Dockerfile

FROM centos
MAINTAINER zhao yonggang <zhaoyg@test.com>

RUN yum update -y
RUN yum install -y java

COPY tomcat8/ /usr/local/tomcat8

VOLUME [ "/usr/local/tomcat8/webapps/" ]

EXPOSE 8080

ENTRYPOINT [ "/usr/local/tomcat8/bin/catalina.sh", "run" ]


[root@vlnx251105 tomcat]# docker build -t zhaoyg/tomcat8 .

[root@vlnx251105 tomcat]# docker run --name sample_app --volumes-from sample -d -P zhaoyg/tomcat8
```

这个容器会复用 sample容器里的卷。也就是说存储在/usr/local/tomcat8/webapps/卷里的WAR文件会从sample容器挂载到sample\_app容器。最终被tomcat加载并执行

```
[root@vlnx251105 tomcat]# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                     NAMES
4029d06715e7        zhaoyg/tomcat8      "/usr/local/tomcat8/b"   26 seconds ago      Up 25 seconds       0.0.0.0:32771->8080/tcp   sample_app


[root@vlnx251105 tomcat]# docker port sample_app 8080
0.0.0.0:32771
```

[http://192.168.251.105:32771/sample/](http://192.168.251.105:32771/sample/)

![](/assets/0F280D60-1235-48F8-8161-8001F8242E04.png)

##### 基于Tomcat应用服务器的构建服务

构建一个简单的基于Sinatra的web应用TProv。这个应用可以通过网页自动展示Tomcat应用。

```
[root@vlnx251105 tomcat]# yum install -y centos-release-scl
[root@vlnx251105 tomcat]# yum install -y rh-ruby22 rh-ruby22-ruby-devel
[root@vlnx251105 tomcat]# ln -s /opt/rh/rh-ruby22/root/usr/lib64/libruby.so.2.2 /lib64/
[root@vlnx251105 tomcat]# yum install -y gcc gcc-c++ make
[root@vlnx251105 tomcat]# /opt/rh/rh-ruby22/root/usr/bin/gem install --no-rdoc --no-ri tprov

[root@vlnx251105 tomcat]# /opt/rh/rh-ruby22/root/usr/local/bin/tprov
[2017-07-17 22:19:52] INFO  WEBrick 1.3.1
[2017-07-17 22:19:52] INFO  ruby 2.2.2 (2015-04-13) [x86_64-linux]
== Sinatra (v2.0.0) has taken the stage on 4567 for development with backup from WEBrick
[2017-07-17 22:19:52] INFO  WEBrick::HTTPServer#start: pid=11323 port=4567
```

[http://192.168.251.105:4567](http://192.168.251.105:4567)

![](/assets/BF80B683-67D3-45D1-AA08-405663A103B6.png)

     62         cid = `docker run --name "#{name}" zhaoyg/fetcher "#{url}" 2>&1`.cho    p   
     63         puts cid
     64         [$?.exitstatus == 0, cid]
     65       end
     66   
     67       def create_instance(name)
     68         cid = `docker run -P --volumes-from "#{name}" -d -t zhaoyg/tomcat8 2    >&1`.chop

![](/assets/77090655-CEBD-47A1-B8A6-6F7F3E5F1394.png)

![](/assets/1404BBF7-2842-4AB8-A126-257B0D8AB461.png)









