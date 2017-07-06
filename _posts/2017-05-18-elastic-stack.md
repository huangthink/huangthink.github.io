---
layout: post
title: Elastic Stack安装教程
date: 2017-05-18 12:30:28.000000000 +09:00
tags: [能工巧匠]
---

### 环境准备

- CentOS7.1
- JDK1.8
- elasticsearch-5.4.3

### JDK安装配置

下载地址：http://download.oracle.com/otn-pub/java/jdk/8u111-b14/jdk-8u111-linux-x64.tar.gz

### elasticsearch安装配置
#### 下载解压

使用命令从官网下载最新版本的Elasticsearch压缩包

```
cd /var/wd
wget -N https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.4.3.tar.gz
tar -zxf elasticsearch-5.4.3.tar.gz
ln -s lasticsearch-5.4.3 elasticsearch
mkdir -p /var/data/elasticsearch
mkdir -p /var/logs/elasticsearch
```
#### 修改配置文件

```
vi elasticsearch/config/elasticsearch.yml

cluster.name: elasticstack
path.data: /var/data/elasticsearch
path.logs: /var/logs/elasticsearch
network.host: 0.0.0.0
http.port: 11200
transport.tcp.port: 11300
discovery.zen.ping.unicast.hosts: ["10.213.162.77", "10.213.162.78", "10.213.162.79"]
discovery.zen.minimum_master_nodes: 3
http.cors.enabled: true
http.cors.allow-origin: "*"
```
#### 创建elasticsearch用户组以及elasticsearch用户

```
groupadd elasticsearch
useradd  elasticsearch -g elasticsearch -p elasticsearch
```
#### 更改相关文件夹以及内部文件的所属用户以及组为elasticsearch

```
chown -R elasticsearch:elasticsearch /var/wd/elasticsearch-5.4.3
chown -R elasticsearch:elasticsearch /var/data/elasticsearch
chown -R elasticsearch:elasticsearch /var/logs/elasticsearch
```

#### 修改服务器相关参数

```
修改vm.map 限制
sysctl -w vm.max_map_count=262144
或
vi /etc/sysctl.conf
vm.max_map_count=262144

修改文件限制
ulimit -n 102400
或
vi /etc/security/limits.conf
elasticsearch hard nofile 102400
elasticsearch soft nofile 102400

```

#### 切换到elasticsearch用户下启动

```
su elasticsearch
cd /var/wd/elasticsearch
./bin/elasticsearch
```
如果想在后台以守护进程模式运行

打开另一个终端进行测试

```
curl  'http://10.213.162.77:11200'
```
你能看到以下返回信息：

```
{
    name: "10.213.162.77",
    cluster_name: "elasticstack",
    cluster_uuid: "aHySi7cBS76DyrIe74eIoA",
    version: {
        number: "5.4.3",
        build_hash: "5395e21",
        build_date: "2016-12-06T12:36:15.409Z",
        build_snapshot: false,
        lucene_version: "6.3.0"
    },
    tagline: "You Know, for Search"
}
```

ELasticsearch集群已经启动并且正常运行。

### head安装配置

#### 安装node

```
cd /usr/local
wget -N https://nodejs.org/dist/v7.2.0/node-v7.2.0-linux-x64.tar.gz
tar -zxf node-v7.2.0-linux-x64.tar.gz
n -s node-v7.2.0-linux-x64/ node
vi /etc/profile
export PATH=$PATH:/usr/local/node/bin
source /etc/profile
```
#### 安装grunt

```
npm install -g grunt-cli
```

#### 安装head

```
git clone git://github.com/mobz/elasticsearch-head.git

```
##### 修改Gruntfile.js

```
connect: {
    server: {
        options: {
            port: 11100,
            hostname: '*',
            base: '.',
            keepalive: true
        }
    }
}
```

##### 修改app.js

```
this.base_uri = this.config.base_uri || this.prefs.get("app-base_uri") || "http://10.213.162.77:11200";
```

##### 运行head

```
npm install 
grunt server
```
### Logstash安装配置

### Kibana安装配置

使用命令从官网下载最新版本的Elasticsearch压缩包，按照文档的要求，一般情况下kibana的版本必须和Elasticsearch安装的版本一致。

```
cd /var/wd
wget -N https://artifacts.elastic.co/downloads/kibana/kibana-5.4.3-linux-x86_64.tar.gz
tar -zxf kibana-5.4.3-linux-x86_64.tar.gz
ln -s kibana-5.4.3 kibana
```

```
vi kibana/config/kibana.yml

server.port: 11201
server.host: "0.0.0.0"
elasticsearch.url: "http://10.213.162.78:11200"
elasticsearch.username: "elastic"
elasticsearch.password: "elastic"
```
### X-Pack安装配置

X-Pack是一个Elastic Stack的扩展，将安全，警报，监视，报告和图形功能包含在一个易于安装的软件包中。在Elasticsearch 5.0.0之前，您必须安装单独的Shield，Watcher和Marvel插件才能获得在X-Pack中所有的功能。

#### Elasticsearch下载X-Pack

在es的根目录（每个节点），运行 ```bin/elasticsearch-plugin``` 进行安装

```
bin/elasticsearch-plugin install x-pack
```

如果你在Elasticsearch已禁用自动索引的创建，在```elasticsearch.yml```配置```action.auto_create_index```允许X-pack创造以下指标

```
action.auto_create_index: .security,.monitoring*,.watches,.triggered_watches,.watcher-history*
```

#### Kibana下载X-Pack

在Kibana根目录运行 ```bin/kibana-plugin``` 进行安装

```
bin/kibana-plugin install x-pack
```

#### 验证X-Pack

在浏览器上输入： ```http://10.213.162.77:11201``` ，可以打开Kibana，此时需要输入用户名和密码登录，默认分别是 ```elastic``` 和 ```changeme```

### 安装参考

1. 每个操作系统安装Elasticsearch的文件选择不同，参考：```https://www.elastic.co/downloads/elasticsearch```，选择对应的文件下载。
2. 安装Kiabna需要根据操作系统做选择，参考：```https://www.elastic.co/guide/en/kibana/current/install.html```，选择对应的文件下载。
3. 安装X-Pack需要根据Elasticsearch安装不同的方式提供不同的安装方法，参考：```https://www.elastic.co/guide/en/x-pack/5.4/installing-xpack.html#installing-xpack```。
4. 离线安装 

```
./bin/elasticsearch-plugin install file:///var/wd/x-pack-5.4.3.zip
./bin/kibana-plugin install file:///var/wd/x-pack-5.4.3.zip
```

| App    | URL                          |
| ------------ | -----------------------------|
| x-pack   | https://artifacts.elastic.co/downloads/packs/x-pack/x-pack-5.4.3.zip          |
