## docker



1. [参考课程的笔记](https://dockertips.readthedocs.io/en/latest/)

2. 运行容器默认是attached模式，容器的输入输出会显示在终端，终端的输入输出也会影响到容器，即ctrl+c。但windows中不会，即ctrl+c没有影响到容器

3. -d则是detached模式运行，在后台运行。

4. ```shell
   #批量操作，例如批量停止
   docker container stop $(docker container ps -aq)
   ```

5. docker跟容器（container）相关的命令，可以省略container，如下图所示：

   <img src="E:\study\kittynote\docker\images\docker省略.png" alt="省" style="zoom:80%;" />

6. 镜像：Docker image是一个 `read-only` 文件。这个文件包含文件系统，源码，库文件，依赖，工具等一些运行application所需要的文件。可以理解成一个模板。docker image具有分层的概念

7. “一个运行中的docker image”。实质是复制image并在image最上层加上一层 `read-write` 的层 （称之为 `container layer` ,容器层）。基于同一个image可以创建多个container

8. 