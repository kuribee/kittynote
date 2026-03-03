## docker架构



1. [参考课程的笔记](https://dockertips.readthedocs.io/en/latest/)

2. docker的架构，如图所示：

   <img src=".\images\docker架构.png" alt="省" style="zoom:80%;" />

3. 可以看到docker的整体架构是模块化的，且属于client-server模式。

4. Docker公司开发了名为Libcontainer的自研工具，用于替代LXC。Libcontainer的目标是成为与平台无关的工具，可基于不同内核为Docker上层提供必要的容器交互功能。

5. 开放容器计划（OCI）。OCI定义了两个容器相关的规范（或者说标准）： 镜像规范； 容器运行时规范。

6. 所有的容器运行代码在一个单独的OCI兼容层中实现。默认情况下，Docker使用runc来实现这一点。runc是OCI容器运行时标准的参考实现。

7. runc实质上是一个轻量级的、针对Libcontainer进行了包装的命令行交互工具。runc生来只有一个作用—创建容器，这一点它非常拿手，速度很快！不过它是一个CLI包装器，实质上就是一个独立的容器运行时工具。因此直接下载它或基于源码编译二进制文件，即可拥有一个全功能的runc。但它只是一个基础工具，并不提供类似Docker引擎所拥有的丰富功能。有时也将runc所在的那一层称为“OCI层”。

8. 所有的容器执行逻辑被重构到一个新的名为containerd（发音为container-dee）的工具中。它的主要任务是容器的生命周期管理—start | stop | pause | rm....  但containerd如果在k8s中，则还有管理镜像的功能。在docker中镜像管理主要是由daremon负责。不过，所有的额外功能都是模块化的、可选的，便于自行选择所需功能。

9. 当使用Docker命令行工具执行命令时，Docker客户端会将其转换为合适的API格式，并发送到正确的API端点。API是在daemon中实现的。这套功能丰富、基于版本的REST API已经成为Docker的标志，并且被行业接受成为事实上的容器API。一旦daemon接收到创建新容器的命令，它就会向containerd发出调用。daemon已经不再包含任何创建容器的代码了！daemon使用一种CRUD风格的API，通过gRPC与containerd进行通信。虽然名叫containerd，但是它并不负责创建容器，而是指挥runc去做。containerd将Docker镜像转换为OCI bundle，并让runc基于此创建一个新的容器。然后，runc与操作系统内核接口进行通信，基于所有必要的工具（Namespace、CGroup等）来创建容器。容器进程作为runc的子进程启动，启动完毕后，runc将会退出。

10. 前面提到，containerd指挥runc来创建新容器。事实上，每次创建容器时它都会fork一个新的runc实例。不过，一旦容器创建完毕，对应的runc进程就会退出。因此，即使运行上百个容器，也无须保持上百个运行中的runc实例。一旦容器进程的父进程runc退出，相关联的containerd-shim进程就会成为容器的父进程。作为容器的父进程，shim的部分职责如下。● 保持所有STDIN和STDOUT流是开启状态，从而当daemon重启的时候，容器不会因为管道（pipe）的关闭而终止。● 将容器的退出状态反馈给daemon。



## docker容器和镜像

1. 镜像：Docker image是一个 `read-only` 文件。这个文件包含文件系统，源码，库文件，依赖，工具等一些运行application所需要的文件。可以理解成一个模板。docker image具有分层的概念

2. 容器：“一个运行中的docker image”。实质是复制image并在image最上层加上一层 `read-write` 的层 （称之为 `container layer` ,容器层）。基于同一个image可以创建多个container

3. 容器目的就是运行应用或者服务，这意味着容器的镜像中必须包含应用/服务运行所必需的操作系统和应用文件。但是，容器又追求快速和小巧，这意味着构建镜像的时候通常需要裁剪掉不必要的部分，保持较小的体积。例如，Docker镜像通常不会包含6个不同的Shell让读者选择—通常Docker镜像中只有一个精简的Shell，甚至没有Shell。镜像中还不包含内核—容器都是共享所在Docker主机的内核。所以有时会说容器仅包含必要的操作系统（通常只有操作系统文件和文件系统对象）。

4. docker + 管理的对象（比如容器，镜像） + 具体操作（比如创建，启动，停止，删除）

5. 运行容器默认是attached模式，容器的输入输出会显示在终端，终端的输入输出也会影响到容器，即ctrl+c。但windows中不会，即ctrl+c没有影响到容器

6. -it是什么，`-i`：stdin 保持打开，`-t`：分配一个伪终端（TTY）。-it只对「会使用 stdin / TTY 的前台程序」有意义。Docker 的默认行为是：

   - stdout / stderr：自动接到宿主终端，或 docker logs
   - stdin：默认是关闭的（EOF）

7. -d，-itd则是detached模式运行，在后台运行。可以使用docker attach，连接到容器的主进程的 STDIN / STDOUT，但此时终端的输入输出会影响到容器（windows除外）。因为docker attach不是新起进程，而是直接“接管”容器启动时的那个进程。如果主进程退出，容器就停了。`Ctrl + C` 可能直接把容器干掉

8. docker exec 是在正在运行的容器里，新起一个进程。不影响容器主进程。退出 shell 容器还在。适合“进去看看 / 调试”

9. 如果不与镜像的设计冲突，例如没有entrypoint指定的命令。那么docker run -it /bin/bash会指定容器的主命令（pid=1）是/bin/bash。即运行时启动一个以 前台交互的bash容器，用户直接进入shell与容器交互。退出是会导致命令结束从而容器关闭。此外，docker run后面可以指定任何命令，例如python xxx.py。只有容器可以执行该命令

10. 一个容器停不停止取决于其执行的命令，跟attached还是detached无关。另外什么是交互式与非交互式shell，交互式模式就是在终端上执行，shell等待你的输入，并且立即执行你提交的命令。这种模式被称作交互式是因为shell与用户进行交互。这种模式也是大多数用户非常熟悉的：登录、执行一些命令、退出。当你退出后，shell也终止了。shell也可以运行在另外一种模式：非交互式模式，以shell script(非交互)方式执行。在这种模式 下，shell不与你进行交互，而是读取存放在文件中的命令,并且执行它们。当它读到文件的结尾EOF，shell也就终止了。所以这就说明了docker run -it /bin/bash不会停止容器的原因。

11. ```shell
    #批量操作，例如批量停止
    docker container stop $(docker container ps -aq)
    ```

12. 正如4中提到的docker中的完整命令是怎么样的，但有时可以省略第二部分，这是为了兼容早期的设计，下图举了一些例子，（当然个人看法是能明白写的命令是操作什么就行，这条笔记只是记录下原来有这么回事）：

    <img src=".\images\docker省略.png" alt="省" style="zoom:80%;" />

13. docker container logs可以查看容器的日志。-f可以实时跟踪日志

14. Docker 的核心能力依赖 Linux 内核；在 Windows 上运行 Docker Server，本质上一定是“借用”一个 Linux 内核，通常通过虚拟机实现（例如 hyper-v，WSL2）。

15. 通过以docker exec等方式进入容器，再ps，是以容器视角看pid。如果是docker container top是以宿主机视角看pid。展示的pid是不一样的

16. docker镜像的获取方式，

    - docker pull从仓库中拉取。通过docker push可以将自己的镜像推送到仓库中
    - docker load读取一个别人docker save的tar包。docker load可以用在离线环境，例如从别人电脑导出到u盘上，利用u盘复制到本机。
    - 可以dockerfile自己编写构建。

17. 一个镜像的完整路径是 [registry域名]/[namespace]/[repository] [:tag]。如果是官方registry，则域名可以省略，如果是官方镜像，则namespace可以省略。

18. 如果一个镜像，有容器基于该镜像运行，那么该镜像无法删除。可以强制删，或者容器删除，再删除镜像

19. 为镜像打标签命令的格式是docker image tag <current-tag> <new-tag>，其作用是为指定的镜像添加一个额外的标签，并且不需要覆盖已经存在的标签。

20. 一个镜像可以根据用户需要设置多个标签。这是因为标签是存放在镜像元数据中的任意数字或字符串。

21. latest是一个非强制标签，不保证指向仓库中最新的镜像！

22. Docker提供--filter参数来过滤docker image ls命令返回的镜像列表内容。

23. 那些没有标签的镜像被称为悬虚镜像，在列表中展示为<none>:<none>。通常出现这种情况，是因为构建了一个新镜像，然后为该镜像打了一个已经存在的标签。当此情况出现，Docker会构建新的镜像，然后发现已经有镜像包含相同的标签，接着Docker会移除旧镜像上面的标签，将该标签标在新的镜像之上。

24. 可以通过docker image prune命令移除全部的悬虚镜像。如果添加了-a参数，Docker会额外移除没有被使用的镜像（那些没有被任何容器使用的镜像）。docker system prune -a删除所有停止容器

25. 所有的Docker镜像都起始于一个基础镜像层，当进行修改或增加新的内容时，就会在当前镜像层之上，创建新的镜像层。即镜像是由多个只读的镜像层构成。

26. 镜像本身就是一个配置对象，其中包含了镜像层的列表以及一些元数据信息。镜像层才是实际数据存储的地方（比如文件等，镜像层之间是完全独立的，并没有从属于某个镜像集合的概念）。镜像的唯一标识是一个加密ID，即配置对象本身的散列值。每个镜像层也由一个加密ID区分，其值为镜像层本身内容的散列值。这意味着修改镜像的内容或其中任意的镜像层，都会导致加密散列值的变化。所以，镜像和其镜像层都是不可变的，任何改动都能很轻松地被辨别。这就是所谓的内容散列（Content Hash）。

27. 在推送和拉取镜像的时候，都会对镜像层进行压缩来节省网络带宽以及仓库二进制存储空间。但是压缩会改变镜像内容，这意味着镜像的内容散列值在推送或者拉取操作之后，会与镜像内容不相符！这显然是个问题。例如，在推送镜像层到Docker Hub的时候，Docker Hub会尝试确认接收到的镜像没有在传输过程中被篡改。为了完成校验，Docker Hub会根据镜像层重新计算散列值，并与原散列值进行比较。因为镜像在传输过程中被压缩（发生了改变），所以散列值的校验也会失败。为避免该问题，每个镜像层同时会包含一个分发散列值（Distribution Hash）。这是一个压缩版镜像的散列值，当从镜像仓库服务拉取或者推送镜像的时候，其中就包含了分发散列值，该散列值会用于校验拉取的镜像是否被篡改过。这个内容寻址存储模型极大地提升了镜像的安全性，因为在拉取和推送操作后提供了一种方式来确保镜像和镜像层数据是一致的。该模型也解决了随机生成镜像和镜像层ID这种方式可能导致的ID冲突问题。

28. Docker通过存储引擎（新版本采用快照机制）的方式来实现镜像层堆栈，并保证多镜像层对外展示为统一的文件系统。Docker在Linux上支持很多存储引擎（Snapshotter）。每个存储引擎都有自己的镜像分层、镜像层共享以及写时复制（CoW）技术的具体实现。但是，其最终效果和用户体验是完全一致的。尽管Windows只支持一种存储引擎，还是可以提供与Linux相同的功能体验。

29. docker login可以登录到远程仓库，从而上传

30. docker commit本质是把容器的“可写层”保存成一个新的镜像 layer。原镜像的 layer 还在，做的改动变成了一个新 layer，镜像结构仍然是 **多层 Union FS**

31. Docker 1.10中引入了新的内容寻址存储模型。作为模型的一部分，每一个镜像现在都有一个基于其内容的密码散列值。为了讨论方便，本书用摘要代指这个散列值。因为摘要是镜像内容的一个散列值，所以镜像内容的变更一定会导致散列值的改变。这意味着摘要是不可变的。只需要在docker image ls命令之后添加--digests参数即可在本地查看镜像摘要。

32. Scratch是一个空的Docker镜像。可以通过scratch来构建一个基础镜像。

33. `docker image history` 命令用于查看指定镜像的历史层信息，它显示了镜像创建过程中的每一层，包括创建时间、创建者、大小和注释等信息。

34. docker history命令显示了镜像的构建历史记录，但其并不是严格意义上的镜像分层。例如，有些Dockerfile中的指令并不会创建新的镜像层。比如ENV、EXPOSE、CMD以及ENTRY- POINT。不过，这些命令会在镜像中添加元数据。

35. 多架构镜像（Multi-architecture Image）的出现解决了如何自动获取符合平台的镜像！Docker（镜像和镜像仓库服务）规范目前支持多架构镜像。这意味着某个镜像仓库标签（repository:tag）下的镜像可以同时支持64位Linux、PowerPC Linux、64位Windows和ARM等多种架构。简单地说，就是一个镜像标签之下可以支持多个平台和架构。为了实现这个特性，镜像仓库服务API支持两种重要的结构：Manifest列表（新）和Manifest。Manifest列表是指某个镜像标签支持的架构列表。其支持的每种架构，都有自己的Mainfest定义，其中列举了该镜像的构成。图左侧是Manifest列表，其中包含了该镜像支持的每种架构。Manifest列表的每一项都有一个箭头，指向具体的Manifest，其中包含了镜像配置和镜像层数据。

    <img src=".\images\mani.png" alt="省" style="zoom:80%;" />

36. 构建多架构的镜像方式有很多种。例如buildx，

37. 如果某个镜像层被多个镜像共享，那只有当全部依赖该镜像层的镜像都被删除后，该镜像层才会被删除。

38. tag并不能代表一个镜像，例如修改镜像使用相同tag(例如1.5)。此时就需要基于tag的pull镜像方式，可以保证拉取的是自己想要的镜像。

39. 在虚拟机模型中，首先要开启物理机并启动Hypervisor引导程序（本书跳过了BIOS和Bootloader代码等）。一旦Hypervisor启动，就会占有机器上的全部物理资源，如CPU、RAM、存储和NIC。Hypervisor接下来就会将这些物理资源划分为虚拟资源，并且看起来与真实物理资源完全一致。然后Hypervisor会将这些资源打包进一个叫作虚拟机（VM）的软件结构当中。这样用户就可以使用这些虚拟机，并在其中安装操作系统和应用。

40. 与虚拟机模型相同，OS也占用了全部硬件资源。在OS层之上，需要安装容器引擎（如Docker）。容器引擎可以获取系统资源，比如进程树、文件系统以及网络栈，接着将资源分割为安全的互相隔离的资源结构，称之为容器。每个容器看起来就像一个真实的操作系统，在其内部可以运行应用。

41. 从更高层面上来讲，Hypervisor是硬件虚拟化（Hardware Virtualization）—Hypervisor将硬件物理资源划分为虚拟资源；另外，容器是操作系统虚拟化（OS Virtualization）—容器将系统资源划分为虚拟资源。

42. 直至明确删除容器前，容器都不会丢弃其中的数据。就算容器被删除了，如果将容器数据存储在卷中，数据也会被保存下来。

43. docker container stop命令就有礼貌多了（就像用枪指着容器的脑袋然后说“你有10s时间说出你的遗言”）。该命令给容器内进程发送将要停止的警告信息，给进程机会来有序处理停止前要做的事情。一旦docker stop命令返回后，就可以使用docker container rm命令删除容器了。这背后的原理可以通过Linux/POSIX信号来解释。docker container stop命令向容器内的PID 1进程发送了SIGTERM这样的信号。就像前文提到的一样，会为进程预留一个清理并优雅停止的机会。如果10s内进程没有终止，那么就会收到SIGKILL信号。这是致命一击。但是，进程起码有10s的时间来“解决”自己。docker container rm <container> -f命令不会先友好地发送SIGTERM，这条命令会直接发出SIGKILL。

44. 通常建议在运行容器时配置好重启策略。这是容器的一种自我修复能力，可以在指定事件或者错误后重启来完成自我修复。重启策略应用于每个容器，可以作为参数被强制传入docker-container run命令中，或者在Compose文件中声明（在使用Docker Compose以及Docker Stacks的情况下）。截至本书撰写时，容器支持的重启策略包括always、unless-stopped和on-failed。

45. always策略是一种简单的方式。除非容器被明确停止，比如通过docker container stop命令，否则该策略会一直尝试重启处于停止状态的容器。一种简单的证明方式是启动一个新的交互式容器，并在命令后面指定--restart always策略，同时在命令中指定运行Shell进程。当容器启动的时候，会登录到该Shell。退出Shell时会杀死容器中PID为1的进程，并且杀死这个容器。但是因为指定了--restart always策略，所以容器会自动重启。--restart always策略有一个很有意思的特性，当daemon重启的时候，停止的容器也会被重启。

46. always和unless-stopped的最大区别，就是那些指定了--restart unless-stopped并处于Stopped (Exited)状态的容器，不会在Docker daemon重启的时候被重启。

47. on-failure策略会在退出容器并且返回值不是0的时候，重启容器。就算容器处于stopped状态，在Docker daemon重启的时候，容器也会被重启。

48. 
