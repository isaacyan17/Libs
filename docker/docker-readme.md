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
