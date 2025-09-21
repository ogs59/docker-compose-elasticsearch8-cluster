## docker-compose-elasticsearch8-cluster
docker-compose部署es8集群证书版

### 一. 系统配置修改

1. 修改文件/etc/security/limits.conf，最后添加以下内容。

``` 
 soft nofile 65536
 hard nofile 65536
 soft nproc 32000
 hard nproc 32000
 hard memlock unlimited
 soft memlock unlimited
```

2. 修改文件 /etc/systemd/system.conf ，分别修改以下内容。

```
DefaultLimitNOFILE=65536
DefaultLimitNPROC=32000
DefaultLimitMEMLOCK=infinity
```

3. 修改文件/etc/sysctl.conf文件最后添加一行,即可永久修改.

```
vm.max_map_count=262144
```

### 二. 准备证书

因为是容器部署没在容器中申请证书，提前用官方的docker镜像准备好，放进对应es和kibana对应的目录在启动容器
证书分别放入es和kibana的目录，最好不要跨容器目录挂载。


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
生成好的证书挂载进容器即可

es要增加相应的证书配置，已经增加到容器变量

```
    environment:
      - node.name=es01
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es02,es03
      - cluster.initial_master_nodes=es01,es02,es03
      - network.host="0.0.0.0"
      - http.cors.enabled=true
      - http.cors.allow-origin="*"
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms2048m -Xmx2048m"
      - ELASTIC_PASSWORD=channel
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=true
      - xpack.security.http.ssl.keystore.path=certs/elastic-certificates.p12
      - xpack.security.http.ssl.keystore.password=""
      - xpack.security.transport.ssl.enabled=true
      - xpack.security.transport.ssl.verification_mode=certificate
      - xpack.security.transport.ssl.keystore.path=certs/elastic-certificates.p12
      - xpack.security.transport.ssl.truststore.path=certs/elastic-certificates.p12
      - xpack.security.transport.ssl.keystore.password=""
      - xpack.security.transport.ssl.truststore.password=""
```


### 三. 容器启动前须知

容器挂载的目录自动生成，启动后会失败，需要修改目录权限

```
chown -R 1000.0 docker-compose-elasticsearch8-cluster
chmod mod 755 docker-compose-elasticsearch8-cluster

```
