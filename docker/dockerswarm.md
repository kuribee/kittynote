## docker swarm

1. Docker Swarm包含两方面：一个企业级的Docker安全集群，以及一个微服务应用编排引擎。集群方面，Swarm将一个或多个Docker节点组织起来，使得用户能够以集群方式管理它们。Swarm默认内置有加密的分布式集群存储（encrypted distributed cluster store）、加密网络（Encrypted Network）、公用TLS（Mutual TLS）、安全集群接入令牌Secure Cluster Join Token）以及一套简化数字证书管理的PKI（Public Key Infrastructure）。用户可以自如地添加或删除节点，这非常棒！编排方面，Swarm提供了一套丰富的API使得部署和管理复杂的微服务应用变得易如反掌。通过将应用定义在声明式配置文件中，就可以使用原生的Docker命令完成部署。此外，甚至还可以执行滚动升级、回滚以及扩缩容操作，同样基于简单的命令即可完成。

2. docker swarm功能类似与k8s

3. 激活swarm，有两个方法：

   - 初始化一个swarm集群，自己成为manager
   - 加入一个已经存在的swarm集群

4. docker swarm init 主要是PKI和安全相关的自动化

   - 创建swarm集群的根证书
   - manager节点的证书
   - 其它节点加入集群需要的tokens

   创建Raft数据库用于存储证书，配置，密码等数据

5. docker swarm leave离开swarm集群

6. docker swarm leave --force强制离开

7. docker service create 创建服务，服务包含多个容器

8. 从集群角度来说，一个Swarm由一个或多个Docker节点组成。这些节点可以是物理服务器、虚拟机、树莓派（Raspberry Pi）或云实例。唯一的前提就是要求所有节点通过可靠的网络相连。节点会被配置为管理节点（Manager）或工作节点（Worker）。管理节点负责集群控制面（Control Plane），进行诸如监控集群状态、分发任务至工作节点等操作。工作节点接收来自管理节点的任务并执行。Swarm的配置和状态信息保存在一套位于所有管理节点上的分布式etcd数据库中。

9. 关于应用编排，Swarm中的最小调度单元是服务。它是随Swarm引入的，在API中是一个新的对象元素，它基于容器封装了一些高级特性，是一个更高层次的概念。

10. 当容器被封装在一个服务中时，我们称之为一个任务或一个副本，服务中增加了诸如扩缩容、滚动升级以及简单回滚等特性。

11. 不包含在任何Swarm中的Docker节点，称为运行于单引擎（Single-Engine）模式。一旦被加入Swarm集群，则切换为Swarm模式

12. 在单引擎模式下的Docker主机上运行docker swarm init会将其切换到Swarm模式，并创建一个新的Swarm，将自身设置为Swarm的第一个管理节点。更多的节点可以作为管理节点或工作节点加入进来。这一操作也会将新加入的节点切换为 Swarm模式。

13. docker swarm join-token worker创建worker token

14. docker swarm join-token manager创建manager token

15. 观察MANAGER STATUS一列会发现，3个节点分别显示为“Reachable”或“Leader”。关于主节点稍后很快会介绍到。MANAGER STATUS一列无任何显示的节点是工作节点。注意，mgr2的ID列还显示了一个星号（*），这个星号会告知用户执行docker node ls命令所在的节点。本例中，命令是在mgr2节点执行的。

16. 从技术上来说，Swarm实现了一种主从方式的多管理节点的HA。这意味着，即使你可能—并且应该—有多个管理节点，也总是仅有一个节点处于活动状态。通常处于活动状态的管理节点被称为“主节点”（leader），而主节点也是唯一一个会对Swarm发送控制命令的节点。也就是说，只有主节点才会变更配置，或发送任务到工作节点。如果一个备用（非活动）管理节点接收到了Swarm命令，则它会将其转发给主节点。

17. 一个旧的管理节点重新接入Swarm会自动解密并获得Raft数据库中长时间序列的访问权，这会带来安全隐患。进行备份恢复可能会抹掉最新的Swarm配置。为了规避以上问题，Docker提供了自动锁机制来锁定Swarm，这会强制要求重启的管理节点在提供一个集群解锁码之后才有权从新接入集群。通过在执行docker swarm init命令来创建一个新的Swarm集群时传入--autolock参数可以直接启用锁。然而，前面已经搭建了一个Swarm集群，这时也可以使用docker swarm update命令来启用锁。

18. 所有的服务都会被 Swarm 持续监控—Swarm 会在后台进行轮训检查（Reconciliation Loop），来持续比较服务的实际状态和期望状态是否一致。如果一致，则皆大欢喜，无须任何额外操作；如果不一致，Swarm会使其一致。换句话说，Swarm会一直确保实际状态能够满足期望状态的要求。

19. 使用docker service ls命令可以查看Swarm中所有运行中的服务。

20. 执行docker service ps命令可以查看服务副本列表及各副本的状态。

21. 服务的默认复制模式（Replication Mode）是副本模式（replicated）。这种模式会部署期望数量的服务副本，并尽可能均匀地将各个副本分布在整个集群中。另一种模式是全局模式（global），在这种模式下，每个节点上仅运行一个副本。可以通过给docker service create命令传递--mode global参数来部署一个全局服务。

22. docker service update对服务滚动升级

23. --replias是副本数。

24. docker service scale是扩缩容

25. overlay网络用于容器间通信，还有一个gw_bridge网络用于对外通信

26. 第一是外部如何访问部署运行在swarm集群内的服务，可以称之为 `入方向` 流量，在swarm里我们通过 `ingress` 来解决

27. 第二是部署在swarm集群里的服务，如何对外进行访问，这部分又分为两块:

    - 第一，`东西向流量` ，也就是不同swarm节点上的容器之间如何通信，swarm通过 `overlay` 网络来解决；
    - 第二，`南北向流量` ，也就是swarm集群里的容器如何对外访问，比如互联网，这个是 `Linux bridge + iptables NAT` 来解决的

28. ingress原理是在每个副本上创建一个gw_bridge网络内的实例，外部访问节点，端口转发到这个实例地址，再对实例地址做负载均衡,负载均衡是连载ingress这个默认创建的overlay网络上

29. 通过服务名访问别的服务，会经过dns返回一个vip，通过vip在进行负载均衡访问服务

30. swarm不支持镜像构建，因为已经是生成环境部署了

31. stack文件不指定网络，会默认创建一个overlay网络

