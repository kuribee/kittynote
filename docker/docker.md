## docker



1. [参考课程的笔记](https://dockertips.readthedocs.io/en/latest/)

2. 镜像：Docker image是一个 `read-only` 文件。这个文件包含文件系统，源码，库文件，依赖，工具等一些运行application所需要的文件。可以理解成一个模板。docker image具有分层的概念

3. 容器：“一个运行中的docker image”。实质是复制image并在image最上层加上一层 `read-write` 的层 （称之为 `container layer` ,容器层）。基于同一个image可以创建多个container

4. docker + 管理的对象（比如容器，镜像） + 具体操作（比如创建，启动，停止，删除）

5. 运行容器默认是attached模式，容器的输入输出会显示在终端，终端的输入输出也会影响到容器，即ctrl+c。但windows中不会，即ctrl+c没有影响到容器

6. -d则是detached模式运行，在后台运行。可以使用docker attach，连接到容器的主进程的 STDIN / STDOUT，但此时终端的输入输出会影响到容器（windows除外）。因为docker attach不是新起进程，而是直接“接管”容器启动时的那个进程。如果主进程退出，容器就停了。`Ctrl + C` 可能直接把容器干掉

7. docker exec 是在正在运行的容器里，新起一个进程。不影响容器主进程。退出 shell 容器还在。适合“进去看看 / 调试”

8. ```shell
   #批量操作，例如批量停止
   docker container stop $(docker container ps -aq)
   ```

9. docker跟容器（container）相关的命令，可以省略container，如下图所示：

   <img src=".\images\docker省略.png" alt="省" style="zoom:80%;" />

10. 