## Docker

### 常用命令
* 查看所有容器 `docker ps -a`

* 本地下载镜像并上传至ecs

  ```
  docker run [image]
  docker save [image] -o [/path/xxx.tar]
  上传ecs
  docker load -i [xxx.tar] 
  ```


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
docker cp nginx:/etc/ssl /home/nginx/  (ssl证书配置)

4. 删除临时容器
  docker stop ngix 
  docker rm ngix 
```

#### 创建容器并运行

```
docker run \
-p 80:80 \
--name nginx \
-v /home/nginx/conf/nginx.conf:/etc/nginx/nginx.conf \
-v /home/nginx/conf.d:/etc/nginx/conf.d \
-v /home/nginx/logs:/var/log/nginx \
-v /home/nginx/html:/usr/share/nginx/html \
-v /home/nginx/ssl:/etc/ssl \ 
-d nginx:latest
```

#### 修改Nginx配置

```
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    include /etc/nginx/conf.d/*.conf;

    server {
        listen     80;
        server_name       localhost;

        location / {
            proxy_pass  http://127.0.0.1:3030;
        }
    }
}

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

#### 打包项目生成jar包

```
1. maven -> lifecycle-> clean
2. maven -> lifecycle-> package
```

#### dockerfile

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
#### build dockerfile构建一个镜像

```
docker build -t [镜像名称] -f [dockerfile路径] .

docker build -t appserver -f ./appservice.dockerfile .
# -f 是找到本地路径下面的dockerfile文件。如果不是默认DockerFile.dockerfile命名的文件则必须要-f 
```

#### 启动容器

```
docker run -d --name [容器名称] -p 8088:8088 [镜像名称]
# -p 8090:8090  宿主机端口:容器内部端口
```

### Redis

> https://www.cnblogs.com/virtulreal/p/16228206.html


```
docker run \
--name redis \
--restart unless-stopped \
-p 6379:6379 \
-p 8011:8001 \
-e REDIS_ARGS="--requirepass test" \
-d redis
```


### Next

[通过nextjs提供的dockerfile进行部署](https://www.nextjs.cn/docs/deployment)

```
FROM node:18-alpine AS base

# Install dependencies only when needed
FROM base AS deps
# Check https://github.com/nodejs/docker-node/tree/b4117f9333da4138b03a546ec926ef50a31506c3#nodealpine to understand why libc6-compat might be needed.
RUN apk add --no-cache libc6-compat
WORKDIR /app

# Install dependencies based on the preferred package manager
COPY package.json yarn.lock* package-lock.json* pnpm-lock.yaml* ./
RUN \
  if [ -f yarn.lock ]; then yarn --frozen-lockfile; \
  elif [ -f package-lock.json ]; then npm ci; \
  elif [ -f pnpm-lock.yaml ]; then corepack enable pnpm && pnpm i --frozen-lockfile; \
  else echo "Lockfile not found." && exit 1; \
  fi


# Rebuild the source code only when needed
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Next.js collects completely anonymous telemetry data about general usage.
# Learn more here: https://nextjs.org/telemetry
# Uncomment the following line in case you want to disable telemetry during the build.
# ENV NEXT_TELEMETRY_DISABLED 1

RUN \
  if [ -f yarn.lock ]; then yarn run build; \
  elif [ -f package-lock.json ]; then npm run build; \
  elif [ -f pnpm-lock.yaml ]; then corepack enable pnpm && pnpm run build; \
  else echo "Lockfile not found." && exit 1; \
  fi

# Production image, copy all the files and run next
FROM base AS runner
WORKDIR /app

ENV NODE_ENV production
# Uncomment the following line in case you want to disable telemetry during runtime.
# ENV NEXT_TELEMETRY_DISABLED 1

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public

# Set the correct permission for prerender cache
RUN mkdir .next
RUN chown nextjs:nodejs .next

# Automatically leverage output traces to reduce image size
# https://nextjs.org/docs/advanced-features/output-file-tracing
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs

EXPOSE 3030

ENV PORT 3030
# set hostname to localhost
ENV HOSTNAME "0.0.0.0"

# server.js is created by next build from the standalone output
# https://nextjs.org/docs/pages/api-reference/next-config-js/output
CMD ["node", "server.js"]

```