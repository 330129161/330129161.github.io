title: Windows下使用Docker启动常用工具

author: yingu

thumbnail: http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/cs/cs-20.jpg

toc: true 

tags:

  - docker
  - 工具

categories: 

  - [工具,docker] 

date: 2020-05-03 19:12:55

---

**以下配置说明都是基于windows系统**

## Docker常用命令

1. `docker search 镜像名` ：搜索一个指定的镜像。例如`docker search redis`,可能会搜索出多个，但是一般选择第一个，也就是starts数最高的一个。<!--more -->

2. `docker pull 镜像:版本号`: 拉取一个镜像到本地的docker仓库，如果未选择版本号，那么默认使用镜像的最新版本。例如`docker pull redis`默认会拉取名为`redis:latest`的镜像，`docker pull redis:6.0`会拉拉取一个`redis 6.0`版本的镜像。想知道具体有哪些版本的可以去[DockerHub]("https://hub.docker.com")中查看。

3. `docker images`:查看当前容器的所有镜像，如果刚才使用了`docker pull redis`命令来拉取，那么此时我们本地仓库中应该就有一个`redis`镜像。`REPOSITORY`表示资源，`IMAGE_ID`就表示镜像的ID，`TAG`表示镜像的版本，我们也可以将`TAG`修改为我们自己习惯的标签。

4. `docker run -itd -v  宿主机目录:容器目录 -p 宿主机端口:容器端口  --name 容器名 --restart=always 镜像ID` ： 

   ```
   docker run -itd -v /C/redis/redis.conf:/etc/redis/redis.conf -v /C/redis/data:/data -p 6379:6379 --name redis-server /etc/redis/redis.conf --restart=always redis
   ```

   通过`docker run`命令启动一个容器：

   - `-i`  ：以交互模式运行容器，通常与 -t 同时使用。
   - `-t`：为容器重新分配一个伪输入终端，通常与 -i 同时使用
   - `-d` ：后台运行容器，并返回容器ID
   - `-e` ：传递环境变量
   - 上面三个命令可以写到一起也就是上面的`-itd`，当然你也可以`-i -t -d`
   - `-v` ：挂载宿主机的一个目录，上面就是将容器中`/etc/redis/redis.conf`的文件挂载到我本机的C盘下redis目录下的`redis.conf`文件，同理`/C/redis/data:/data`就是将容器中的/data目录挂载到C盘下的`/redis/data`目录中。
   - `-p`：映射一个端口，本机上可以通过映射的端口访问到容器的端口
   - `--name`：为启动的容器起一个名称
   - `--restart=always`：表示是否随docker启动，类似于windows中的一个开机启动。如果未设置那么默认为不随docker启动
   - `IMAGE_ID` ：表示通过指定镜像启动的容器，上面命令中是通过REPOSITORY来启动的，效果跟直接使用`IMAGE_ID` 一样，通过`REPOSITORY:TAG`的方式可以确定一个镜像，上面未指定TAG默认为`redis:latest`。

5. `docker ps `：查看运行中的容器，如果通过`docker run`启动成功后，就有一个名为 redis-server的容器了，`CONTAINER ID`表示容器ID，`IMAGE`表示基于哪一个镜像启动的。

6. `docker ps -a`：查看所有容器

7. `docker exec -it 容器ID /bin/bash`：表示在运行的容器中执行命令

8. `docker cp 容器ID:目录路径 宿主机路径`：将容器中指定目录文件拷贝到宿主机目录中

9. `docker cp 宿主机路径 容器ID:目录路径`：将宿主机目录的文件拷贝到容器中

10. `docker stop 容器ID`：停止一个容器

11. `docker start 容器ID`: 启动一个容器

12. `docker restart 容器ID`：重启一个容器

13. `docker rm 容器ID`：删除一个容器

14. `docker rmi 镜像ID`：删除一个镜像，如果镜像下面有容器，会提示无法删除，如果强制删除添加-f

15. `docker commit -a 提交人 -m 备注 容器ID 仓库名:标签`：将容器提交为一个镜像。例如：`docker commit -m 'commit images' -a 'zhangsan'  9adeb59430256  redis:test`

16. `docker load -i myredis.tar`:将宿主机的`myredis.tar`加载到docker中，加载成功后docker中会多一个刚才导入的镜像

17. `docker save -o es.tar redis:test1 redis:test2`:将docker中的多个镜像打包到宿主机中

18. `docker logs  --since 30m 容器ID`：查询指定容器30分钟内的日志

## Docker启动portainer

`Portainer`是`Docker`的图形化管理工具，提供状态显示面板、应用模板快速部署、容器镜像网络数据卷的基本操作。启动命令：

```csharp
docker run -d -p 9000:9000 -v /var/run/docker.sock:/var/run/docker.sock -v /C/portainer:/data --name portainer-server --restart=always portainer/portainer
```

- `-v /var/run/docker.sock:/var/run/docker.sock`：加上此设置表示本地模式，不加表示远程模式。
- `-v /C/portainer:/data`：将`portainer`的数据目录映射到本机中。

如果是在windows中使用`portainer`,由于`portainer`是在容器中，此时`portainer`想管理宿主机中的`docker`服务，就必须将宿主机的端口暴露出来，让`portainer`访问。

1. 右键docker图标打开setting->General会看到`Expose daemon on tcp://localhost:2375 without TLS`选项，将这个设置勾上即可。
2. 通过http://localhost:9000访问到`portainer`后，如果是第一次访问需要先设置密码，登陆后在`Remote`标签页`EndPoint Url`处填上`docker.for.win.localhost:2375`，`name`自己随便取个名，然后点击`Connect`按钮即可。

## Docker启动redis

```
docker run -itd -v /C/redis/redis.conf:/etc/redis/redis.conf -v /C/redis/data:/data -p 6379:6379 --name redis-server --restart=always redis /etc/redis/redis.conf --appendonly yes
```

- `-v /C/redis/redis.conf:/etc/redis/redis.conf`：将redis的配置映射出来，这样每次修改配置通过本机即可
- `-v /C/redis/data:/data`：将redis中的数据目录映射出来，避免误删容器导致数据丢失。
- `--appendonly yes`：打开redis持久化配置
-  `--restart=always`：随容器启动

## Docker启动nginx

```
docker run -itd -p 80:80 -v /C/nginx/nginx.conf:/etc/nginx/nginx.conf -v /C/nginx/conf.d/default.conf -v /C/nginx/log:/var/log/nginx --name nginx-server nginx
--restart=always
```

- `-v /C/nginx/nginx.conf:/etc/nginx/nginx.conf`：将nginx的配置文件映射出来
- `-v /C/nginx/log:/var/log/nginx`：将日志映射出来
- `--restart=always`：随容器启动

## Docker启动mysql

此处使用5.7版本，8.0版本需要设置加密方式比较麻烦

```
docker run -itd -p 3307:3306 -v /C/mysql/my.cnf:/etc/mysql/my.cnf -v /C/mysql/data:/var/lib/mysql --name mysql-server -e MYSQL_ROOT_PASSWORD=123456  
--restart=always mysql:5.7
```

- `-v /C/mysql/my.cnf:/etc/mysql/my.cnf`：将`mysql`的配置文件映射出来
- `-v /C/mysql/data:/var/lib/mysql`：将`mysql`的数据目录映射出来
- `-e MYSQL_ROOT_PASSWORD=123456`：首次启动时需要指定root密码
- `--restart=always`：随容器启动

## Docker启动rabbitmq

`docker search rabbitmq:3.7.7-management`这种是带控制台的，如果直接使用`rabbitmq`拉取的没有控制台

```
docker run -itd -p 5672:5672 -p 15672:15672 -v --hostname my-rabbitmq -e RABBITMQ_DEFAULT_VHOST=my-vhost -e RABBITMQ_DEFAULT_USER=admin -e  RABBITMQ_DEFAULT_PASS=admin --name rabbitmq-server --restart=always rabbitmq:3.7.7-management
```

- `-p 5672:5672 -p 15672:15672`：1572是`rabbitmq`服务端口，15672是`rabbitmq`控制台端口
- -`-hostname my-rabbitmq` ：主机名
- `-e RABBITMQ_DEFAULT_VHOST=my-vhost `：指定虚拟机
- `-e RABBITMQ_DEFAULT_USER=admin` ：指定用户名
- `-e RABBITMQ_DEFAULT_PASS=admin：`：指定密码
- `--restart=always`：随容器启动

## Docker启动nacos

通过mysql创建 [nacos-db.sql](https://github.com/alibaba/nacos/blob/develop/config/src/main/resources/META-INF/nacos-db.sql) 中的表，由于nacos和mysql都在docker中，配置`MYSQL_SERVICE_HOST`时，使用docker中mysql服务的ip地址。

```
docker run -itd  -e PREFER_HOST_MODE=hostname -v /C/nacos/logs:/home/nacos/logs  -e MODE=standalone  -e SPRING_DATASOURCE_PLATFORM=mysql  -e MYSQL_SERVICE_HOST=172.17.0.3 -e MYSQL_SERVICE_PORT=3306 -e MYSQL_SERVICE_USER=root -e MYSQL_SERVICE_PASSWORD=123456 -e MYSQL_SERVICE_DB_NAME=nacos-config -e MYSQL_SLAVE_SERVICE_HOST=172.17.0.3 -p 8848:8848  --name nacos-server --restart=always  nacos/nacos-server
```

- -e PREFER_HOST_MODE=hostname ：表示支持hostname，默认为ip模式
- -e MODE=standalone：表示单机模式
- -v /C/nacos/logs:/home/nacos/logs：将日志映射出来
- -e MYSQL_SERVICE_*：`MYSQL_SERVICE_`开头的为mysql的配置信息

## Docker启动elasticsearch

```
docker run -itd --restart=always --name elasticsearch-server -v  /C/elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml -v /C/elasticsearch/data:/usr/share/elasticsearch/data -v  /C/elasticsearch/logs:/usr/share/elasticsearch/logs -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" elasticsearch
```

