首先需要了解docker命令的两种模式：client模式和daemon模式


#### 1 client模式

```
docker [OPTIONS] COMMAND [arg...]
```

其中OPTION参数成为flag，任何时候执行一个docker命令，docker都需要先解析这些flag，，然后按照用户声明的command向指定的子命令执行对应的操作。

如果子命令为daemon，docker就会创建一个运行的宿主机的daemon进程，即执行daemon模式。其余子命令都会执行client模式。处于client模式下的docker命令工作流程包含以下步骤。

1. 解析flag信息。

docker命令支持大量的OPTION，或者说flag，这里列出对于运行在client模式下的docker比较重要的一些flag。

* debug，对应-D和--debug参数，它将向系统中添加DEBUG环境变量且赋值为1，并把日志显示等级调为DEBUG级，这个flag用于启动调试模式。
* loglevel，对应-l和--log-level参数，默认等级为info，即只输出普通的操作信息。用户可以指定的日志等级现在由panic、fatal、error、warn、info、debug这几种。
* hosts，对应-H和--host=[]参数，对于client模式，就是指本次操作需要连接的docker daemon位置，而对于daemon模式，则提供所要监听的地址。若hosts变量或者或者系统环境变量DOCKER_HOST不为空，说明用户指定了host对象；否则使用默认设定，默认情况下linux系统设置为unix:///var/run/docker.sock。
* protoAddrParts，这个信息来自于-H 参数中://前后的两部分的组合，即与docker daemon建立通信的协议方式与socket地址。

2. 创建client实例

client的创建就是在已有配置参数信息的基础上，调用api／client／cli.go#NewDockerCli，需要设置好proto（传输协议）、addr（host的目标地址）和tlsConfig（安全传输层协议的配置），另外还有配置标准输入输出及错误输出。


3.执行具体的命令

docker client对象创建成功后，剩下的执行具体命令的过程就交给cli/cli.go来处理了。

◉ 从命令映射到对应的方法

cli主要通过反射机制，从用户输入的命令（比如run）得到匹配的执行方法（比如cmdrun），这也是所谓“约定大于配置”的方法命名规范。

同时，cli会根据参数列表的长度判断是否用于多级docker命令支持（例如未来也许会加入一条命令，如docker group run可以指定一组docker容器一起运行某个命令），然后根据找到的执行方法，把剩余参数传入并执行。若传入的方法不合法或参数不正确，则返回docker命令的帮助信息并退出。

◉ 执行对应的方法，发起请求

找到具体的执行方法后，就可以执行，虽然请求内容会有所不同，但执行流程大致相同。

* 解析传入的参数，并针对参数进行配置处理
* 获取与docker daemon通信所需的认证配置信息
* 根据命令业务类型，给docker daemon 发送POST、GET等请求
* 读取来自docker daemon的返回结果

由此可见，在请求执行过程中，大多都是将命令行中关于请求的参数进行初步处理，并添加相应的辅助信息，最终通过指定的协议给docker daemon发送docker client API 请求，主要的任务执行均由docker daemon完成。

#### 2 daemon模式


docker运行时使用docker daemon子命令，就会运行docker daemon。

一旦docker进入了daemon模式，剩下的初始化和启动工作就都由docker的docker/daemon.go#cmddaemon来完成

docker daemon通过一个server模块（api/server/server.go）接收来自client的请求，然后根据请求类型，交由具体的方法去执行。因此daemon首先需要启动并初始化这个server。另一方面，启动server后，docker进程需要初始化一个daemon对象（daemon/daemon.go）来负责处理server接收到的请求。

下面是docker daemon启动与初始化过程的详细解析

1.API Server的配置和初始化过程

首先，在docker/daemon.go#cmddaemon中，docker会继续按照用户的配置完成server的初始化并启动它。这个server又被称为API server，就是专门负责响应用户请求并将请求脚本daemon具体方法去处理的过程。启动过程如下

(1) 整理解析用户指定的各项参数

(2) 创建PID文件

(3) 加载所需的server辅助配置，包括日志、是否允许远程访问、版本以及TLS认证信息等。

(4) 根据上述server配置，加上之前解析出来的用户指定的server配置（比如hosts），通过goroutine的方式启动api server。这个server监听的socket位置就是hosts的值。

(5) 创建一个负责处理业务daemon对象（对应daemon/daemon.go）作为负责处理用户请求的逻辑实体。

(6) 对APIserver中的路由表进行初始化，即将用户的请求和对应的处理函数相对应起来。

(7) 设置一个channel，保证上述goroutine只有在server出错的情况下才会退出

(8) 设置信号捕获，当docker daemon进程收到 INT、TERM、QUIT信号时，关闭API server，调用shutdowndaemon停止这个daemon。

(9) 如果上述操作都成功，API server就会与上述daemon绑定，并允许接受来自client连接。

(10) 最后，docker daemon进程向宿主机的init守护进程发送“READY=1”信号，表示这个docker daemon已经开始正常工作了。


shutdowndaemon关闭一个daemon的流程如下

(1) 创建并设置一个channel，使用select监听数据。在正确完成关闭daemon工作后将该channel关闭，标识该工作的完成；否则在超时（15秒）后报错。

(2) 调用daemon／daemon.go#shutdown方法执行如下工作

* 遍历所有运行中的容器，先使用SIGTERM软杀死容器进程，如果10秒不能完成，则使用SIGKILL强制杀死。
* 如果netController被初始化过，调用#libnetwork/controller.go#GC方法进行垃圾回收。
* 结束运行中的镜像存储驱动进程。


2. daemon 对象的创建与初始化过程

既然API server 是同daemon对象绑定起来共同完成工作的，那么创建daemon对象的这个过程对应的正是daemon/daemon.go#NewDaemon方法。

NewDaemon过程会按照docker的功能点，逐条为daemon对象所需的属性设置用户或者系统指定的值，这是一个相当复杂的过程。

这个过程需要完成的配置包括如下功能点：docker容器的配置信息、检测系统支持及用户权限、配置工作路径、加载并配置graphdriver、创建docker网络环境、创建并初始化镜像数据库、创建容器管理驱动、检测DNS配置和加载已有docker容器等。

◉ Docker容器的配置信息

容器配置信息的主要功能是：供用户自由配置docker容器的可选功能，使得docker容器的运行更贴近用户期待的运行场景。配置信息的处理包含以下几个部分。

* 设置默认的网络最大传输单元：当用户没有对 -mtu参数进行指定时，将其设置为1500.否则，沿用用户指定的参数值。
* 检测网桥配置信息：此部分配置为进一步配置docker网络提供铺垫。


◉ 检测系统支持及用户权限

初步处理完docker的配置信息之后，docker对自身运行的环境进行了一系列的检测，主要包括3个方面。

* 操作系统类型对docker daemon的支持，目前docker daemon只能运行在linux系统上。
* 用户权限的级别，必须是root权限。
* 内核版本与处理器的支持，只支持amd64架构的处理器，且内核版本必须升至3.10.0及以上。

◉ 配置daemon工作路径

配置docker daemon的工作路径，主要是创建docker daemon运行中所在的工作目录，默认为 /var/lib/docker。若该目录不存在，则会创建，并赋予0700权限。


◉ 配置docker容器所需的文件环境

这一步docker daemon会在docker工作根目录/var/lib/docker下面初始化一些重要的目录和文件，来构建docker容器工作所需的文件系统环境。

第一，创建容器配置文件目录。docker daemon在创建docker容器之后，需要将容器内的配置文件放到这个目录下统一管理。目录默认位置为：/var/lib/docker/containers，它下面会为每个具体容器保存如下几个配置文件.

```
[root@vlnx251106 ~]# ls /var/lib/docker/containers/
20a452cb9a7e2e20830012d33d97847ee279d7acca074066e77a2ad4e1615070

[root@vlnx251106 ~]# ls /var/lib/docker/containers/20a452cb9a7e2e20830012d33d97847ee279d7acca074066e77a2ad4e1615070/
checkpoints     hostconfig.json  hosts        resolv.conf.hash  shm
config.v2.json  hostname         resolv.conf  secrets

```

这些配置文件里包含了这个容器的所有元数据。


第二，配置graphdriver目录。它用于完成docker容器镜像管理所需的底层存储驱动层。所以，在这一步的配置工作就是加载并配置镜像存储驱动graphdriver，创建存储驱动管理镜像层文件系统所需的目录和环境，初始化镜像层元数据存储。

创建graphdriver时，首先会从环境变量DOCKER_DRIVER中读用户指定的驱动，若为空，则开始遍历优先级数组选择一个graphdriver。在Linux环境下，优先级从高到低依次为aufs、btrfs、zfs、devicemapper、overlay和vfs。在不同操作系统下，优先级列表的内容和顺序都会不同，而且随着内核的发展以及驱动的完善，会继续发生变化。

需要注意，目前vfs在docker中是用来管理volume的，并不作为镜像存储使用。另外，由于目前在overlay文件系统上运行的docker容器不兼容SELinux，因此当config中配置信息需要启用SELinux并且driver的类型为overlay时，该过程就会报错。

当识别出对应的driver（比如aufs）后，docker会执行这个driver对应的初始化方法（位于daemon/graphdriver/aufs/aufs.go)，这个初始化的主要工作包括：尝试加载内核aufs模块来确定docker主机支持aufs；发起statfs系统调用获取当前docker主目录（/var/lib/docker/)的文件系统信息，确定aufs是否支持该文件系统；创建aufs驱动根目录（默认:/var/lib/docker/aufs)并将该目录配置为私有挂载；在根目录下创建mnt、diff和layers目录作为aufs驱动的工作环境。


上述工作完成后，graphdriver的配置工作就完成了。

第三，配置镜像目录。主要工作是在docker主目录下创建一个image目录，来存储所有镜像和镜像层管理数据，默认目录为"/var/lib/docker/image／".在image目录下，每一个graphdriver都有一个具体的目录用于存储使用该graphdriver存储的镜像相关的元数据。

根据上一步graphdriver的选择情况（这里以aufs为例），创建image/aufs/layerdb/目录作为镜像层元数据存储目录，并创建MetadataStore用来管理这些元数据。根据graphdriver与元数据存储结构创建layerStore，用来管理所有的镜像层和容器层，将逻辑镜像层的操作映射到物理存储驱动层graphdriver的操作；创建用于对registry的镜像上传下载的uploadManager和downloadManager。

创建image/aufs/imagedb/目录用于存储镜像的元数据，并根据layerStore创建imageStore，用来管理镜像的元数据。

第四，调用volume/local/local.go#New创建volume驱动目录（默认为 /var/lib/docker/volumes），docker中volume是宿主机上挂载到docker容器内部的特定目录。volumes目录下有一个metadata.db数据库文件用于存储volume相关的元数据，其余以volume ID命令的文件夹用于存储具体的volume内容。默认的volume驱动是local，用户也可以通过插件的形式使用其他volume驱动来存储。

第五，准备“可信镜像”所需的工作目录。在docker工作根目录下创建trust目录。这个存储目录可以根据用户给出的可信url加载授权文件，用来处理可信镜像的授权和验证过程。

第六，创建 distributionMetadataStore和referenceStore。referenceStore用于存储镜像的仓库列表。记录镜像仓库的持久化文件位于docker根目录下的image/[graphdriver]/repositories.json中，主要用于做镜像ID与镜像仓库名之间的映射。distributionMetadataStore存储与第二版镜像仓库registry有关的元数据，主要用于做镜像层的diff_if与registry中镜像层元数据之间的映射。

第七，将持久化在docker根目录中的镜像、镜像层以及镜像仓库等的元数据内容恢复到daemon的imageStore、layerStore和referenceStore中。

第八，执行镜像迁移。由于docker 1.10版本以后，镜像管理部分使用了基于内容寻址存储（content-addressable storage）。升级到1.10以上的新版本后，在第一次启动daemon时，为了将老版本中的graph镜像管理迁移到新的镜像管理体系中，这里会检查docker根目录中是否存在graph文件夹，如果存在就会读取graph中的老版本镜像信息，计算校验和并将镜像数据写入到新版的imageStroe和layerStore中。需要注意的是，迁移镜像中计算校验和是一个非常占用CPU的工作，并且在未完成镜像迁移时，docker daemon是不会响应任何请求的，所以如果你本地的老版本镜像和容器比较多，或者是在对服务器负载和响应比较敏感的线上环境尝试升级docker版本，那就要注意妥善安排时间了。docker官方也提供了迁移工具让用户在老版本daemon运行的使用进行镜像的迁移。

综上，这里docker daemon需要在docker根目录（/var/lib/docker)下创建并初始化一系列跟容器问及爱你系统密切相关的目录和文件。

◉ 创建docker daemon网络

创建docker daemon运行环境的时候，创建网络环境是极为重要的一个部分。这不仅关系着容器对外的通信，同样也关系着容器间的通信。网路部分早已被抽离出来作为一个单独的模块，称为libnetwork。libnetwork通过插件的形式为docker提供网络功能，使得用户可以根据自己的需求实现自己的driver来提供不同网络功能。截止到docker1.10版本，libnetwork实现了host、null、bridge和overlay的驱动。其中bridge driver为默认驱动，和之前版本中的docker网络功能是基本等价的。需要注意的是，同之前的docker网络一样，bridge driver并不提供跨主机通信的能力，overlay driver则适用于多主机环境。


◉ 初始化execdriver

execdriver是docker 中用来管理docker容器的驱动。docker会调用execdriver中的NewDriver()函数来创建信息execdriver。

在创建execdriver的时候，需要注意以下5部分信息。

* 运行时中指定使用的驱动类型，在默认配置文件中默认使用native，即其对应的容器运行时为libcontainer；
* 用户定义的execdriver选项，即 -exec-opt参数值。
* 用户定义的 -exec-root参数值，docker execdriver运行的root路径，默认为 /var/run/docker；
* docker运行时的root路径，默认为 /var/lib/docker；
* 系统功能信息，包括容器的内存限制功能、交换区内存限制功能、数据转发功能以及AppArmor安全功能等。


AppArmor通过host主机是否存在 /sys/kernel/security/apparmor来判断是否加入 AppArmor配置。

最后，如果选择了native作为这个execdriver的驱动实现，上述driver的创建过程就会新建一个libcontainer，这个libcontainer会在后面创建和启动linux容器时发挥作用。


◉ daemon对象诞生

docker daemon进程在经过以上诸多设置以及创建对象之后，最终创建出了daemon对象实例，其属性总结如下。

* ID：根据传入的证书生成的容器ID，若没有传入则自动使用ECDSA加密算法生成。
* repository：部署所有docker容器的路径
* containers：用于存储具体dokcer容器信息的对象
* execCommands：docker容器所执行的命令
* referenceStore：存储docker镜像仓库名和镜像ID的映射。
* distributionMetadataStore：v2版registry相关的元数据存储。 
* trustKey：可信认证书
* idIndex：用于通过简短有效的字符串前缀定位唯一的镜像
* sysInfo：docker所在宿主机的系统信息
* configStore：docker所需要的配置信息
* execDriver：docker容器执行驱动，默认为native类型
* statsCollector：收集容器网络及cgroup状态信息。
* defaultLogConfig：提供日志的默认配置信息。
* registryService：镜像存储服务相关信息
* EventsService：事件服务相关信息
* volumes：volume所使用的驱动，默认为local类型
* root：docker运行的工作根目录
* uidMaps：uid的对应图
* gidMaps：gid的对应图
* seccomEnabled：是否使用seccompute。
* nameIndex：记录键和其名字的对应关系
* linkIndex：容器的link目录，记录容器的link关系



◉ 恢复已有的docker容器

当docker daemon启动时，会去查看在daemon.repository也就是在/var/lib/docker/containers中的内容。若有已经存在的docker容器，则将相应信息收集并进行维护，同时重启restart policy为always的容器。


综上所述，docker daemon的启动看起来非常复杂，，这是docker在演进的过程中不断添加功能点造成的。但不管今后docker的功能点增加多少，docker daemon进程的启动都将遵循以下3步。

(1) 首先是启动一个API Server，它工作在用户通过-H指定的socket上面

(2) 然后docker使用NewDaemon方法创建一个daemon对象来保存信息和处理业务逻辑 

(3) 最后将上述API Server和daemon对象绑定起来，接收并处理client的请求

只不过，NewDaemon方法的长度会不断增加而已。


#### 3 从client到daemon

一个已经在运行的daemon是如何相应并处理来自client的请求的呢？

1. 发起请求

(1) docker run命令开始运行，用户端的docker进入client模式。

(2) 经过初始化，新建出一个client

(3) 上述client通过反射机制找到了Cmdrun方法

CmdRun在解析过用户提供的容器参数等一系列操作后，最后发出了这样两个请求：

```
"POST", "/containers/create?"+containerValues //创建容器
"POST", "/containers/"+createResponse.ID+"/start" //启动容器
```
至此，client的主要任务结束。

daemon在启动后会维护一个API Server来响应上述请求，同样遵循“约定大于配置”的原则，daemon端负责响应第一个create请求的方法是：api/server/server.go#postContainersCreate.

在1.6版本及以前，docker daemon会将一个创建容器的Job交给所谓的docker engine来接管接下来的任务。不过这个过程已经被完全废弃并且再也不会回来了。

2. 创建容器

在这一步docker daemon并不需要真正创建一个linux容器，它只需要解析用户通过client提交的POST表单，然后使用这些参数在daemon中新建一个container对象出来即可。这个container实体就是container/container_unix.go，其中的CommonContainer字段定义在container/container.go中，为linux平台和windows平台上容器共有的属性，这里将linux平台上容器最重要的定义片段一并列举如下。

```
// Definition of Docker Container
ID                    string
Created               time.Time
Path                  string
Config                *runconfig.Config
ImageID               string `json:"Image"`
NetworkSettings       *network.Settings
Name                  string
ExecDriver            string 
RestartCount          int
UpdateDns             bool
MountPoints           map[string]*mountPoint
command               *execdriver.Command
monitor               *containerMonitor

...

AppArmorProfile       string
HostnamePath          string
HostsPath             string
ShmPath               string
MqueuePath            string
ResolvConfPath        string
SeccompProfile        string

```

这里需要额外注意的是daemon属性，即container是能够知道管理它的daemon进程信息的。

上述过程完成后，container的信息会作为Response返回给client，client会紧接着发送start请求。

3.启动容器

API Server接收到start 请求后会告诉docker daemon进行container启动容器操作，这个过程是daemon/start.go.

⚠️注意： 1.7版本以后的docker不仅把所有的client端的请求都使用了一个对应的api/client/{请求名称}.go文件来定义，在daemon端，所有请求的处理过程也放在daemon/{请求名称}.go文件中来定义。

此时，由于container所需的各项参数，比如NetworkSettings、ImageID等，都已经在创建容器过程中赋好了值，docker daemon会在start.go中直接执行daemon.ContainerStart，就能够在宿主机上创建对应的容器了。

创建容器的过程中由docker daemon来创建namespace，配置cgroup，挂载rootfs。 containerMonitor将daemon设置为自己的supervisor。经过一系列调用后，daemon.ContainerStart实际上执行操作的是：

```
containerMonitor.daemon.Run(container ...)
```

4.最后一步

在docker daemon完成了所有准备工作，下达了执行run操作的命令后，所有需要跟操作系统打交道的任务都交给ExecDriver.Run（具体是哪种Driver由container决定）来完成。

execdriver是daemon的一个重要组成部分，它封装了对namespace、cgroup等所有对OS资源进行操作的所有方法。而在docker中，execdriver的默认实现（native）就是libcontainer了。

在这最后一步，docker daemon只需要向execdriver提供如下三大参数，接着等待返回结果就可以了。

* commandv: 该容器需要的所有配置信息集合（container的属性之一）；
* pipes：用于将容器stdin、stdout、stderr重定向到daemon
* startCallback(): 回调方法。



















