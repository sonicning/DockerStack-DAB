Docker Stacks and Distributed Application Bundles(DAB )是Docker1.12和Docker Compose 1.8之后，和swarm模式，Nodes和Services一起引入到Docker Engine API的实验性功能特性。

一个Dockerfile可以构建成一个image，并且可以从该image创建容器。 类似地，一个docker-compose.yml可以构建成一个DAB，并且可以从该DAB创建stacks。 在这个意义上，DAB是一种多服务的可分发image格式。

从Docker 1.12和Compose 1.8开始，这些功能都是实验性的。 Docker Engine和Docker Registry都不支持bundle的分发。

## 实践
理论先放放，直接上干货，下面以部署Docker币的应用为例动手操作，有了感性认识后，我们再来结合实际分析理解。
### 测试环境
**Docker Host 3台 (CentOS 7.3.1611)**
```bash
[root@SwarmManager ~]# lsb_release -d
Description:	CentOS Linux release 7.3.1611 (Core)

[root@SwarmNode1 ~]# lsb_release -d
Description:	CentOS Linux release 7.3.1611 (Core)

[root@SwarmNode2 ~]# lsb_release -d
Description:	CentOS Linux release 7.3.1611 (Core)
```
<span id="version">**Docker Version 3台一致1.13.0**</span>
```bash
[root@SwarmManager ~]# docker version
Client:
 Version:      1.13.0
 API version:  1.25
 Go version:   go1.7.3
 Git commit:   49bf474
 Built:        Tue Jan 17 09:55:28 2017
 OS/Arch:      linux/amd64

Server:
 Version:      1.13.0
 API version:  1.25 (minimum version 1.12)
 Go version:   go1.7.3
 Git commit:   49bf474
 Built:        Tue Jan 17 09:55:28 2017
 OS/Arch:      linux/amd64
 Experimental: false
```
**docker-compose版本**
```bash
[root@SwarmManager dockercoin]# docker-compose version
docker-compose version 1.10.0, build 4bd6f1a
docker-py version: 2.0.1
CPython version: 2.7.9
OpenSSL version: OpenSSL 1.0.1t  3 May 2016
```
**Pull Dockercoins images**

鉴于GFW导致的间歇性抽风，建议先在每一个Docker Host上Pull一次，如果多次无法下载，建议使用业界良心的[DaoCloud加速器服务](http://www.daocloud.io/mirror)。
```bash
docker pull jpetazzo/dockercoins_hasher:1465439244
docker pull jpetazzo/dockercoins_rng:1465439244
docker pull jpetazzo/dockercoins_webui:1465439244
docker pull jpetazzo/dockercoins_worker:1465439244
docker pull redis
```
### Swarm 创建
**初始化Swarm Manager节点**
```bash
[root@SwarmManager ~]# docker swarm init --advertise-addr 100.101.21.237
Swarm initialized: current node (p2lbee6ertr9q8w19go10rlhk) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-0gpmq5mmbbzzfdzui4yd4pp6wz1m978k4tu1x7e6yw3kngig3i-9ufh5lezv80223o35b5sml0b5 \
    100.101.21.237:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.

[root@SwarmManager ~]#
```
**加入Worker节点**
```bash
[root@SwarmNode1 ~]# docker swarm join \
>     --token SWMTKN-1-0gpmq5mmbbzzfdzui4yd4pp6wz1m978k4tu1x7e6yw3kngig3i-9ufh5lezv80223o35b5sml0b5 \
>     100.101.21.237:2377
This node joined a swarm as a worker.

[root@SwarmNode2 ~]# docker swarm join \
>     --token SWMTKN-1-0gpmq5mmbbzzfdzui4yd4pp6wz1m978k4tu1x7e6yw3kngig3i-9ufh5lezv80223o35b5sml0b5 \
>     100.101.21.237:2377
This node joined a swarm as a worker.
```
**查看节点信息**（需要在Manager节点上执行）
```bash
[root@SwarmManager ~]# docker node ls
ID                           HOSTNAME      STATUS  AVAILABILITY  MANAGER STATUS
mrkfyn8wu50m2t4kmwufst56l    SwarmNode1    Ready   Active        
p2lbee6ertr9q8w19go10rlhk *  SwarmManager  Ready   Active        Leader
qaj13t4dlbpd4ez4rgbzjadfh    SwarmNode2    Ready   Active        
[root@SwarmManager ~]#
```
**Compose文件**
```bash
[root@SwarmManager dockercoin]# cat docker-compose.yml
version: "3"
services:
  rng:
    image: jpetazzo/dockercoins_rng:1465439244
    ports:
    - "8001:80"
    networks:
    - dockercoins
  hasher:
    image: jpetazzo/dockercoins_hasher:1465439244
    ports:
    - "8002:80"
    networks:
    - dockercoins
  webui:
    image: jpetazzo/dockercoins_webui:1465439244
    ports:
    - "8000:80"
    networks:
    - dockercoins
  redis:
    image: redis:latest
    ports:
    - "6379:6379"
    networks:
    - dockercoins
  worker:
    image: jpetazzo/dockercoins_worker:1465439244
    links:
      - rng
      - hasher
      - redis
    networks:
    - dockercoins
networks:  
  dockercoins:
    external: true


```
>注意，这里redis我们用latest镜像，在默认jpetazzo大神的[docker-compose.yml](https://github.com/jpetazzo/orchestration-workshop/tree/master/dockercoins)文件中是没有暴露redis的端口的，我调试了很久都报错  最后加入了ports - "6379:6379"才成功

```
Redis error { [Error: Redis connection to redis:6379 failed - getaddrinfo ENOTFOUND redis redis:6379]
  code: 'ENOTFOUND',
  errno: 'ENOTFOUND',
  syscall: 'getaddrinfo',
  hostname: 'redis',
  host: 'redis',
  port: 6379 }
```

### 拉取镜像
使用docker-compose pull 命令拉取yml文件中定义的镜像。我们之前已经Pull了，这里仅仅是看到镜像都有Digest信息
```bash
[root@SwarmManager dockercoin]# docker-compose pull
Pulling worker (jpetazzo/dockercoins_worker:1465439244)...
1465439244: Pulling from jpetazzo/dockercoins_worker
Digest: sha256:69e99fdf089f5bbe8fb3bdf1223d5536d63bbeed5e551468f97550f7afede000
Status: Image is up to date for jpetazzo/dockercoins_worker:1465439244
Pulling redis (redis:latest)...
latest: Pulling from library/redis
Digest: sha256:a027a470aa2b9b41cc2539847a97b8a14794ebd0a4c7c5d64e390df6bde56c73
Status: Image is up to date for redis:latest
Pulling hasher (jpetazzo/dockercoins_hasher:1465439244)...
1465439244: Pulling from jpetazzo/dockercoins_hasher
Digest: sha256:6fdc37944c2ebeb929700aa555b68cafe3b399193af13996054b4c34e70890e2
Status: Image is up to date for jpetazzo/dockercoins_hasher:1465439244
Pulling rng (jpetazzo/dockercoins_rng:1465439244)...
1465439244: Pulling from jpetazzo/dockercoins_rng
Digest: sha256:7216c50255d8176cbd857fa0b24270f8d051fc0cde65c44bfddedfac6f085d29
Status: Image is up to date for jpetazzo/dockercoins_rng:1465439244
Pulling webui (jpetazzo/dockercoins_webui:1465439244)...
1465439244: Pulling from jpetazzo/dockercoins_webui
Digest: sha256:d445cb0d43b5cc4058c9abb609da96d2cb3ae49187a7c44baf011906119c1375
Status: Image is up to date for jpetazzo/dockercoins_webui:1465439244
```
### 生成DAB文件
使用docker-compose bundle基于yml文件生成dab文件
```bash
[root@SwarmManager dockercoin]# docker-compose bundle
WARNING: Unsupported top level key 'networks' - ignoring
WARNING: Unsupported key 'links' in services.worker - ignoring
Wrote bundle to dockercoin.dab
```
Compose v2里的network参数和links参数，目前bundle还不支持所以出现警告并忽略。应用部署的时候会自动创建一个overlay network

**查看dab文件**
```bash
[root@SwarmManager dockercoin]# ll
total 8
-rw-r--r--. 1 root root 1100 Jan 25 11:37 dockercoin.dab
-rw-r--r--. 1 root root  514 Jan 25 11:21 docker-compose.yml
[root@SwarmManager dockercoin]# cat dockercoin.dab
{
  "Services": {
    "hasher": {
      "Image": "jpetazzo/dockercoins_hasher@sha256:6fdc37944c2ebeb929700aa555b68cafe3b399193af13996054b4c34e70890e2",
      "Networks": [
        "dockercoins"
      ],
      "Ports": [
        {
          "Port": 80,
          "Protocol": "tcp"
        }
      ]
    },
    "redis": {
      "Image": "redis@sha256:a027a470aa2b9b41cc2539847a97b8a14794ebd0a4c7c5d64e390df6bde56c73",
      "Networks": [
        "dockercoins"
      ],
      "Ports": [
        {
          "Port": 6379,
          "Protocol": "tcp"
        }
      ]
    },
    "rng": {
      "Image": "jpetazzo/dockercoins_rng@sha256:7216c50255d8176cbd857fa0b24270f8d051fc0cde65c44bfddedfac6f085d29",
      "Networks": [
        "dockercoins"
      ],
      "Ports": [
        {
          "Port": 80,
          "Protocol": "tcp"
        }
      ]
    },
    "webui": {
      "Image": "jpetazzo/dockercoins_webui@sha256:d445cb0d43b5cc4058c9abb609da96d2cb3ae49187a7c44baf011906119c1375",
      "Networks": [
        "dockercoins"
      ],
      "Ports": [
        {
          "Port": 80,
          "Protocol": "tcp"
        }
      ]
    },
    "worker": {
      "Image": "jpetazzo/dockercoins_worker@sha256:69e99fdf089f5bbe8fb3bdf1223d5536d63bbeed5e551468f97550f7afede000",
      "Networks": [
        "dockercoins"
      ]
    }
  },
  "Version": "0.1"
}
```
这里对每个服务进行描述，包含了对应的image、端口、网络信息等。
需要注意的是，DAB目前（在Docker 1.12中）仅仅支持Expose的端口，Published的端口会自动分配（从30000端口开始），不支持自定义，不过在后面我们可以手动来修改。

### 部署Stack
使用docker deploy来部署dockercoin这个stack的时候会报错，因为我们使用的不是实验性的docker daemon [参加前文这里](#version)
```bash
[root@SwarmManager dockercoin]# docker deploy dockercoins
only supported with experimental daemon
[root@SwarmManager dockercoin]# docker stack deploy dockercoins --bundle-file ./dockercoin.dab
Loading bundle from ./dockercoin.dab
Creating network dockercoins_dockercoins
Creating service dockercoins_redis
Creating service dockercoins_rng
Creating service dockercoins_webui
Creating service dockercoins_worker
Creating service dockercoins_hasher
```
注意：Stack的名字需要跟DAB文件名保持一致
**查看端口**
```bash
[root@SwarmManager dockercoin]# docker service inspect dockercoins_webui |grep PublishedPort
                    "PublishedPort": 30001,
```
可见在Docker 1.13版本中，这个PublishedPort是可以在yml文件中指定的
这个在Docker 1.12实验版本中，这个端口是自动分配的。
访问30001对应的webui应用,成功
![DockerCoin](./pic/1.PNG)

在Docker 1.13中，可以直接使用docker-compose.yml文件来deploy，先删除已创建的stack，再测试如下：
```bash
[root@SwarmManager dockercoin]# docker stack rm dockercoins
Removing service dockercoins_hasher
Removing service dockercoins_redis
Removing service dockercoins_webui
Removing service dockercoins_rng
Removing service dockercoins_worker
[root@SwarmManager dockercoin]# docker stack deploy --compose-file ./docker-compose.yml dockercoins
Unsupported Compose file version: "2". The only version supported is "3" (or "3.0")
```
由于我们docker-compose.yml文件内是写的version 2，所以有如上报错。现修改yml文件中version为3（仅仅把2改为3）
```bash
[root@SwarmManager dockercoin]# docker stack deploy --compose-file ./docker-compose.yml dockercoins
network "dockercoins" is declared as external, but could not be found. You need to create the network before the stack is deployed (with overlay driver)
```
创建dockercoins这个overlay network，来解决报错
```bash
[root@SwarmManager dockercoin]# docker network create -d overlay dockercoins
zom1i3a6wpgh2zwykm2marfta
[root@SwarmManager dockercoin]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
8ce732dc7e0b        bridge              bridge              local
0902e1fa8f10        docker_gwbridge     bridge              local
zom1i3a6wpgh        dockercoins         overlay             swarm
e5b4473f6039        host                host                local
y5kb6binhjjs        ingress             overlay             swarm
7c260a787d11        none                null                local
```
这样再执行一次,并查看stack信息和service信息
```bash
[root@SwarmManager dockercoin]# docker stack deploy --compose-file ./docker-compose.yml dockercoins
Ignoring unsupported options: links

Creating service dockercoins_rng
Creating service dockercoins_hasher
Creating service dockercoins_webui
Creating service dockercoins_redis
Creating service dockercoins_worker
[root@SwarmManager dockercoin]# docker service ls
ID            NAME                MODE        REPLICAS  IMAGE
8uam2h0eqm6m  dockercoins_worker  replicated  1/1       jpetazzo/dockercoins_worker:1465439244
fexvogttuz10  dockercoins_rng     replicated  1/1       jpetazzo/dockercoins_rng:1465439244
n0lr6zqvxvrd  dockercoins_hasher  replicated  1/1       jpetazzo/dockercoins_hasher:1465439244
w8bdasi2vs64  dockercoins_redis   replicated  1/1       redis:latest
zjo4sciryiv7  dockercoins_webui   replicated  1/1       jpetazzo/dockercoins_webui:1465439244
[root@SwarmManager dockercoin]# docker stack ps dockercoins
ID            NAME                  IMAGE                                   NODE          DESIRED STATE  CURRENT STATE           ERROR  PORTS
u1ndt34nhmta  dockercoins_worker.1  jpetazzo/dockercoins_worker:1465439244  SwarmNode1    Running        Running 21 minutes ago         
l9x8a70uook3  dockercoins_redis.1   redis:latest                            SwarmManager  Running        Running 22 minutes ago         
y1nj8k8tswtw  dockercoins_webui.1   jpetazzo/dockercoins_webui:1465439244   SwarmNode2    Running        Running 22 minutes ago         
hxptm5aw16ci  dockercoins_hasher.1  jpetazzo/dockercoins_hasher:1465439244  SwarmNode1    Running        Running 22 minutes ago         
7bbd7b3udthm  dockercoins_rng.1     jpetazzo/dockercoins_rng:1465439244     SwarmManager  Running        Running 22 minutes ago       
```
四个service已经被成功创建并运行，REPLICAS是这个service下面有多少个正在运行的副本，可以简单理解为容器，严格意义上应该是Task。dockercoin这个stack有4个容器在运行，NAME为容器的名字，NODE为容器所在的host。
>注意，以上演示表明，在Docker1.13中，就可以直接使用stack来deploy新版本的docker-compose.yml文件，就不需要像Docker1.12实验版本中一样，通过docker-compose把对应的yml文件转换成为dab文件了，也就是说，在stack的部署上，我们完全可以不需要Docker-compose了

### 访问应用
http://100.101.21.237:8000/

![DockerCoins-8000](./pic/2.PNG)

### 服务扩展
使用docker service scale实现服务的在线手动扩展,再查看service和stack信息
```bash
[root@SwarmManager dockercoin]# docker service scale dockercoins_worker=2
dockercoins_worker scaled to 2
[root@SwarmManager dockercoin]# docker service ls
ID            NAME                MODE        REPLICAS  IMAGE
8uam2h0eqm6m  dockercoins_worker  replicated  2/2       jpetazzo/dockercoins_worker:1465439244
fexvogttuz10  dockercoins_rng     replicated  1/1       jpetazzo/dockercoins_rng:1465439244
n0lr6zqvxvrd  dockercoins_hasher  replicated  1/1       jpetazzo/dockercoins_hasher:1465439244
w8bdasi2vs64  dockercoins_redis   replicated  1/1       redis:latest
zjo4sciryiv7  dockercoins_webui   replicated  1/1       jpetazzo/dockercoins_webui:1465439244
[root@SwarmManager dockercoin]# docker stack ps dockercoins
ID            NAME                  IMAGE                                   NODE          DESIRED STATE  CURRENT STATE                   ERROR  PORTS
u1ndt34nhmta  dockercoins_worker.1  jpetazzo/dockercoins_worker:1465439244  SwarmNode1    Running        Running 27 minutes ago                 
l9x8a70uook3  dockercoins_redis.1   redis:latest                            SwarmManager  Running        Running 27 minutes ago                 
y1nj8k8tswtw  dockercoins_webui.1   jpetazzo/dockercoins_webui:1465439244   SwarmNode2    Running        Running 27 minutes ago                 
hxptm5aw16ci  dockercoins_hasher.1  jpetazzo/dockercoins_hasher:1465439244  SwarmNode1    Running        Running 27 minutes ago                 
7bbd7b3udthm  dockercoins_rng.1     jpetazzo/dockercoins_rng:1465439244     SwarmManager  Running        Running 27 minutes ago                 
57r5coa7zpxv  dockercoins_worker.2  jpetazzo/dockercoins_worker:1465439244  SwarmNode2    Running        Running less than a second ago  
```
可见名为dockercoins_worker.2的容器作为worker服务的第二个副本在SwarmNode2上运行
**访问测试**
![DockerCoin-2Workers](./pic/3.PNG)

经过服务扩展后，每秒生成的Docker币由4个变为了8个
我又试着把worker变成4个，之后又试着变成1个
```bash
[root@SwarmManager dockercoin]# docker service scale dockercoins_worker=4
dockercoins_worker scaled to 4
[root@SwarmManager dockercoin]# docker stack ps dockercoins
ID            NAME                  IMAGE                                   NODE          DESIRED STATE  CURRENT STATE                   ERROR  PORTS
u1ndt34nhmta  dockercoins_worker.1  jpetazzo/dockercoins_worker:1465439244  SwarmNode1    Running        Running 31 minutes ago                 
l9x8a70uook3  dockercoins_redis.1   redis:latest                            SwarmManager  Running        Running 32 minutes ago                 
y1nj8k8tswtw  dockercoins_webui.1   jpetazzo/dockercoins_webui:1465439244   SwarmNode2    Running        Running 32 minutes ago                 
hxptm5aw16ci  dockercoins_hasher.1  jpetazzo/dockercoins_hasher:1465439244  SwarmNode1    Running        Running 32 minutes ago                 
7bbd7b3udthm  dockercoins_rng.1     jpetazzo/dockercoins_rng:1465439244     SwarmManager  Running        Running 32 minutes ago                 
57r5coa7zpxv  dockercoins_worker.2  jpetazzo/dockercoins_worker:1465439244  SwarmNode2    Running        Running 4 minutes ago                  
rkdufguu0clk  dockercoins_worker.3  jpetazzo/dockercoins_worker:1465439244  SwarmNode2    Running        Running less than a second ago         
dwbe1lh3em1l  dockercoins_worker.4  jpetazzo/dockercoins_worker:1465439244  SwarmManager  Running        Running 3 seconds ago                  
[root@SwarmManager dockercoin]# docker service scale dockercoins_worker=1
dockercoins_worker scaled to 1
```
![DockerCoin-4to1Workers](./pic/4.PNG)

### 总结
我们基本上已经完成Docker Stack和DAB的初探，也对比了Docker 1.12和Docker1.13的差异
在Docker1.13中Stack功能是吸引大家去升(tian)级(keng)到新版的Docker Engine了
DAB和Docker-compose v3都还有很多地方需要完善，比如volume的支持等
Docker越来越完善的生态链和工具链，已经把容器编排做得越来越简单了（相对于k8s来说）
未来会怎么样呢？让我们拭目以待
