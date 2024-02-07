# 测试工作支撑

## Zentao

禅道开源版：http://dl.cnezsoft.com/zentao/docker/docker_zentao.zip

数据库用户名：root，默认密码：123456。

注意：需要关闭selinux
---
```
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
reboot
```

下载安装包，解压缩。 进入docker_zentao目录

1、构建镜像  
```
docker build -t zentao .
```

2、运行镜像  
```
docker run --name zentao -p 80:80 -d \
    -v /data/www:/app/zentaopms -v /data/data:/var/lib/mysql \
    -e MYSQL_ROOT_PASSWORD=123456 \
    zentao:latest
```

## TestLink

1、为应用程序和数据库创建新网络 
```
docker network create testlink-tier
```

2、为MariaDB持久性创建卷并创建MariaDB容器
```
docker run -d --name mariadb \
  -e ALLOW_EMPTY_PASSWORD=yes \
  -e MARIADB_USER=bn_testlink \
  -e MARIADB_DATABASE=bitnami_testlink \
  --net testlink-tier \
  --volume /path/to//mariadb:/bitnami \
  bitnami/mariadb:latest
```
> 注意：您需要为容器指定一个名称，以便TestLink解析主机

3、为Testlink持久性创建卷并启动容器
```
docker run -d --name testlink -p 80:80 -p 443:443 \
  -e ALLOW_EMPTY_PASSWORD=yes \
  -e TESTLINK_DATABASE_USER=bn_testlink \
  -e TESTLINK_DATABASE_NAME=bitnami_testlink \
  -e TESTLINK_USERNAME=admin
  -e TESTLINK_PASSWORD=p@ssw0rd
  -e TESTLINK_EMAIL=admin@example.com
  --net testlink-tier \
  --volume /path/to//testlink:/bitnami \
  bitnami/testlink:1.9.19-debian-9-r133
```

然后，可以通过 http://your-ip/ 访问TestLink


