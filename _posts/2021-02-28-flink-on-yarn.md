---
title: flink on yarn
description: flink on yarn
categories:
  - big-data
tags:
  - flink
---

## flink执行应用的三种模式

flink可通过如下三种模式执行应用：应用模式，单任务模式和会话模式，它们的不同在于：1，集群的生命周期和资源的隔离保障；2，应用主函数是在客户端执行，还是在集群上执行。

应用模式下，每次提交应用都会创建一个集群，并且应用的主函数在集群的任务管理器中执行，相反，在单任务模式和会话模式下，应用主函数在客户端执行，客户端需要首先下载应用依赖到本地，并执行主函数从而获取任务图，然后将两者提交到集群。应用模式下，集群只会在单个应用的任务间共享，应用完成后集群马上被销毁。在单个应用的粒度上，应用模式和单任务模式提供同样的资源隔离和负载均衡。由于在任务管理器上面执行主函数，从而可以节约cpu及网络带宽。需要注意，在应用模式下，注册的路径需要保障任务管理器可以正常访问。

为了提供更好的资源隔离保障，单任务模式使用资源管理框架，如yarn，k8s等为每一个提交的任务提供资源，集群只对单个任务有效。任务完成后，集群将会被销毁，资源被释放。这样，异常的任务只会影响自己，另外，它还将任务记账的压力分散到多个任务管理器上，因为每个任务都有一个自己的任务管理器。

会话模式假设已经存在正在运行的集群，然后使用集群中的资源执行提交的应用，所有应用在同一个集群中执行，并竞争资源。该模式下，用户不用担心运行每个任务的集群的资源压力，但是，如果某个任务出问题将会影响到集群上所有其他任务，另外，任务管理器也会又更大的记账压力。

## 通过yarn资源管理器执行上面的模式

应用模式将在yarn上面启动一个集群，然后在集群的任务管理器中执行应用的主函数，启动应用：./bin/flink run-application -t yarn-application ./examples/streaming/TopSpeedWindowing.jar；获取任务列表：./bin/flink list -t yarn-application -Dyarn.application.id=application_XXXX_YY；取消任务：./bin/flink cancel -t yarn-application -Dyarn.application.id=application_XXXX_YY <jobId>；通过yarn杀死任务：yarn application -kill <ApplicationId>；使用yarn.provided.lib.dirs启动应用：./bin/flink run-application -t yarn-application -Dyarn.provided.lib.dirs="hdfs://myhdfs/my-remote-flink-dist-dir" hdfs://myhdfs/jars/my-application.jar。

单任务模式首先在yarn上启动一个flink集群，并在本地运行应用的jar包，最后提交任务图到集群上的任务管理器。任务结束后，集群也同时停止。提交任务：./bin/flink run -t yarn-per-job --detached ./examples/streaming/TopSpeedWindowing.jar；任务列表：./bin/flink list -t yarn-per-job -Dyarn.application.id=application_XXXX_YY；取消任务：./bin/flink cancel -t yarn-per-job -Dyarn.application.id=application_XXXX_YY <jobId>。

会话模式需首先启动flink集群，export HADOOP_CLASSPATH=`hadoop classpath`;./bin/yarn-session.sh --detached;./bin/flink run ./examples/streaming/TopSpeedWindowing.jar;echo "stop" \| ./bin/yarn-session.sh -id application_XXXXX_XXX;会话模式会创建一个隐藏文件： /tmp/.yarn-properties-<username>，后续提交任务会通过该文件获取集群属性，当然也可手工指定：./bin/flink run -t yarn-session -Dyarn.application.id=application_XXXX_YY ./examples/streaming/TopSpeedWindowing.jar

## faq

在wsl中通过：sbin/start-dfs.sh启动hdfs时，出现： java.net.SocketException: Permission denied异常，服务使用50070的默认端口，并非<1024的限制端口，使用administrator运行wsl一样不行。原来windows平台上面的Hyper-V禁用了该端口，可以通过：netsh interface ipv4 show excludedportrange protocol=tcp进行查看，发现50070被限制了：50060-500159。一方面可以通过修改：hdfs-site.xml中的属性：dfs.http.address，指定非限制端口，如：5070；另外可以通过如下操作取消限制：1，禁用Hyper-V，dism.exe /Online /Disable-Feature:Microsoft-Hyper-V；2，重启并增加端口，netsh int ipv4 add excludedportrange protocol=tcp startport=50070 numberofports=1；3启用Hyper-V，dism.exe /Online /Enable-Feature:Microsoft-Hyper-V /All

如果通过wsl在移动硬盘上面进行操作，则需要首先将移动硬盘挂载到wsl中，mkdir /mnt/f，sudo mount -t drvfs f: /mnt/f -o metadata，注意，只有添加了- o参数，wsl中才可以通过chmod修改文件权限，否则，chmod不生效。如果挂载中出现device is busy之类得错误，则lsof \| grep /mnt/f; kill -9 ; umount /mnt/f等操作，删除占用设备得进程，重新mount。

可通过hdfs dfs -getfacl /查看目录权限，hdfs dfs -setfacl -m user:root:rwx /设置目录权限。

## ref:

- [hadoop yarn](https://hadoop.apache.org/docs/current/hadoop-yarn/hadoop-yarn-site/YARN.html)
- [flink cluster on yarn](https://ci.apache.org/projects/flink/flink-docs-release-1.12/deployment/resource-providers/yarn.html)
