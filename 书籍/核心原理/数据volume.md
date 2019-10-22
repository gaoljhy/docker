Docker 数据卷

Docker 的镜像是由一系列的只读层组合而来的，当启动一个容器时，Docker 加载镜像的所有只读层，并在最上层加入一个读写层。这个设计使得 Docker 可以提高镜像构建、存储和分发的效率，节省了时间和存储空间，然而也存在如下问题。

❑ 容器中的文件在宿主机上存在形式复杂，不能在宿主机上很方便地对容器中的文件进行访问。

❑ 多个容器之间的数据无法共享。

❑ 当删除容器时，容器产生的数据将丢失。

为了解决这些问题，Docker 引入了数据卷（volume）机制。volume 是存在于一个或多个容器中的特定文件或文件夹，这个目录以独立于联合文件系统的形式在宿主机中存在，并为数据的共享与持久化提供以下便利。

❑ volume 在容器创建时就会初始化，在容器运行时就可以使用其中的文件。

❑ volume 能在不同的容器之间共享和重用。

❑ 对 volume 中数据的操作会马上生效。

❑ 对 volume 中数据的操作不会影响到镜像本身。

❑ volume 的生存周期独立于容器的生存周期，即使删除容器，volume 仍然会存在，没有任何容器使用的 volume 也不会被 Docker 删除。

Docker 提供了 volumedriver 接口，通过实现该接口，我们可以为 Docker 容器提供不同的 volume 存储支持。当前官方默认实现了 local 这种 volumedriver，它使用宿主机的文件系统为 Docker 容器提供 volume。本节接下来的讨论都将默认针对 local 这种 volumedriver。

#### 1 数据卷的使用方式

为容器添加 volume，类似于 Linux 的 mount 操作，用户将一个文件夹作为 volume 挂载到容器上，可以很方便地将数据添加到容器中供其中的进程使用。多个容器可以共享同一个 volume，为不同容器之间的数据共享提供了便利。

1. 创建 volume

Docker1.9 版本引人了新的子命令，即 docker volume。用户可以使用这个命令对 volume 进行创建、查看和删除，与此同时，传统的-v 参数创建 volume 的方式也得到了保留。

用户可以使用 docker volume create 创建一个 volume，以下命令创建了一个指定名字的volume。

```
docker volume create --name vol_ simple
```

```
说明 
Docker 当前并未对 volume 的大小提供配额管理，用户在创建 volume 时也无法指定 volume 的大小。在用户使用 Docker 创建 volume 时，由于采用的是默认的 local volumedriver，所以 volume 的文件系统默认使用宿主机的文件系统，如果用户需要创建其他文件系统的 volume，则需要使用其他的 volumedriver。
```

用户在使用 docker run 或 docker create 创建新容器时，也可以使用-v 标签为容器添加 volume, 以下命令创建了一个随机名字的volume,并挂载到容器中的/data目录下。

```
docker run -d -v /data centos /bin/bash
```

以下命令创建了一个指定名字的volume,并挂载到容器中的/data目录下。

```
docker run -d -v vol_ simple:/data centos /bin/bash
```

Docker在创建volume的时候会在宿主机/var/lib/docker/volume/中创建-一个以volume ID 为名的目录，并将 volume 中的内容存储在名为_data 的目录下。

使用 docker volume inspect 命令可以获得该 volume 包括其在宿主机中该文件夹的位置等信息。

```
docker volume inspect vol_ simple
{
      {
          "Name": "vol_ simple",
          "Driver": "local",
          "Mountpoint": "/var/lib/docker/volumes/vol_ simple/_ data"
      }
}
```

2. 挂载 volume

用户在使用 docker run 或 docker create 创建新容器时，可以使用-v 标签为容器添加 volume。用户可以将自行创建或者由 Docker 创建的 volume 挂载到容器中，也可以将宿主机上的目录或者文件作为 volume 挂载到容器中。下面分别介绍这两种挂载方式。

用户可以使用如下命令创建volume,并将其创建的volume挂载到容器中的/data 目录下。

```
docker volume create --name vol_ simple

docker run -d -v vol_ simple:/data centos /bin/bash
```

如果用户不执行第一条命令而直接执行第二条命令的话, Docker 会代替用户来创建一个名为 vol_ simple的volume,并将其挂载到容器中的/data目录下。

用户也可以使用如下命令创建一个随机ID的volume，并将其挂载到/data 目录下。

```
docker run -d -v /data centos /bin/bash
```

以上命令都是将自行创建或者由 Docker 创建的 volume 挂载到容器中。Docker 同时也允许我们将宿主机上的目录挂载到容器中。

```
docker run -v /host/dir:/container/dir centos /bin/bash
```

使用以上命令将宿主机中的/host/dir文件夹作为一个volume挂载到容器中的/container/dir。文 件夹必须使用绝对路径，如果宿主机中不存在/host/dir,将创建一个空文件夹。在/host/dir文件夹 中的所有文件或文件夹可以在容器的/container/dir文件夹下被访问。如果镜像中原本存在/container/dir文件夹，该文件夹下原有的内容将被隐藏，以保持与宿主机中的文件夹一致。

用户还可以将单个的文件作为 volume 挂载到容器中。

```
docker run -it --name vol_ file -v /host/file:/container/file centos /bin/bash
```

使用上条命令将主机中的/host/file文件作为一个volume挂载到容器中的/container/file。文件必 须使用绝对路径，如果文件中不存在/host/file,则Docker会创建一个同名空目录。挂载后文件内 容与宿主机中的文件-致，也就是说如果容器中原本存在/container/ile,该文件将被隐藏。

将主机上的文件或文件夹作为 volume 挂载时，可以使用:ro 指定该 volume 为只读。

```
docker run -it --name vol_ read_ only -v /host/dir:/container/dir:ro centos /bin/bash
```

类似于 SELinux 这类的标签系统，可以在 volume 挂载时使用 z 和 Z 来指定该 volume 是否可以共享。Docker 中默认的是 z，即共享该 volume。用户也可以在挂载时使用 Z 来标注该 volume 为私有数据卷。

```
docker run -it --name vol_ unshared  -v /host/dir:/container/dir:Z centos /bin/bash
```

在使用 docker run 或 docker create 创建新容器时，可以使用多个-v 标签为容器添加多个volume。

```
docker run -it --name vol_ mult -v /data1 -v /data2 -v /host/dir:/container/dir centos /bin/bash
```

3. 使用 Dockerfile 添加 volume

使用 VOLUME 指令向容器添加 volume。

VOLUME /data

在使用 docker build命令生成镜像并且以该镜像启动容器时会挂载一个volume到/data。与上 文中 vol_ simple例子类似，如果镜像中存在/data 文件夹，这个文件夹中的内容将全部被复制到宿主机中对应的文件夹中，并且根据容器中的文件设置合适的权限和所有者。

类似地，可以使用 VOLUME指令添加多个 volume。

VOLUME ["/data1", "/data2"]

与使用 docker run -v 不同的是，VOLUME 指令不能挂载主机中指定的文件夹。这是为了保证 Dockerfile 的可移植性，因为不能保证所有的宿主机都有对应的文件夹。

需要注意的是，在 Dockerfile 中使用 VOLUME 指令之后的代码，如果尝试对这个 volume 进行修改，这些修改都不会生效。在下面的例子中，在创建 volume 后，尝试在其中添加一些初始化的文件并改变文件所有权”。

```
FROM ubuntu 
RUN useradd foo 
VOLUME /data 
RUN touch /data/file 
RUN chown -R foo:foo /data
```

通过这个Dockerfile创建镜像并启动容器后，该容器中存在用户foo,并且能看到在/data挂载 的volume,但是/data文件夹内并没有文件fle,更别说fle的所有者并没有被改变为foo。这是由于 Dockerfile 中除了 FROM 指令的每- -行都是基于上一行生成的临时镜像运行一个容器，执行一条指令并执行类似 docker commit 的命令得到一个新的镜像，这条类似 docker commit 的命令不会对挂载的 volume 进行保存。所以上面的 Dockerfile 最后两行执行时，都会在一个临时的容器上挂载 /data，并对这个临时的 volume 进行操作，但是这一行指令执行并提交后，这个临时的 volume 没有被保存，我们通过最后生成的镜像创建的容器所挂载的 volume 是没有操作过的。

如果想要对 volume 进行初始化或者改变所有者，可以使用以下方式。

```
FROM ubuntu
RUN useradd foo
RUN mkdir /data && touch /data/file
RUN chown -R foo: foo /data
VOLUME /data
```

通过这个 Dockerfile 创建镜像并启动容器后，volume 的初始化是符合预期的，这是由于在挂 载volume时，/data已经存在，/data中的文件以及它们的权限和所有者设置会被复制到volume 中。

此外，与 RUN 指令在镜像构建过程中执行不同，CMD 指令和 ENTRYPOINT 指令是在容器启动时执行，使用如下 Dockerfile 也可以达到对 volume 初始化的目的。

```
FROM ubuntu
RUN useradd foo
VOLUME /data
CMD touch /data/file && chown -R foo:foo /data
```
4. 共享 volume (--volumes-from)

在使用 docker run 或 docker create 创建新容器时，可以使用--volumes-from 标签使得容器与已有的容器共享 volume。

```
docker run --rm -it --name vol_use --volumes-from vol_ simple centos /bin/bash
```

新创建的容器 vol_use 与之前创建的容器 vol. Simple 共享 volume，这个 volume 目的目录也是 /data。如果被共享的容器有多个 volume（如上文中出现的 vol_ mult），新容器也将有多个 volume，并且其挂载的目的目录也与 vol_ mult 中的相同。

可以使用多个--volumes-from 标签，使得容器与多个已有容器共享 volume。

```
docker run --rm -it --name vol_use_mult --volumes-from vol_1 --volumes-from vol_2 centos /bin/bash
```


一个容器挂载了一个 volume，即使这个容器停止运行，该 volume 仍然存在，其他容器也可以使用--volumes-from 与这个容器共享 volume。如果有一些数据，比如配置文件、数据文件等，要在多个容器之间共享，一种常见的做法是创建一个数据容器，其他的容器与之共享 volume。

```
docker run --name vol_data -v /data centos echo "This is a data-only container"
docker run -it --name vol_share1 --volumes-from vol_data centos /bin/bash
docker run -it --name vol_share2 --volumes-from vol_data centos /bin/bash
```

上述命令首先创建了一个挂载了 volume 的数据容器 vol_data，这个容器仅仅输出了一条提示后就停止运行以避免浪费资源。接下来的两个容器 vol_ share1 和 vol_ share2 与这个数据容器共享这个 volume。这样就将两个需要共享数据的容器进行了较好的解耦，避免了容器之间因为共享数据而产生相互依赖。

5. 删除 volume

如果创建容器时从容器中挂载了volume,在/var/ib/docker/volumes 下会生成与 volume 对应的目录，使用 docker rm 删除容器并不会删除与 volume 对应的目录，这些目录会占据不必要的存储空间，即使可以手动删除，因为有些随机生成的目录名称是无意义的随机字符串，要知道它们是否与未被删除的容器对应也十分麻烦。所以在删除容器时需要对容器的 volume 妥善处理。在删除容器时一并删除 volume 有以下 3 种方法。


❑ 使用 docker volume rm  <volume_name> 删除 volume。

❑ 使用 docker rm -v  <container_name> 删除容器。

❑ 在运行容器时使用 docker run --rm，--rm 标签会在容器停止运行时删除容器以及容器所挂载的 volume。

需要注意的是，在使用 docker volume rm 删除 volume 时，只有当没有任何容器使用该 volume 的时候，该 volume 才能成功删除。另外两种方法只会对挂载在该容器上的未命名的 volume 进行删除，而会对用户指定名字的 volume 进行保留。

如果 volume 是在创建容器时从宿主机中挂载的，无论对容器进行任何操作都不会导致其在宿主机中被删除，如果不需要这些文件，只能手动删除它们。

6. 备份、恢复或迁移 volume

volume 作为数据的载体，在很多情况下需要对其中的数据进行备份、迁移，或是从已有数据 恢复。以上文中创建的容器 vol_simple 为例，该容器在/data 挂载了一个volume。如果需要将这里 面的数据备份，一个很容易想到的方法是使用 docker inspect 命令查找到/data 在宿主机上对应的文件夹位置，然后复制其中的内容或是使用 tar 进行打包；同样地，如果需要恢复某个 volume 中的数据，可以查找到 volume 对应的文件夹，将数据复制进这个文件夹或是使用 tar 从存档文件中恢复。这些做法可行但并不值得推荐，下面推荐一个用--volumes -from 实现的 volume 的备份与恢复方法。

备份 volume 可以使用以下方法。

```
docker run --rm --volumes-from vol_simple -v $(pwd):/backup centos tar cvf /backup/data.tar /data
```

vol_ simple 容器包含了我们希望备份的一个 volume，上面这行命令启动了另外一个临时的容器，这个容器挂载了两个 volume，第一个 volume 来自于 vol_ simple 容器的共享，也就是需要备份的 volume, 第二个volume将宿主机的当前目录挂载到容器的/backup下。容器运行后将要备份的内容(/data文件夹)备份到/backup/data.tar，然后删除容器，备份后的 data.tar 就留在了当前目录。

恢复 volume 可以使用以下方法。

```
docker run -it --name vol_ bck -v /data ubuntu /bin/bash
docker run --rm --volumes-from vol_ bck -v $(pwd):/backup centos tar xvf /backup/data.tar -C /
```

首先运行了一个新容器作为数据恢复的目标。第二行指令启动了一个临时容器，这个容器挂载了两个 volume，第一个 volume 与要恢复的 volume 共享, 第二个 volume将宿主机的当前目录挂载 到容器的/backup下。由于之前备份的data.tar在当前目录下，那么它在容器中的/backup也能访问 到，容器启动后将这个存档文件中的/data恢复到根目录下，然后删除容器，恢复后的数据就在 vol_bck 的 volume 中了。




