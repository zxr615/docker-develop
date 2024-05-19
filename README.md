# 手摸手带你使用 docker-compose 编排一个开发环境（Nginx + Mysql + Redis + PHP-FPM + Laravel）

## 介绍

在最近一份工作之前，我一直都是使用在本地安装开发环境的模式开发，一直也没遇到过什么问题，虽然有尝试换成 docker，但好像换了也没什么区别，所以还是一直保持着本地开发。直到换了一家公司后，问题逐渐的显露出来。

最大的问题就是项目的 PHP 版本不一致，有新有旧，有用 PHP7.2 、 PHP8.0、PHP8.1，扩展也不一样，使用 pecl 安装扩展，如在 7.2 安装了 Redis 扩展，在 8.0 可能就安装不上了，提示已经安装过，当然也是能处理，但也是烦不胜烦。这时候的 docker 就能解决我实际的问题了，优秀的环境隔离性，让我有了动力换成 docker 开发环境。

本文目标是使用本地搭建的流程，使用最简单的配置，渐进地把环境搭建起来，实现小且够用的环境，再根据自己的需求调整。调优的配置则不会涉及，毕竟能力有限🐶。

搭建的过程是遇到问题，提出问题，解决问题，而不是直接告诉你为什么需要这个，为什么需要那个，这个是怎么来的，都会描述的清楚。

由于我也是边学边写，所以文章也一定有很多地方错漏，不合理、错误的地方，请大家多多指点。

## 代码

其实涉及的代码量确实不大，文章写的似乎有些啰嗦，结合成品代码看的话应该会好一些。

仓库：[https://github.com/zxr615/docker-develop](https://github.com/zxr615/docker-develop)

目录树：

```console
.
├── data
│   ├── mysql
│   └── redis
├── docker-compose.yml
├── mysql
│   ├── Dockerfile
│   └── my.cnf
├── nginx
│   ├── Dockerfile
│   ├── conf.d
│   │   └── laravel-app.conf
│   └── nginx.conf
├── php83
│   ├── Dockerfile
│   ├── php.ini
│   └── www.conf
├── redis
│   ├── Dockerfile
│   └── redis.conf
└── www
```

## 本地模式

> 使用 Mac 系统演示，其实与 Windows、WSL 流程大差不差，软件包的命令不同罢了。

让我们先回顾一下一个请求，需要使用到的软件，这里我们尽可的精简，安装了Nginx、PHP-FPM、MySQL 以及 Redis 四个软件，通常在公司会有 Mysql、Redis 的环境提供，但在这里作为演示，将这两个软件也安装上了。

`浏览器 -> Nginx -> PHP-FPM -> Code(MySQL、Redis...)`

了解了需要的软件后，我们现在开始安装：

> brew 是一个在 Mac 上用的包管理工具，如 Ubuntu 的 apt-get

```console
brew install nginx@1.25
brew install php@8.2
brew install mysql@8.4
brew install redis@7.2
```

经过一系列的配置后，我们就可以正常运行项目了。

## 前置储备

本文不会介绍 Docker 的基本概念、安装方法和基础指令等，只会提供具体的实践步骤。

在搭建过程中，我们最常访问的三个网站是：

[Docker Hub](https://hub.docker.com/) 用到的软件容器镜像仓库，如：Nginx、PHP-FPM、MySQL 等。

[Docker — 从入门到实践](https://yeasy.gitbook.io/docker_practice) 入门文章，也可以当做手册使用，命令不记得了，就搜索一下回忆回忆。

[Docker 镜像名称都是写什么意思](https://runtuchigua.cn/archives/812/) 在 Docker Hub 中会提供非常多不同后缀的镜像名，可以**先阅读**一下本片文章。

## 开始搭建

上述的请求过程是：`浏览器 -> Nginx -> PHP-FPM -> Code(MySQL、Redis...)`，我们调整一下顺序，将搭建的顺序改成这样：

1. Nginx 一个请求的开始，所以必须是它第一。
2. Mysql 由于 Mysql 是被程序代码所依赖的，所以我们在 PHP-FPM 之前安装好。
3. Redis 同上。
4. PHP-FPM 将代码依赖安装完成之后，再安装 PHP-FPM 进行调试。

涉及的文件全部在 `~/docker-develop` 文件夹下 `mkdir~/docker-develop`

### docker-compose.yml

编排容器的配置文件。

*~/docker-develop/docker-compose.yml*

`mkdir ~/docker-develop && touch ~/docker-develop/docker-compose.yml`

我们先创建一个网络，一会其他的服务都走同时使用这个网络进行通信。

```yaml
networks:
  internal: # 这里的名字随便修改成什么都可以。
```

### Nginx

1. 创建 nginx 目录以及 Dockerfile 文件

    `mkdir ~/docker-develop/nginx && touch ~/docker-develop/nginx/Dockerfile`

2. 我们先去 [Docker Hub](https://hub.docker.com/) 找到 [Nginx](https://hub.docker.com/_/nginx) 镜像，这里我们选用：`nginx:1.25.5-alpine` 这个镜像

3. 修改 Dockerfile

    *~/docker-develop/nginx/Dockerfile*

    ```dockerfile
    FROM nginx:1.25.5-alpine
    ```

4. 修改 docker-compose.yml

    ```yaml
    services:
      nginx:
        build: ./nginx #构建nginx 目录下的 Dockerfile 
        labels: #为容器添加辅助说明信息
          - nginx
        ports: # 暴露端口信息
          - 80:80
          - 443:443
        restart: always # 容器退出后的重启策略为始终重启
        networks: # 我们第一步填写的网络名称
          - internal
    ```

5. 运行 docker-compose

    这个时候其实我们已经可以运行 docker-compose 来启动项目了，只是目前只有 Nginx 一个服务而已，那就让我们来启动看看。

    `docker-compose up --build`

    ```console
    $ docker-compose up --build
    [+] Building 1.6s (5/5) FINISHED                         docker:desktop-linux
     => [nginx internal] load build definition from Dockerfile              0.0s
     => => transferring dockerfile: 62B                                     0.0s
     => [nginx internal] load metadata for docker.io/library/nginx:1.25.5-alpine                                                
     ...
    nginx  | 2024/05/16 10:43:50 [notice] 1#1: start worker process 36
    nginx  | 2024/05/16 10:43:50 [notice] 1#1: start worker process 37
    ```
    请求测试：`curl 127.0.0.1`
    ```html
    <!DOCTYPE html>
      <html>
      <head>
      <title>Welcome to nginx!</title>
      <style>
    ```

6. 配置文件

    光运行起来还不够，我们还需要 nginx 的基础配置文件，来管理 Nginx 的基础配置，还有站点等信息。

    **nginx.conf 配置**

    来自 [Nginx](https://hub.docker.com/_/nginx) 描述，可搜索下面的关键字定位页面位置，我们得知 nginx.conf 文件存在于容器内的 `/etc/nginx/nginx.conf` 目录下，所以我们得把 `nginx.conf` 文件拿出来，放到本地修改使用。
    > Build a new image with your configuration file  
    > COPY nginx.conf /etc/nginx/nginx.conf


    **将容器的 `nginx.conf` 复制到宿主机上**

    本地环境的话，就直接修改配置后再 reload 一下就完事了，但因为容器每次构建都是一个新的容器，配置文件也是新的了，不能够每次都去配置一遍吧？

    那么就要用我们自己的配置文件替换容器内的配置文件，所以这里还要多一步操作，要把默认的配置文件拿到宿主机中，修改之后，重新构建后用我们自己的配置文件替换掉，容器中默认的配置文件。

    ```sh
    # 复制容器里的 nginx.conf 文件到本地
    $ docker-compose cp nginx:/etc/nginx/nginx.conf ~/docker-develop/nginx/nginx.conf
    Successfully copied 2.56kB to ~/docker-develop/nginx/nginx.conf
    ```
    
    除了使用 `docker-compose cp` 命令复制文件出来，我们还可以直接进入容器，找到文件把想要的文件复制出来。
    
    ```sh
    $ docker-compose ps
    NAME      IMAGE                  COMMAND                   ...
    nginx     docker-develop-nginx   "/docker-entrypoint.…"    ...
    
    $ docker-compose exec nginx sh
    / #
    这个时候你就进入了 nginx 这个容器的内部，简单理解就是只装了 nginx 的一个 linux 操作系统，可以执行 cd、cat 等命令，把文件 cat 后复制C-V 复制出来
    ```
    
    **修改 nginx.conf 内容**
    
    *~/docker-develop/nginx/nginx.conf*

    ```nginx
    user  nginx;
        ...
    http {
            include       /etc/nginx/mime.types;
            ... 
            # gzip    on; # 默认是注释的
            gzip      on; # 修改 nginx.conf 将 gzip 打开
            include /etc/nginx/conf.d/*.conf;
        }
    ```
    
    可以看到，还引入了一个配置文件夹，`*.conf` 通常是放各种站点的 server 配置，所以也是要替换容器默认的配置文件的。
    
    我们先创建一个 `conf.d` 的文件夹，用来存放站点信息，先留空，后面讲到 PHP-FPM 的时候再往 `conf.d` 内写入站点文件。

    ```sh
    mkdir ~/docker-develop/nginx/conf.d
    ```

6. 将 nginx 本地的 nginx.conf 覆盖到容器中去

    修改 Dockerfile 文件

    *~/docker-develop/nginx/Dockerfile*

    ```dockerfile
    FROM nginx:1.25.5-alpine
    
    # 将修改后的 nginx.conf 复制到容器中
    COPY ./nginx.conf /etc/nginx/nginx.conf
    # 将站点信息也复制进去 
    COPY conf.d /etc/nginx/conf.d 
    ```

7. 再次构建 nginx

    `docker-compose up --build` 

    ```
    $ docker-compose up --build
    [+] Building 2.4s (6/6) FINISHED                              docker:desktop-linux
    => [nginx internal] load build definition from Dockerfile     0.0s
    ...
    nginx  | 2024/05/17 07:50:52 [notice] 1#1: start worker processes
    nginx  | 2024/05/17 07:50:52 [notice] 1#1: start worker process 22
    ```
    
    如果运行没有问题，我们可以再加个 `-d` 参数，让它保持在后台运行 `docker-compose up --build -d`。
    
    如果没有涉及到 Dockerfile 的修改，就不用加 `--build` 参数了，变成 `docker-compose up -d`
    
    `docker-compose ps`
    
    ```
    NAME                     IMAGE                  STATUS          ...
    docker-develop-nginx-1   docker-develop-nginx   Up 1 minutes    ...
    ```

自此，Nginx 环境就已经搭建好了，是不是看着很复杂繁琐，我也觉得；但是麻烦一时，方便日后，想想日后要是再需要搭建新环境时，只需要安装一个 Docker，然后 `docker-compose up --build -d` 等待片刻，环境就好了，是不是也很方便呢。

### Mysql

1. 创建 Dockerfile

    ```
    mkdir ~/docker-develop/mysql && touch ~/docker-develop/mysql/Dockerfile
    ```
    
3. 编辑 Dockerfile

    我们先去 [Docker Hub](https://hub.docker.com/) 找到 [MySQL](https://hub.docker.com/_/mysql) 镜像，这里我们选用：`mysql:8.4.0` 这个镜像

    *~/docker-develop/mysql/Dockerfile*

    ```dockerfile
    FROM mysql:8.4.0
    ```

4. 修改 docker-compose.yml

    *~/docker-develop/docker-compose.yml*

    ```yaml
    services:
      nginx:
        ...
      mysql:
          build: ./mysql
          labels:
            - mysql
            - database
          expose: 
            - 3306
          ports:
            - 3306:3306
          environment:
            MYSQL_ROOT_PASSWORD: s3qGmtrvYALxvF7M
          networks:
            - internal
    ```

    相比 nginx 的内容，这里多了一个 environment 的参数，为什么需要这个参数呢？这就得在  [MySQL](https://hub.docker.com/_/mysql) 找答案了，所以咱们还是得多看文档，下面引用的，复制原文在页面搜索相对应的介绍

    
    > 1. 这是简单启动的案例，这里指定了 -e 参数，也就是 environment
    > Starting a MySQL instance is simple: 
    > $ docker run --name some-mysql -e MYSQL_ROOT_PASSWORD=my-secret-pw -d mysql:tag
    > 
    > 2. docker-compose.yml 案例 
    > Example docker-compose.yml for mysql:
    > environment:
    >       MYSQL_ROOT_PASSWORD: example
    > 
    > 3. Environment Variables 环境变量的介绍
    > This variable is mandatory and specifies the password that will be set for the MySQL root superuser account. In the above > example, it was set to my-secret-pw.
    > 翻译：此变量是必需的，指定将为 MySQL root 超级用户帐户设置的密码。在上面的示例中，它被设置为 my-secret-pw 。
    

5. 配置文件

    根据 [文档](https://hub.docker.com/_/mysql) 的描述我们知道，mysql 的配置文件在容器里的 `/etc/mysql/my.cnf` 位置，所以，我们还是将 `my.cnf` 复制下来，修改后再替换虚拟机中的配置。

    > Using a custom MySQL configuration file
    > 
    > The default configuration for MySQL can be found in /etc/mysql/my.cnf, which may !includedir additional directories such as /etc/mysql/conf.d or /etc/mysql/mysql.conf.d. Please inspect the relevant files and directories within the mysql image itself for more details.


    复制配置文件 my.cnf

    ```sh
    # 复制容器里的 my.cnf 文件到本地，实际找到的 my.cnf 路径是在：/etc/my.cnf
    $ docker-compose cp mysql:/etc/my.cnf ~/docker-develop/mysql/my.cnf
    Successfully copied 2.56kB to ～/docker-develop/mysql/my.cnf
    ```
    
    更新 Dockerfile 覆盖容器内配置
    
    *~/docker-develop/mysql/Dockerfile*

    ```
    FROM mysql:8.4.0
    
    COPY my.cnf /etc/my.cnf
    ```


到这里是否发现，和 NGINX 的配置流程几乎相同，也没有什么很复杂的点。

5. 持久化数据

    MySQL 与 Nginx 有什么不同呢？那就是 MySQL 是有数据产生的，Nginx 没有数据产生(忽略日志)。

    上述都有复制宿主机的配置文件到容器，就是因为重启容器之后数据会丢失，到了 MySQL 这里重新构建后，数据也会丢失，所以要将数据给持久化(保存)到宿主机上。

    新建一个文件夹 `data` 存持久化的数据 ，在 data 目录下再创建对应的文件夹做区分，因为接下来要安装的 Redis 也存在数据持久化的问题。

    `mkdir -p ~/docker-develop/data/mysql`

    [MySQL](https://hub.docker.com/_/mysql) 文档 Where to Store Data 一节，我们知道数据是存在容器内的 /var/lib/mysql 文件夹中

    > Where to Store Data
    > The -v /my/own/datadir:/var/lib/mysql part of the command mounts the /my/own/datadir directory from the underlying host system as /var/lib/mysql inside the container, where MySQL by default will write its data files.

    编辑 docker-compose.yml 

    *~/docker-develop/data/mysql*

    ```yaml
    nginx:
    	...
    mysql:
        build: ./mysql
        volumes: # 将宿主机的 ./data/mysql_data 与容器的 /var/lib/mysql 目录相对应，实现数据数据持久化
            - ./data/mysql:/var/lib/mysql
        ...
    ```

6. 再次构建 MySql

    `docker-compose up mysql --build` 

    ```
    $ docker-compose up mysql --build
    [+] Building 1.1s (7/7) FINISHED                                docker:desktop-linux
    => [mysql internal] load build definition from Dockerfile       0.0s
    [+] Running 1/0
     ✔ Container docker-develop-mysql-1  Created                    0.0s
    ```

    `docker-compose ps`

    ```console
    NAME                     IMAGE                  STATUS          ...
    docker-develop-mysql-1   docker-develop-mysql   Up 17 minutes   ...
    docker-develop-nginx-1   docker-develop-nginx   Up 1 hours      ...
    ```

    可以使用 telnet 或者数据库链接工具测试是否成功

    ```
    $ telnet 127.0.0.1 3306
    Trying 127.0.0.1...
    Connected to localhost.
    Escape character is '^]'.
    ```
    
    

### Redis

1. 创建 Dockerfile

    ```
    mkdir ~/docker-develop/redis && touch ~/docker-develop/redis/Dockerfile
    ```

2. 编辑 Dockerfile

    我们先去 [Docker Hub](https://hub.docker.com/) 找到 [Redis](https://hub.docker.com/_/redis) 镜像，这里我们选用：`redis:7.2.4-alpine` 这个镜像

    *~/docker-develop/redis/Dockerfile*

    ```dockerfile
    FROM redis:7.2.4-alpine
    ```

3. 修改 docker-compose.yml

    *~/docker-develop/docker-compose.yml*

    ```yaml
    services:
      nginx:
        ...
      mysql:
        ...
      redis:
          build: ./redis
          labels:
            - redis
          expose:
            - 6379
          volumes:
            - ./data/redis:/data # 数据化持久目录，MySQL 一节细讲了，这里就不赘述了。
          ports:
            - 6379:6379
          restart: always
          networks:
            - internal
    ```

4. 创建数据化持久目录

    ```
    mkdir ~/docker-develop/data/redis
    ```

5. 配置文件

    在 [Redis](https://hub.docker.com/_/redis) 文章中只看到了如何配置的说明，但在容器内没有找到 redis.conf 的文件，所以我们可以从 Github 找到对应的 redis.conf 文件，下载下来使用。

    相关文档说明：

    > You can create your own Dockerfile that adds a redis.conf from the context into /data/, like so.
    > 
    > FROM redis
    > COPY redis.conf /usr/local/etc/redis/redis.conf
    > CMD [ "redis-server", "/usr/local/etc/redis/redis.conf" ]

    下载配置文件：

    在 github 搜索 redis，切换到对应的 [Tags](https://github.com/redis/redis/tree/7.2.4) ，根目录下有一个 redis.conf 文件，我们把它下载下来。

    > 可能需要代理，如果访问不了的话，那就直接页面打开，吧 redis.conf 文件复制下来也行。

    ```
    wget https://raw.githubusercontent.com/redis/redis/7.2.4/redis.conf -O ~/docker-develop/redis/redis.conf
    ```

    修改 redis.conf 基础配置，

    *~/docker-develop/redis/redis.conf*

    ```
    # 下面的修改都是基于本地开发环境的情况下，方便调试，测试，生产这样请勿这样配置
    bind 0.0.0.0 ::    # 监听所有 ip
    protected-mode no  # 关闭保护模式
    appendonly yes     # 开启数据持久化
    ```

    修改 Dockerfile 文件

    *~/docker-develop/redis/Dockerfile*

    ```
    FROM redis:7.2.4-alpine
    
    COPY redis.conf /usr/local/etc/redis/redis.conf
    
    CMD [ "redis-server", "/usr/local/etc/redis/redis.conf" ]
    ```

    可以看到，这里的配置几乎就是照搬文档的，所以，有什么问题可以多看看文档。

6. 构建&启动 redis

    `docker-compose up redis --build  `

    ```console
    docker-compose up redis --build                                                                            
    [+] Building 8.9s (7/7) FINISHED           docker:desktop-linux
    => [redis internal] load build definition from Dockerfile                  0.0s
    => => transferring dockerfile: 170B                                        0.0s
    => [redis internal] load metadata for docker.io/library/redis:7.2.4-alpine 1.6s
    ...
     ⠋ Container docker-develop-redis-1  Created   0.0s 
    Attaching to redis-1
    ```

    `docker-compose ps`

    ```console
    NAME                     IMAGE                  STATUS          ...
    docker-develop-mysql-1   docker-develop-mysql   Up 17 minutes   ...
    docker-develop-nginx-1   docker-develop-nginx   Up 1 hours      ...
    docker-develop-redis-1   docker-develop-redis   Up 1 hours      ...
    ```

### PHP-FPM

1. 创建 Dockerfile

    ```
    mkdir ~/docker-develop/php83 && touch ~/docker-develop/php83/Dockerfile
    ```

2. 编辑 Dockerfile

    我们先去 [Docker Hub](https://hub.docker.com/) 找到 [PHP-FPM](https://hub.docker.com/_/php) 镜像，这里我们选用：`php:8.3-fpm-alpine` 这个镜像

    *~/docker-develop/php83/Dockerfile*

    ```dockerfile
    FROM php:8.3-fpm-alpine
    ```

3. 修改 docker-compose.yml

    *~/docker-develop/docker-compose.yml*

    ```yaml
    services:
      nginx:
        ...
      mysql:
        ...
      redis:
        ...
      php83:
        build: ./php83
        labels:
          - php
          - php-fpm
        expose:
          - 9000
        restart: always
        networks:
          - internal
    ```

4. 启动

    `docker-compose up php83 --build`

5. 配置文件

    >    Configuration
    >
    >    This image ships with the default php.ini-development and php.ini-production configuration files.
    >    It is strongly recommended to use the production config for images used in production environments!
    >    The default config can be customized by copying configuration files into the $PHP_INI_DIR/conf.d/ directory.
    >    ...
    >
    >    ```php
    >    FROM php:8.2-fpm-alpine
    >    
    >    # Use the default production configuration
    >    RUN mv "$PHP_INI_DIR/php.ini-production" "$PHP_INI_DIR/php.ini"
    >    ```

文档的原文描述提供了两个 php.ini 的配置文件 php.ini-development 和 php.ini-production，这里我们是开发环境。就把 php.ini-development 复制出来使用，而 php.ini-production 就不用理会了。

这里的 php.ini 路径是定义了一个变量 $PHP_INI_DIR，我们要如何知道这个变量的值呢？

- 推荐使用 `docker-compose run php83 sh -c 'echo $PHP_INI_DIR'` 

```console
$ docker-compose run php83 sh -c 'echo $PHP_INI_DIR'
/usr/local/etc/php
```
- 使用命令：`docker-compose exec php82 sh` 进入容器后，使用 `echo $PHP_INI_DIR` 可以找出变量值。

- 查询 php 的 [Dockerfile](https://github.com/docker-library/php/blob/d4474c4041a17923f13a26a2595c786a98de9308/8.3/bookworm/fpm/Dockerfile#L42C17-L42C31) 中的定义

**复制配置文件到宿主机**

```
# php.ini
docker-compose cp php83:/usr/local/etc/php/php.ini-development ~/docker-develop/php83/php.ini
# php-fpm 
docker-compose cp php83:/usr/local/etc/php-fpm.d/www.conf ~/docker-develop/php83/www.conf
```

6. 修改 Dockerfile 文件，替换配置文件

    *~/docker-develop/php83/Dockerfile*

    ```dockerfile
    FROM php:8.3-fpm-alpine
    
    COPY php.ini "$PHP_INI_DIR/php.ini"
    COPY www.conf /usr/local/etc/php-fpm.d/www.conf
    ```

基础软件包已经全部安装完成，接下来我们要整合起来，让他们能够协同工作。

7. 整体构建一次

    `docker-compose up --build -d && docker-compose ps`

    ```
    NAME                     IMAGE                   STATUS           
    docker-develop-mysql-1   docker-develop-mysql   Up 22 hours       
    docker-develop-nginx-1   docker-develop-nginx   Up About a minute 
    docker-develop-php83-1   docker-develop-php83   Up About a minute 
    docker-develop-redis-1   docker-develop-redis   Up 6 hours        
    ```

    如果 STATUS 状态不正确，那么就吧 `-d` 参数去掉看看报错，调整好之后再加上 `-d` 参数

## 整合

我们通过构建一个 Laravel 项目来串通所有用到的应用，代码放在 `~/docker-develop/www` 下

1. 创建代码目录

    `mkdir ~/docker-develop/www`

2. 拉取项目

    `git clone git@github.com:laravel/laravel.git ~/docker-develop/www/laravel-app`

3. 写一个接口，方便后面调试
    *~/docker-develop/www/laravel-app/routes/web.php*

    ```
    Route::get('/get_php_version', function () {
        return phpversion();
    });
    
    Route::get('/get_db_version', function () {
        return DB::connection()->getPdo()->getAttribute(PDO::ATTR_SERVER_VERSION);
    });
    
    Route::get('/get_redis_version', function () {
        $info = \Illuminate\Support\Facades\Redis::info();
        return $info['redis_version'];
    });
    
    ```
    
    
    
4. 配置 NGINX 站点

    *~/docker-develop/nginx/conf.d*

    `vim ~/docker-develop/nginx/conf.d/laravel-app.conf`

    直接复制文档的 [NGINX 站点配置](https://learnku.com/docs/laravel/10.x/deployment/14840#62e0b5)，这里我们要修改 server_name、root、fastcgi_pass 这三个参数

    ```nginx
    server {
        listen 80;
        listen [::]:80;
        server_name laravel-app.test; # 原来的值：example.com;
        root /www/laravel-app/public; # 原来的值：/srv/example.com/public;
        ...
        location ~ \.php$ {
            fastcgi_pass php83:9000; # 原来的值：unix:/var/run/php/php8.1-fpm.sock;
            fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
            include fastcgi_params;
        }
        ...
    }
    ```

    我们要知道，虽然现在是在本地修改了配置文件，但执行 docker-compose up 的时候，是会将配置文件覆盖到容器里面使用的，所以真正起作用的是容器里面的配置，而非本地的配置，了解这些后，我们再来看如何修改这三个参数。

    - server_name 我们修改成：`server_name laravel-app.test`

    - root 我们知道是要配置 laravel 项目的路径，但现在 nginx 和 php 是分属在两个容器里面，问题来了。

        - 我只知道本地路径是在 `~/docker-develop/www/laravel` 里，不知道容器里面项目路径在哪里。
        - 两个容器怎样才能相互访问代码文件？

        这个时候我们需要修改 php 的 Dockerfile 以及 docker-compose.yml 了

        *~/docker-develop/php83/Dockerfile*

        ```dockerfile
        FROM php:8.3-fpm-alpine
        
        COPY php.ini "$PHP_INI_DIR/php.ini"
        COPY www.conf /usr/local/etc/php-fpm.d/www.conf
        
        # 我们将工作的目录设置在 /www 文件夹下，这样每次进容器都会默认切换到 /www 目录下了
        WORKDIR /www
        ```

        *~/docker-develop/docker-compose.yml*

        ```dockerfile
        services:
        	php-fpm82:
            build: ./php82
            volumes:
              - ~/docker-develop/www:/www
            ...
          nginx:
            build: ./nginx
            volumes:
              - ~/docker-develop/www:/www
        ```

        使用 volumes 将上面两个问题解决了，现在我们知道了 root 的目录应该怎么设置了吧。

        `root /www/laravel-app/public`

    - fastcgi_pass 我们通常在本地设置为：127.0.0.1:9000 、localhost:9000 或者文档中 unix 的形式，在这里我们只需要将容器名称就可以了。

        `fastcgi_pass php-fpm83:9000`

4. 测试一下：

    `docker-compose up --build`

    `curl http://laravel-app.test/version`

    ```sh
    $ curl http://laravel-app.test/version
    <br />
    <b>Warning</b>:  require(/www/laravel-app/public/../vendor/autoload.php): Failed to open stream: No such file or directory in <b>/www/laravel-app/public/index.php</b> on line <b>13</b><br />
    ```

    问题又来了，我们前面知识 clone 了项目，没有使用 comopser install 呀，还有，我们现在也没有 composer，那就得来安装 composer 了。

5. 在 PHP 容器内安装 composer

    *~/docker-develop/php83/Dockerfile*

    ```dockerfile
    FROM php:8.3-fpm-alpine
    
    COPY php.ini "$PHP_INI_DIR/php.ini"
    COPY www.conf /usr/local/etc/php-fpm.d/www.conf
    
    WORKDIR /www
    
    # 在容器内部安装 composer 
    RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
    ```

    重新构建后，还要执行 compose install，同样的也可以进入容器手动执行。

    `docker-compose run php83 sh -c 'cd /www/laravel-app && composer install -vvv'`

    跑一下 Laravel 基础配置，脚本在项目下的 composer.json 中定义的

    `docker-compose run php83 sh -c 'cd /www/laravel-app && composer run-script post-root-package-install'`

    `docker-compose run php83 sh -c 'cd /www/laravel-app && composer run-script post-create-project-cmd`

6. 再测一下

    `docker-compose up --build`

    测试是否能正常访问到项目

    `curl http://laravel-app.test/version`

    ```console
    $ curl http://laravel-app.test/version
    8.3.7%
    ```

    测试 MySql 交互是否正常

    配置数据库：
    
    *~/docker-develop/www/laravel-app/.env*
    
    ```php
    DB_CONNECTION=mysql 
    DB_HOST=mysql        # 这里填写容器名就好，与上面的 fastcgi_pass 相同的方法
    DB_PORT=3306
    DB_DATABASE=null     # 目前还没有任何数据库，我们这里填写 null
    DB_USERNAME=root 
    DB_PASSWORD=GqLWQ2YU # docker-compose.yml 中配置的 MYSQL_ROOT_PASSWORD 变量的值
    ```

    进入容器测试

    ```sh
    $ docker-compose exec php83 sh
    /www # ls
    laravel-app
    
    /www # cd laravel-app/
    /www/laravel-app # php artisan tinker
    Psy Shell v0.12.3 (PHP 8.3.7 — cli) by Justin Hileman
    > DB::statement('CREATE DATABASE laravel_app');
    
    Illuminate\Database\QueryException  could not find driver (Connection: mysql, SQL: CREATE DATABASE laravel_app).
    ```
    
    使用 Laravel 的 tinker 工具常识创建一个 laravel_app 的数据库，失败了，提示 could not find driver，根据 [Laravel 文档](https://learnku.com/docs/laravel/10.x/deployment/14840#df1b28) 数据库相关的有 PDO 扩展。
    
    那我们就在在容器中查看当前与 PDO 相关的扩展：
    
    ```
    /www/laravel-app # php -m | grep -i pdo
    PDO
    pdo_sqlite
    ```
    
    发现有 PDO 和一个 pdo_sqlite，可以推断出，应该还需要一个 pdo_mysql 的扩展，那怎么安装呢？我们去看看 MySQL 的文档：
    
    > ## How to install more PHP extensions
    >
    > We provide the helper scripts `docker-php-ext-configure`, `docker-php-ext-install`, and `docker-php-ext-enable` to more easily install PHP extensions.
    
    所以根据文档所描述，我们可以使用简单的 docker-php-ext-install 来安装扩展，用 docker-php-ext-enable 开启扩展，扩展列表这里：[install-php-extensions](https://github.com/mlocati/docker-php-extension-installer).
    
    编辑 Dockerfile，安装 `pdo_mysql` 扩展
    
    *~/docker-develop/php83/Dockerfile*
    
    ```dockerfile
    FROM php:8.3-fpm-alpine
    ...
    # 安装扩展
    # https://github.com/mlocati/docker-php-extension-installer
    RUN docker-php-ext-install pdo_mysql 
    # 开启扩展
    RUN docker-php-ext-enable pdo_mysql   
    ```
    
    重新构建后，再试试数据库是否正常
    
    `docker-compose up --build php83 -d`
    
    ```
    # 查看扩展是否已加载
    /www # php -m | grep -i pdo
    PDO
    pdo_mysql
    pdo_sqlite
    ```
    
    再次创建数据库成功：
    
    ```
    /www/laravel-app # php artisan tinker
    Psy Shell v0.12.3 (PHP 8.3.7 — cli) by Justin Hileman
    > DB::statement('CREATE DATABASE laravel_app');
    = true
    ```

8. 测试 redis 链接情况

    配置 redis

    ```php
    REDIS_CLIENT=phpredis
    REDIS_HOST=redis
    REDIS_PASSWORD=null
    REDIS_PORT=6379
    ```

    请求

    ```sh
    curl http://laravel-app.test/get_redis_version
    ...
    Class "Redis" not found
    ...
    ```

    很明显是没有 redis 扩展，但这里我使用 docker-php-ext-install 找不到 redis 的扩展，但文档上是有 redis 的扩展，这里我们使用 pecl 安装吧

    *~/docker-develop/php83/Dockerfile*

    ```dockerfile
    FROM php:8.3-fpm-alpine
    ...
    RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
    # $PHPIZE_DEPS 包含软件：autoconf dpkg-dev dpkg file g++ gcc libc-dev make pkgconf re2c
    # iputils 常用的 ping 工具在此包内，方便调试
    RUN apk add --no-cache \
            $PHPIZE_DEPS \ 
            iputils 
    
    pecl install redis-6.0.2
    
    # 开启扩展
    RUN docker-php-ext-enable pdo_mysql redis
    ```

    再次请求

    ```sh
    curl http://laravel-app.test/get_redis_version
    7.2.4
    ```

## 总结

我们按照本地搭建环境的流程，成功搭建了一套 docker 的开发流程，看起来颇为繁琐，但写了两个 Dockerfile 之后不知道你是否发现，其实很多都是重复的工作，不重复的也都是针对某个软件特别安装的，也不算很困难。

## 拓展

1. php 我给它的命名是 php83，这代表安装了 php8.3 这个版本，可以拓展再安装个 php72，然后在 nginx 站点的 `fastcgi_pass php72:9000;` 即可。
2. 通常还需要 supervisor 来运行 Laravel 的定时任务、队列等功能，可以参考 composer 的安装方式安装 supervisor。
