---
title: "Running Dcm4chee Arc Light on Docker"
date: 2021-02-03T09:26:58+08:00
tags: [ "DICOM" ]
keywords: [ "DICOM" ]
categories: [ "DICOM" ]
draft: true
isCJKLanguage: true
---

首先需要[安装Docker](https://docs.docker.com/get-docker/)

安装完成后，拉取以下3个仓库

``` bash
$ docker pull dcm4che/slapd-dcm4chee:2.4.56-23.1
$ docker pull dcm4che/postgres-dcm4chee:13.1-23
$ docker pull dcm4che/dcm4chee-arc-psql:5.23.0
```

拉取完成后，就可以启动服务了，这里有两种方式，一种是使用Docker命令行依次启动上面3个服务，不过比较麻烦，也容易出错，另一种是直接使用Docker Copmose，相对来说要简单很多。这里就直接使用 Docker Compose的方式。

创建以下两个文件：

`docker-compose.yml`

``` yaml
version: "3"
services:
  ldap:
    image: dcm4che/slapd-dcm4chee:2.4.56-23.1
    logging:
      driver: json-file
      options:
        max-size: "10m"
    ports:
      - "389:389"
    env_file: docker-compose.env
    volumes:
      - /var/local/dcm4chee-arc/ldap:/var/lib/openldap/openldap-data
      - /var/local/dcm4chee-arc/slapd.d:/etc/openldap/slapd.d
  db:
    image: dcm4che/postgres-dcm4chee:13.1-23
    logging:
      driver: json-file
      options:
        max-size: "10m"
    ports:
      - "5432:5432"
    env_file: docker-compose.env
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /etc/timezone:/etc/timezone:ro
      - /var/local/dcm4chee-arc/db:/var/lib/postgresql/data
  arc:
    image: dcm4che/dcm4chee-arc-psql:5.23.0
    logging:
      driver: json-file
      options:
        max-size: "10m"
    ports:
      - "8080:8080"
      - "8443:8443"
      - "9990:9990"
      - "9993:9993"
      - "11112:11112"
      - "2762:2762"
      - "2575:2575"
      - "12575:12575"
    env_file: docker-compose.env
    environment:
      WILDFLY_CHOWN: /opt/wildfly/standalone /storage
      WILDFLY_WAIT_FOR: ldap:389 db:5432
    depends_on:
      - ldap
      - db
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /etc/timezone:/etc/timezone:ro
      - /var/local/dcm4chee-arc/wildfly:/opt/wildfly/standalone
      - /var/local/dcm4chee-arc/storage:/storage
```

`docker-compose.env`

``` ini
TZ=Asia/Shanghai
STORAGE_DIR=/storage/fs1
POSTGRES_DB=pacsdb
POSTGRES_USER=pacs
POSTGRES_PASSWORD=pacs
```

其中`TZ`是用来设置时区的。

启动

``` bash
$ docker-compose -p dcm4chee up -d
```

停止

``` bash
$ docker-compose -p dcm4chee stop
```

重新启动

``` bash
$ docker-compose -p dcm4chee start
```

删除

``` bash
$ docker-compose -p dcm4chee down
```

**参考**

[Run minimum set of archive services on a single host](https://github.com/dcm4che/dcm4chee-arc-light/wiki/Run-minimum-set-of-archive-services-on-a-single-host)
