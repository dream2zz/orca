虎鲸计划：提升测试动力
===

![](images/orca-mini.fw.png)

<!-- type 1-Context.md 2-Support.md 3-Manager.md > README.md -->
<!-- TOC -->

- [1. 背景](#1-背景)
- [2. 目标](#2-目标)
- [3. 测试工作支撑](#3-测试工作支撑)
    - [3.1. Zentao](#31-zentao)
    - [3.2. TestLink](#32-testlink)
- [4. 测试环境管理](#4-测试环境管理)
    - [4.1. 测试环境架构](#41-测试环境架构)
    - [4.2. 自动化工作流](#42-自动化工作流)
    - [4.3. 现有环境梳理](#43-现有环境梳理)

<!-- /TOC -->

# 1. 背景

伴随着研发团队的成长，测试工作也出现了众多痛点：

1. 测试环境构建对测试人员部署有一定能力要求，人工处理量大，并且容易出错；每个人都需要具备部署能力，在文档有限、培训有限条件下，很难做到人人掌握。

2. 从零搭建项目测试环境特别冗长，近几年几乎没有改变。

3. 项目间测试环境具有共性，一个项目对应一套虚拟环境(虚拟机集群)，甚至一个测试迭代对应一套环境，冗余度非常高，需考虑做资源的整体整合。

4. 测试环境搭建流程，没有做到统一性、规范性，对项目组、测试或其他部门来说，没有做到已有资源的整合（文档标准化、部署标准化、自动化或半自动化）。

以上种种情况，导致我们的IT基础设施中虚拟机的数量非常庞大，用途非常不明确，资源越来越紧张，与项目的互动同步越来越脆弱。

我们需要学习一下行业当中的优秀IT企业对基础设施编排的做法，对测试工作管理的最佳实践，以及测试与开发之间环境搭配的最佳实践。

# 2. 目标

提升测试动力，本期方案主要的目标是：

* 测试工作管理支撑平台的升级，采用容器平台，节省资源；

* 打造测试环境管理工作流，统筹规划开发环境、测试环境，简化部署过程，统一规范标准，融合共用资源。


# 3. 测试工作支撑

## 3.1. Zentao

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

## 3.2. TestLink

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


# 4. 测试环境管理

## 4.1. 测试环境架构

* 以裸机或者工作站为基础，操作系统可以是linux或者window;

* 在上面的基础之上，部署自动化套件，提供自动化平台；

* 运用测试环境管理能力，批量启动需要的虚拟机。

![](images/automate.png)


## 4.2. 自动化工作流

1. 开发、测试、运维人员编写Vagrant工程文件，并提交到Git仓库；

2. 负责人向管理员提交工单需求，申请自动化平台资源；

3. 管理员审批工单需求，并同意新虚拟机群准入；

4. 管理员拉取Vagrant工程，执行测试环境管理，并校验。

![](images/automate-flow.png)


## 4.3. 现有环境梳理

1. 本篇测试环境，既不取代软件在客户现场的部署方式，也不覆盖项目组中个性化的需求（基于其他虚拟化）。

2. 容器平台依然为容器化部署项目的主要部署地，假如需要验证部署方案，本篇方案是首选。

3. 测试环境伴随测试工作，应该是一个多变的临时的环境，也即每次测试迭代都应该从初始环境开始，而不应该携带上一次测试留下的痕迹，测试迭代结束后应该立即清理本轮测试环境。

4. 后期，我们会加上持续部署工具，以辅助我们对环境的管理。


