# RocketMQ
## Docker安装RocketMQ

首先确定已经装好了docker。

- 拉取镜像

```shell
docker pull rocketmqinc/rocketmq
```

- 启动namesrv服务

```shell
docker run -d -p 9876:9876 -v {RmHome}/data/namesrv/logs:/root/logs -v {RmHome}/data/namesrv/store:/root/store --name rmqnamesrv -e "MAX_POSSIBLE_HEAP=100000000" rocketmqinc/rocketmq sh mqnamesrv
```

其中，**{RmHome}** 要替换成你的宿主机想保存 MQ 的**日志与数据**的地方，通过 docker 的 -v 参数使用 volume 功能，把你本地的**目录映射到容器内的目录**上。否则所有数据都默认保存在容器运行时的内存中，重启之后就又回到最初的起点。

- 启动broker服务

  启动之前，需要在{RmHome}/conf 目录下创建broker.conf文件，内容如下：

  ```properties
  brokerClusterName = DefaultCluster
  brokerName = broker-a
  brokerId = 0
  deleteWhen = 04
  fileReservedTime = 48
  brokerRole = ASYNC_MASTER
  flushDiskType = ASYNC_FLUSH
  brokerIP1 = 119.3.133.125
  ```

启动服务

```shell
docker run -d -p 10911:10911 -p 10909:10909 -v  {RmHome}/data/broker/logs:/root/logs -v  {RmHome}/rocketmq/data/broker/store:/root/store -v  {RmHome}/conf/broker.conf:/opt/rocketmq/conf/broker.conf --name rmqbroker --link rmqnamesrv:namesrv -e "NAMESRV_ADDR=namesrv:9876" -e "MAX_POSSIBLE_HEAP=200000000" rocketmqinc/rocketmq sh mqbroker -c /opt/rocketmq/conf/broker.conf
```

至此，namesrv和broker已经启动完毕，运行docker ps查看

- 安装控制台

  

  ```shell
  docker pull styletang/rocketmq-console-ng
  ```

启动控制台容器

```shell
docker run -e "JAVA_OPTS=-Drocketmq.config.namesrvAddr={docker宿主机ip}:9876 -Drocketmq.config.isVIPChannel=false" -p 8080:8080 -t styletang/rocketmq-console-ng
```

注意java_opts参数因不同的console系统内容可能不同，可以结合console系统中的配置文件做相应修改

- 启动控制台

在浏览器中输入：[http://{docker宿主机ip}:8080/](http://10.1.7.105:8080/) 