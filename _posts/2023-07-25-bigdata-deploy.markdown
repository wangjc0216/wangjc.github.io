---
layout: post
title:  "大数据组件部署"
date:   2023-07-25 20:00:00 +0800
categories: bigdata
---

主要部署了minio、dolphin、dinky、flink、kafka，有部分还没有做HA，在后续会接着更新。


**minio**

```
helm  install minio minio  --set global.storageClass=nfs --set apiIngress.hostname=minio-data.aikosolar.net --set persistence.size=200Gi --set service.type=NodePort --set auth.rootPassword=admin@data -ndata
```



**dolphin**
> https://dolphinscheduler.apache.org/zh-cn/docs/3.1.6/guide/installation/kubernetes#appendix-configuration

- 打包

重新打包镜像，其中mysql-connector的jar包下载在：[jar包下载](https://repo1.maven.org/maven2/mysql/mysql-connector-java/8.0.16/mysql-connector-java-8.0.16.jar)

datax的jar包可以参考官方文档：[官方文档](https://github.com/alibaba/DataX/blob/master/userGuid.md)

需要注意，在worker要安装python2.7

因为dolphin需要安装一些jar包，所以需要自己来打镜像。
```
dsimage# cat Dockerfile-dolphinscheduler-*
FROM dolphinscheduler.docker.scarf.sh/apache/dolphinscheduler-alert-server:3.1.7
COPY mysql-connector-java-8.0.16.jar /opt/dolphinscheduler/libs

FROM dolphinscheduler.docker.scarf.sh/apache/dolphinscheduler-api:3.1.7
COPY mysql-connector-java-8.0.16.jar /opt/dolphinscheduler/libs

FROM dolphinscheduler.docker.scarf.sh/apache/dolphinscheduler-master:3.1.7
COPY mysql-connector-java-8.0.16.jar /opt/dolphinscheduler/libs

FROM dolphinscheduler.docker.scarf.sh/apache/dolphinscheduler-tools:3.1.7
COPY mysql-connector-java-8.0.16.jar /opt/dolphinscheduler/tools/libs

FROM dolphinscheduler.docker.scarf.sh/apache/dolphinscheduler-worker:3.1.7
RUN mkdir -p /opt/module/
COPY mysql-connector-java-8.0.16.jar /opt/dolphinscheduler/libs
COPY datax  /opt/module/datax
RUN apt -y update && apt install -y python-is-python3 && apt-get install -y python2.7 python-pip
```

执行对应的构建：
```
cd /root/data/ds/apache-dolphinscheduler-3.1.7-src/deploy/kubernetes/dsimage/
docker build   -t    some-registry/library/dolphinscheduler-alert-server:3.1.7     -f Dockerfile-dolphinscheduler-alert-server-3.1.7 .
docker build   -t    some-registry/library/dolphinscheduler-api:3.1.7     -f Dockerfile-dolphinscheduler-api-3.1.7 .
docker build   -t    some-registry/library/dolphinscheduler-master:3.1.7     -f Dockerfile-dolphinscheduler-master-3.1.7 .
docker build   -t    some-registry/library/dolphinscheduler-tools:3.1.7     -f Dockerfile-dolphinscheduler-tools-3.1.7 .
docker build   -t    some-registry/library/dolphinscheduler-worker:3.1.7     -f Dockerfile-dolphinscheduler-worker-3.1.7 .

docker push some-registry/library/dolphinscheduler-alert-server:3.1.7
docker push some-registry/library/dolphinscheduler-api:3.1.7     
docker push some-registry/library/dolphinscheduler-master:3.1.7     
docker push some-registry/library/dolphinscheduler-tools:3.1.7     
docker push some-registry/library/dolphinscheduler-worker:3.1.7     
```

- values.yaml中修改image.registry

需要修改成自己的镜像仓库

- values.yaml中，修改mysql配置：
```
externalDatabase:
  type: "mysql"
  host: "172.16.1.60"
  port: "3306"
  username: "dolphin"
  password: "dolphin"
  database: "dolphinscheduler"
  params: "characterEncoding=utf8"
```

- helm 安装
```
tar -zxvf apache-dolphinscheduler-<version>-src.tar.gz
cd apache-dolphinscheduler-<version>-src/deploy/kubernetes/dolphinscheduler
helm repo add bitnami https://charts.bitnami.com/bitnami
helm dependency update .
helm install dolphinscheduler . --set image.tag=3.1.7 -ndata
```

- common.propertites  配置文件修改

S3 修改：(https://www.infoq.cn/article/uxfpyzpt3mbbwv9tlspr)

```
  FS_DEFAULT_FS: s3a://dfs
  FS_S3A_ACCESS_KEY: admin
  FS_S3A_ENDPOINT: minio.data:9000
  FS_S3A_SECRET_KEY: admin@data
  RESOURCE_STORAGE_TYPE: S3
```

DATAX_HOME修改: /opt/module/datax/ ，并不是完整路径，和官方文档写的略有不同。


- 部署完成

可以登录验证了！


**datax:（在flink中内置了）**
> https://github.com/alibaba/DataX/blob/master/userGuid.md


**dinky**
> https://github.com/DataLinkDC/dinky

git版本切换到tag v0.7.3 部署在172.16.97.197 msbdcn12  8888
在导入数据时，先导入了dinky.sql, 0.7.0 dinky_ ddl ,dinky_dml.sql , 0.7.3 dinky_ddl.sql

如果在master，则会因为table名称修改而报错。


**kafka install**
> https://artifacthub.io/packages/helm/bitnami/kafka

- helm install
```
   helm install data-kafka  --set replicaCount=3,externalAccess.enabled=true,externalAccess.service.type=NodePort,externalAccess.service.nodePorts[0]='30787',externalAccess.service.nodePorts[1]='31317',externalAccess.service.nodePorts[2]='30113',externalAccess.service.port=9094,externalAccess.autoDiscovery.enabled=true,serviceAccount.create=true,rbac.create=true oci://registry-1.docker.io/bitnamicharts/kafka -ndata
```
- 修改，删掉ready probe

否则会因为dns问题而crash，对应[issue](https://github.com/bitnami/charts/issues/17797)



**flink(没有持久化)**

1. helm install flink bitnami/flink --set image.tag=v1.17.1  -ndata

2. helm pull bitnami/flink 拉下来在修改limit和request

3. 在flink helm 目录中 helm upgrade flink . -f values.yaml -ndata

- 注意，jobmanager和taskmanager都要修改resources和limit和request