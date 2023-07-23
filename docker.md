### 一、镜像操作

1. docker search  (检索镜像的详细信息)

   eg: docker search mysql

2. docker pull    拉取镜像

   eg: docker pull mysql:5.6

3. docker images    查看本地所有镜像

4. docker rmi iamges-id     删除镜像

   eg：docker rmi cf6527af4ce6

5. docker build     构建镜像

6. docker push     推送镜像到服务

7. docker save     保存镜像为一个压缩包

8. docker load    加载镜像为压缩包

9. systemctl start docker     启动docker（stop/restart  关闭/重启docker）

### 二、容器操作

**docker run    创建并运行**

1. docker run --name [自定义名字] -d [指定镜像模板] 

   **创建并运行**指定容器，–name代表自定义容器名字，-d代表后台运行

   eg:  docker run --name mynginx -d nginx/cf6527af4ce6

2. docker ps [-a]    查看运行中的容器，加上-a代表查看所有容器

3. docker stop [容器名称/容器id]  停止当前运行的容器

4. docker start [容器名称/容器id]  启动指定的容器

5.  docker rm 容器id    删除容器（不能删除正在运行中的容器除非加上-f参数）

6. docker run -p [主机端口:容器内部端口] --name [自定义名字] -d [指定镜像模板]

   将容器内部端口映射到主机上去，可以让外网进行访问

   docker run --name r1 -p 80:80 -d nginx

7. docker logs [容器名称/容器id]    查看容器日志

8. docker exec    进入容器内部执行命令

9. docker pause/unpause    暂停或继续容器的运行

**docker exec -it r1 bash**

**docker exec:  进入容器内部，执行命令**

**-it：给当前进入的容器创建一个标准输入输出终端，允许我们与容器进行交互**

**r1：要进入容器的名称**

**bash：进入容器后执行的命令，bash是一个Linux终端交互命令**

### 三、常用命令

1. docker xx --help    查看xx命令的语法

   docke save --help

2. docker save -o nginx.tar nginx:latest

   将nginx:latest保存为nginx.tar

3. docker load -i nginx.tar

   将nginx.tar加载为本地镜像

### 四、数据卷

**数据卷（volume）**是一个虚拟目录，指向宿主机文件系统中的某个目录

文件位置 /var/lib/docker/volumes/

/var与/root在同一级目录，均在根目录(/)下

**1.数据卷操作基本语法**

docker volume [command]

docker volume creat    创建一个新的数据卷

docker volume inspect    显示一个或多个volume信息

docker volume ls    列出所有的volume信息

docker volume prune   删除未使用的volume

docker volume rm    删除一个或者多个指定的volume

**2.挂载数据卷**

```bash
docker run \
--name r1 \
-v html:/root/html \
-p 8080:80 \
nginx
```

docker run：创建并运行容器

--name r1：给容器起名为r1

-v html:/root/html：把html数据卷挂载到容器内的/root/html这个目录中

-p 8080:80：把宿主机的8080端口映射到容器内的80端口

nginx：镜像名称

**-v volumeName:/targetContainerPath**

**如果容器运行时volume不存在，会自动创建volume**

**3.目录挂载**（与数据卷挂载语法类似）

-v [宿主机目录:容器内目录]

-v [宿主机文件:容器内文件]

### 五、自定义镜像dockerfile

镜像结构：镜像是将应用程序及其需要的系统函数库、环境、配置、依赖打包而成

镜像是分层结构，每一层成为一个Layer

baseImage：包含基本的系统函数库、环境变量、文件系统

Entrypoint：入口，是镜像启动应用的命令

其他：在BaseImage基础上添加依赖、安装程序、完成整个应用的安装和配置

#### 什么是dockerfile

**dockerfile**是一个文本文件，其中包含一个个指令，用指令来说明要执行什么操作来构建镜像。一个指令会形成一层Layer

FROM：指定基础镜像    FROM centos:6

ENV：设置环境变量，可在后面指令适用    ENV key value

COPY：拷贝本地文件到镜像的指定目录    COPY ./mysql-5.7.rpm /tmp

RUN：执行Linux的shell命令，一般是安装过程的命令    RUN yum install gcc

EXPOSE：指定容器运行时监听的端口，是给镜像使用者看的    EXPOSE 8080

ENTRYPOINT：镜像中应用的启动命令，容器运行时调用    ENTRYPOINT java -jar xx.jar

#### 是什么DockerCompose

**Docker Compose**可以基于Compose文件帮我们快速的部署分布式应用，而无需手动一个个创建和运行容器

**Compose**文件是一个文本文件，通过指令定义集群中的每个容器如何运行（相当于多个docker run的集合）

docker run

```she
docker run \
--name mysql \
-e MYSQL_ROOT_PASSWORD=root \
-p 3306:3306 \
-v /tmp/mysql/conf/hmy.cnf:/etc/mysql/conf.d/hmy.cnf \
-v /tmp/mysql/data:/var/lib/mysql \
-d \
mysql:5.7.25
```

compose

```yaml
version: "3.8"

service:
  mysql:
    image: 5.7.25
    environment:
    MYSQL_ROOT_PASSWORD=root
    volumes:
      - /tmp/mysql/data:/var/lib/mysql \
      - /tmp/mysql/conf/hmy.cnf:/etc/mysql/conf.d/hmy.cnf
    web:
      build: . # '.'表示从当前目录构建镜像
      ports:
        - 8090: 8090
```





