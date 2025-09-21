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

容器挂载的目录自动生成，启动后会因权限问题启动失败，需要修改目录权限

```
chown -R 1000.0 docker-compose-elasticsearch8-cluster
chmod mod 755 docker-compose-elasticsearch8-cluster

```

### 四. Kibana配置

kibana8的配置跟7区别很大，很多配置废弃了，调试的时候需要多看日志报错。

kibana8通过默认账号kibana_system去连接es集群,最好不要用其他账号链接，可能对索引有影响。

通过elastic账号修改kibana_system的密码，elastic的密码在es的环境变量中可以找到


```
1. 修改账号密码
curl -k -u elastic -X POST 'https://192.168.1.125:9200/_security/user/kibana_system/_password' -H 'Content-Type: application/json' -d'
{
  "password": "NewPassword123!"
}'


2. 验证账号


[root@anolios-01 config]# curl -k -u kibana_system:channel 'https://192.168.1.125:9200/_nodes?filter_path=nodes.*.version,nodes.*.http.publish_address,nodes.*.ip&pretty'
{
  "nodes" : {
    "vlHFIfAnSVKrZbkOFKh0jw" : {
      "ip" : "192.168.64.4",
      "version" : "8.18.3",
      "http" : {
        "publish_address" : "192.168.64.4:9200"
      }
    },
    "v9lRHYMzTOCSIbfTJsKQzA" : {
      "ip" : "192.168.64.3",
      "version" : "8.18.3",
      "http" : {
        "publish_address" : "192.168.64.3:9200"
      }
    },
    "zZ_OuD27RgCHeAXuoAJPSQ" : {
      "ip" : "192.168.64.2",
      "version" : "8.18.3",
      "http" : {
        "publish_address" : "192.168.64.2:9200"
      }
    }
  }
}


3. kibana的配置
调试了半天，懒的改进环境变量了，强迫症的可以改改

# ================= Kibana Server 配置 =================
server.host: "0.0.0.0"
server.port: 5601

# 启用 Kibana HTTPS
server.ssl.enabled: true
server.ssl.keystore.path: /usr/share/kibana/config/certs/elastic-certificates.p12
server.ssl.keystore.password: ""

# ================= Elasticsearch 连接配置 =================
elasticsearch.hosts: ["https://192.168.1.125:9200"]
elasticsearch.username: "kibana_system"
elasticsearch.password: "channel"

# 启用 HTTPS 验证 Elasticsearch 证书
elasticsearch.ssl.certificateAuthorities: [ "/usr/share/kibana/config/certs/elastic-certificates.p12" ]
#elasticsearch.ssl.verificationMode: certificate
elasticsearch.ssl.verificationMode: none

# ================= Kibana 其他配置 =================
logging.root.level: info
xpack.monitoring.ui.container.elasticsearch.enabled: true


xpack.encryptedSavedObjects.encryptionKey: "hG7s9K2bF1dPq4JwzYtLr3Ux8VvM2x9Q"

```


### 后话

所有的配置我都已经改好了，使用的时候拉取下来，清空我es kibana的数据，生成2个证书，放进对应目录，改下es的jvm限制和账号就可直接使用了
