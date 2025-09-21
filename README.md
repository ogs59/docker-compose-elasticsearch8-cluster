## docker-compose-elasticsearch8-cluster
docker-compose部署es8集群证书版

### 系统配置修改

1. 修改文件/etc/security/limits.conf，最后添加以下内容。

 soft nofile 65536
 hard nofile 65536
 soft nproc 32000
 hard nproc 32000
 hard memlock unlimited
 soft memlock unlimited


2. 修改文件 /etc/systemd/system.conf ，分别修改以下内容。

DefaultLimitNOFILE=65536
DefaultLimitNPROC=32000
DefaultLimitMEMLOCK=infinity


3. 修改文件/etc/sysctl.conf文件最后添加一行,即可永久修改.

vm.max_map_count=262144

### 容器启动前须知

容器挂载的目录自动生成，启动后会失败，需要修改目录权限

chown -R 1000.0 docker-compose-elasticsearch8-cluster
chmod mod 755 docker-compose-elasticsearch8-cluster

### 证书相关

因为是容器部署需要先准备证书，然后挂在的es和kibana容器，证书生成后copy了一份放在了kibana文件夹下。

通过官方镜像获取证书

1. 先生成ca证书

```
echo -e "\n" | docker run --rm \
  -v $PWD/certs:/certs \
  docker.elastic.co/elasticsearch/elasticsearch:8.18.3 \
  /usr/share/elasticsearch/bin/elasticsearch-certutil cert \
  --ca /certs/elastic-stack-ca.p12 \
  --ca-pass "" \            # CA 没有密码
  --out /certs/elastic-certificates.p12 \
  --pass ""
```

2. 生成集群证书

```
docker run --rm \
  -v $PWD/certs:/certs \
  docker.elastic.co/elasticsearch/elasticsearch:8.18.3 \
  /usr/share/elasticsearch/bin/elasticsearch-certutil cert \
  --ca /certs/elastic-stack-ca.p12 \
  --ca-pass "" \
  --out /certs/elastic-certificates.p12 \
  --pass "" \
  --silent

```

