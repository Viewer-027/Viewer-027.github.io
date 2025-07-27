## docker快速部署wordpress

https://www.cnblogs.com/pzy-Albert/p/18391693

## docker制作wordpress镜像容器

```bash
# 创建dockerfile目录
[root@docker02 ~]# mkdir /Dockerfile

# 进入dockerfile目录
[root@docker02 ~]# cd /Dockerfile

# 编写Docker File
[root@docker02 Dockerfile]# vim Dockerfile
```

```bash
FROM myos:yum-linux

# 安装依赖库和工具
RUN yum -y update && \
    yum -y install epel-release && \
    yum -y install wget && \
    yum -y install tar && \
    yum -y install openssl && \
    yum -y install numactl-libs && \
    yum -y install ncurses-compat-libs && \
    yum -y install libaio && \
    yum -y install perl && \
    yum clean all

WORKDIR /app
# 下载并安装 MySQL
COPY mysql-8.0.13-linux-glibc2.12-x86_64.tar.xz mysql-8.0.13-linux-glibc2.12-x86_64.tar.xz
RUN tar xf mysql-8.0.13-linux-glibc2.12-x86_64.tar.xz && \
    useradd mysql -s /sbin/nologin -M && \
    mv mysql-8.0.13-linux-glibc2.12-x86_64 /app/mysql-8.0.13 && \
    rm -f mysql-8.0.13-linux-glibc2.12-x86_64.tar.gz && \
    ln -s /app/mysql-8.0.13 /app/mysql && \
    chown mysql.mysql -R /app/mysql* && \
    cp /app/mysql/support-files/mysql.server /etc/init.d/mysqld && \
    cd /app/mysql/bin/ && \
    ./mysqld --initialize-insecure --user=mysql --basedir=/app/mysql --datadir=/app/mysql/data

# 部署wordpress
COPY wordpress-6.1.1-zh_CN.tar.gz  /wordpress-6.1.1-zh_CN.tar.gz
RUN mkdir /code && \
    yum install -y nginx php-fpm php php-json php-mysqlnd && \
    groupadd www -g 666 && \
    useradd www -u 666 -g 666 -s /sbin/nologin/ -M && \
    sed -i 's#user nginx#user www#' /etc/nginx/nginx.conf && \
    sed -i 's#user = apache#user = www#' /etc/php-fpm.d/www.conf && \
    sed -i 's#group = apache#group = www#' /etc/php-fpm.d/www.conf && \
    cd /code && \
    tar xf /wordpress-6.1.1-zh_CN.tar.gz -C /code  && \
    chown -R www.www /code/*

# 复制wordpress配置文件
COPY wp.conf /etc/nginx/conf.d/

# 设置环境变量
ENV PATH=/app/mysql/bin:$PATH
ENV PHP_FPM_CONF_FILE=/etc/php-fpm.conf
ENV NGINX_CONF_FILE=/etc/nginx/nginx.conf

# 创建wordpress用户并复制 MySQL 配置文件
COPY my.cnf /etc/my.cnf
RUN /etc/init.d/mysqld start && \
    /app/mysql/bin/mysql -e "create database wordpress character set utf8mb4;" && \
    /app/mysql/bin/mysql -e "create user wpuser01@localhost identified by 'wordpress';" && \
    /app/mysql/bin/mysql -e "grant all privileges on wordpress.* to wpuser02@localhost;"
# 创建数据目录
VOLUME /app/mysql/data

# 暴露 MySQL 端口
EXPOSE 3306 80

# 启动服务
CMD service mysqld start && nginx -c $NGINX_CONF_FILE && php-fpm -c $PHP_FPM_CONF_FILE
```

## 准备 Dockerfile 所需文件

```bash
# mysql配置文件
[root@docker02 Dockerfile]# cat my.cnf
[mysqld]
basedir=/app/mysql
datadir=/app/mysql/data

# wordpress数据文件
[root@docker02 Dockerfile]# cat wp.sql
-- MySQL dump 10.13  Distrib 5.7.42, for linux-glibc2.12 (x86_64)
--
-- Host: localhost    Database: wp
-- ------------------------------------------------------
-- Server version       5.7.42

/*!40101 SET @OLD_CHARACTER_SET_CLIENT=@@CHARACTER_SET_CLIENT */;
/*!40101 SET @OLD_CHARACTER_SET_RESULTS=@@CHARACTER_SET_RESULTS */;
/*!40101 SET @OLD_COLLATION_CONNECTION=@@COLLATION_CONNECTION */;
/*!40101 SET NAMES utf8 */;
/*!40103 SET @OLD_TIME_ZONE=@@TIME_ZONE */;
/*!40103 SET TIME_ZONE='+00:00' */;
/*!40014 SET @OLD_UNIQUE_CHECKS=@@UNIQUE_CHECKS, UNIQUE_CHECKS=0 */;
/*!40014 SET @OLD_FOREIGN_KEY_CHECKS=@@FOREIGN_KEY_CHECKS, FOREIGN_KEY_CHECKS=0 */;
/*!40101 SET @OLD_SQL_MODE=@@SQL_MODE, SQL_MODE='NO_AUTO_VALUE_ON_ZERO' */;
/*!40111 SET @OLD_SQL_NOTES=@@SQL_NOTES, SQL_NOTES=0 */;
/*!40103 SET TIME_ZONE=@OLD_TIME_ZONE */;

/*!40101 SET SQL_MODE=@OLD_SQL_MODE */;
/*!40014 SET FOREIGN_KEY_CHECKS=@OLD_FOREIGN_KEY_CHECKS */;
/*!40014 SET UNIQUE_CHECKS=@OLD_UNIQUE_CHECKS */;
/*!40101 SET CHARACTER_SET_CLIENT=@OLD_CHARACTER_SET_CLIENT */;
/*!40101 SET CHARACTER_SET_RESULTS=@OLD_CHARACTER_SET_RESULTS */;
/*!40101 SET COLLATION_CONNECTION=@OLD_COLLATION_CONNECTION */;
/*!40111 SET SQL_NOTES=@OLD_SQL_NOTES */;

-- Dump completed on 2023-09-10  9:20:32

# wordpress的nginx配置文件
[root@docker02 Dockerfile]# cat wp.conf
server{
        listen 80;
        server_name _;
        root /code/wordpress;

        location / {
                index index.php index.html;
        }
        location ~ \.php$ {
                fastcgi_pass 127.0.0.1:9000;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                include /etc/nginx/fastcgi_params;
        }
}

# 上传mysql-5.7.42安装包
 wget https://cdn.mysql.com//archives/mysql-8.0/mysql-8.0.13-linux-glibc2.12-x86_64.tar.xz
```

## 开始制作成镜像

```bash
# 使用docker build执行dockerfile并指定镜像名与标签，最后的点是当前目录
[root@docker02 Dockerfile]# docker build -t wordpress:1 .

# 使用docker run运行并进入容器
[root@docker02 Dockerfile]# docker run --rm --name wordpress -p 80:80 -it wordpress:1 bash
## 启动mysql
[root@d1fd726876f9 app]# /etc/init.d/mysqld start
## 启动nginx
[root@d1fd726876f9 app]# nginx -c /etc/nginx/nginx.conf
nginx -s reload
## 后台启动php-fpm
[root@d1fd726876f9 app]# php-fpm -c /etc/php-fpm.conf &
```

docker快速部署wordpress

```bash
sudo apt-get remove docker docker-engine docker.io containerd runc

sudo apt-get update
sudo apt-get upgrade

sudo apt-get install apt-transport-https ca-certificates curl gnupg-agent software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

```bash
docker pull wordpress

docker run -it --name wordpress -p 9999:80 -v /usr/local/share/wordpress:/var/www/html -d wordpress

--name wordpress => 为容器指定一个名称，此处命名为 wordpress

-p 9999:80 => 将容器内部的端口映射到宿主机的端口，此处容器将80端口映射到宿主机的9999端口

-v /usr/local/share/wordpress:/var/www/html => 使用挂载卷将宿主机的目录挂载到容器内部，此处是将宿主机的 /usr/local/share/wordpress 挂载到容器内部的 /var/www/html 目录。宿主机目录是什么意思呢，就是打开 ftp 你所看到的目录就是宿主机上实际的目录，操作（增删改）宿主机上 /usr/local/share/wordpress 这个目录下的文件，等同于操作容器内部的 /var/www/html 这个目录，可以看成自动同步吧，我是这么理解的

-d => 这个参数代表以 “detached” 模式运行容器，就是在后台运行

-d 后面的wordpress => 就是刚才拉取的 wordpress 镜像名称
```

配置Mysql

```bash
docker run -d --name mysql -v /usr/local/share/mysql/data:/var/lib/mysql -v /usr/local/share/mysql/conf:/etc/mysql/conf.d -v /usr/local/share/mysql/logs:/var/log/mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 mysql:8

docker exec -it mysql /bin/bash
mysql -u root -p

GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;

create database wordpress;

docker inspect <mysql-container-name> | grep IPAddress

```

编辑

```bash
wp-config-sample.php
```
