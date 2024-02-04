## Docker

### 常用命令
* 查看所有容器 `docker ps -a`

### nginx

#### 下载镜像
#### 处理外部挂载的配置文件

```
1. 先创建外部挂载的文件
如ngix.conf
mkdir -p /[your root]/nigx/conf/nginx.conf
mkdir -p /[your root]/nigx/logs
mkdir -p /[your root]/nigx/html
mkdir -p /[your root]/nigx/conf.d

2. 启动一个容器
 docker run --name nginx -p 3001:80 -d nginx
 
 3.拷贝nginx容器对应的文件默认配置
docker cp nginx:/etc/nginx/nginx.conf /opt/docker/nginx/conf/nginx.conf
docker cp nginx:/etc/nginx/conf.d /opt/docker/nginx/conf.d
docker cp nginx:/usr/share/nginx/html /opt/docker/nginx

4. 删除临时容器
  docker stop ngix 
  docker rm ngix 
```

#### 创建容器并运行

```
docker run \
-p 3001:80 \
--name nginx \
-v /home/nginx/conf/nginx.conf:/etc/nginx/nginx.conf \
-v /home/nginx/conf.d:/etc/nginx/conf.d \
-v /home/nginx/logs:/var/log/nginx \
-v /home/nginx/html:/usr/share/nginx/html \
-d nginx:latest
```

### MySql

```
docker run \
-p 3307:3306 \
--name isaac_sql \
-e TZ=Asia/Shanghai \
-e MYSQL_ROOT_PASSWORD=[密码] \
-d mysql \
--character-set-server=utf8mb4 \
--collation-server=utf8mb4_unicode_ci 
```

### Spring 

### 打包项目生成jar包

```
1. maven -> lifecycle-> clean
2. maven -> lifecycle-> package
```

### dockerfile

```
FROM adoptopenjdk/openjdk11
COPY babywhat-0.0.1-SNAPSHOT.jar bb.jar
RUN bash -c "touch /bb.jar"
EXPOSE 8088
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
RUN echo 'Asia/Shanghai' > /etc/timezone
ENTRYPOINT ["java", "-jar","bb.jar"]
```

将dockerfile和jar包放在同一目录下
### build dockerfile构建一个镜像

```
docker build -t [镜像名称] -f [dockerfile路径] .

docker build -t appserver -f ./appservice.dockerfile .
# -f 是找到本地路径下面的dockerfile文件。如果不是默认DockerFile.dockerfile命名的文件则必须要-f 
```

### 启动容器

```
docker run -d --name [容器名称] -p 8088:8088 [镜像名称]
# -p 8090:8090  宿主机端口:容器内部端口
```