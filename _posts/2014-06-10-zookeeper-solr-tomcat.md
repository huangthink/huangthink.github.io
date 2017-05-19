---
layout: post
title: "Zookeeper、Solr和Tomcat安装配置实践"
date: 2014-06-10 21:38:02 +0800
comments: true
categories: blog
tags: [能工巧匠]
image:
  feature: abstract-7.jpg
---

### 三台服务器:

* 192.168.19.210(myid=210) master
* 192.168.19.211(myid=211) slave1
* 192.168.19.212(myid=212) slave2

<!-- more -->

### ZooKeeper集群配置

安装ZooKeeper集群，在上面3分节点上分别安装，使用的版本是zookeeper-3.4.5。首先在master上安装配置：
	
{% highlight bash %}
cd /tmp
wget -N http://mirrors.cnnic.cn/apache/zookeeper/zookeeper-3.4.5/zookeeper-3.4.5.tar.gz
tar -zxf zookeeper-3.4.5.tar.gz
mv ./zookeeper-3.4.5 /opt/zookeeper
mkdir /opt/zookeeper/data
mkdir /opt/zookeeper/logs
	
echo 'tickTime=2000' > /opt/zookeeper/conf/zoo.cfg
echo 'initLimit=10' >> /opt/zookeeper/conf/zoo.cfg
echo 'syncLimit=5' >> /opt/zookeeper/conf/zoo.cfg
echo 'dataDir=/opt/zookeeper/data' >> /opt/zookeeper/conf/zoo.cfg
echo 'dataLogDir=/opt/zookeeper/logs' >> /opt/zookeeper/conf/zoo.cfg
echo 'clientPort=2181' >> /opt/zookeeper/conf/zoo.cfg
echo 'server.210=master:2888:3888' >> /opt/zookeeper/conf/zoo.cfg
echo 'server.211=slave1:2888:3888' >> /opt/zookeeper/conf/zoo.cfg
echo 'server.212=slave2:2888:3888' >> /opt/zookeeper/conf/zoo.cfg
echo '210' > /opt/zookeeper/data/myid
{% endhighlight %}

然后将master上的zookeeper复制到其他两个节点上：

```
scp -r /opt/zookeeper root@slave1:/opt/
scp -r /opt/zookeeper root@slave2:/opt/
```

修改slave1、slave2的myid文件：

`vi /opt/zookeeper/data/myid`

其他设置：
	
{% highlight bash %}
开机启动
echo '/opt/zookeeper/bin/zkServer.sh start' >> /etc/rc.d/rc.local
环境变量
echo 'export PATH=$PATH:/opt/zookeeper/bin' >> /etc/profile
source /etc/profile
ZooKeeper服务命令
/opt/zookeeper/bin/zkServer.sh start
/opt/zookeeper/bin/zkServer.sh status
/opt/zookeeper/bin/zkServer.sh stop
/opt/zookeeper/bin/zkServer.sh restart
{% endhighlight %}
### SolrCloud配置

##### 首先在一个节点上对SOLR进行配置，我们选择master节点。

{% highlight bash %}
cd /tmp
wget -N http://archive.apache.org/dist/lucene/solr/4.2.1/solr-4.2.1.tgz
tar -zxf solr-4.2.1.tgz
mkdir -p /opt/solr/webapps
cd /opt/solr/webapps
jar -xf /tmp/solr-4.2.1/example/webapps/solr.war
mkdir -p /opt/solr/home
{% endhighlight %} 
vi /opt/solr/home/solr.xml
{% highlight xml %}
<?xml version="1.0" encoding="UTF-8" ?>
<solr persistent="true">
    <cores defaultCoreName="Designer" adminPath="/admin/cores" host="${host:}" hostPort="${jetty.port:}" zkClientTimeout="${zkClientTimeout:15000}">
	 </cores>
</solr>
{% endhighlight %} 
{% highlight bash %}
 #master's lib
ln -s /opt/solr/webapps/WEB-INF/lib /opt/solr/lib
 #master's conf
mkdir -p /opt/solr/conf
{% endhighlight %} 
注意：这里并没有配置任何的core元素，等到整个配置安装完成之后，通过SOLR提供的REST接口，来实现Collection以及Shard的创建，从而来更新这些配置文件。

#### ZooKeeper管理监控配置文件:

{% highlight java %}
java -cp .:/opt/solr/lib/* org.apache.solr.cloud.ZkCLI -cmd upconfig -zkhost master:2181,slave1:2181,slave1:2181 -confdir /opt/solr/conf/Designer -confname Designer -solrhome /opt/solr/home
{% endhighlight %} 

检查一下ZooKeeper上的存储情况：

{% highlight bash %}
cd /opt/zookeeper
bin/zkCli.sh -server master:2181
ls /
ls /configs
{% endhighlight %} 

#### Tomcat配置与启动:

{% highlight bash %}
mkdir -p /opt/tomcat/conf/Catalina/localhost
vi /opt/tomcat/conf/Catalina/localhost/solr.xml
{% endhighlight %} 

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8" ?>
<Context docBase="/opt/solr/webapps" reloadable="true">
	  <Environment name="solr/home" type="java.lang.String" value="/opt/solr/home" override="true" />
</Context>
{% endhighlight %} 

{% highlight java %}	
	#tomcat catelina.sh
	vi /opt/tomcat/bin/catalina.sh
	JAVA_OPTS="
	 -Xmx6G
	 -Xms6G
	 -Xss256K
	 -Xmn2300M
	 -XX:SurvivorRatio=5
	 -XX:+UseParNewGC
	 -XX:PermSize=128M
	 -XX:MaxPermSize=128M
	 -XX:MaxDirectMemorySize=768m
	 -XX:LargePageSizeInBytes=128m
	 -XX:+DisableExplicitGC
	 -XX:SoftRefLRUPolicyMSPerMB=0
	 -XX:+OptimizeStringConcat
	 -XX:+UseFastAccessorMethods
	 -XX:CompileThreshold=100
	 -XX:+AggressiveOpts
	 -XX:+TieredCompilation
	 -XX:+DoEscapeAnalysis
	 -XX:+UseBiasedLocking
	 -XX:+EliminateLocks
	 -XX:BiasedLockingStartupDelay=0
	 -XX:+UseConcMarkSweepGC
	 -XX:CMSInitiatingOccupancyFraction=70
	 -XX:+UseCMSCompactAtFullCollection
	 -XX:CMSFullGCsBeforeCompaction=2
	 -XX:+CMSParallelRemarkEnabled
	 -XX:+UseCMSInitiatingOccupancyOnly
	 -XX:+CMSClassUnloadingEnabled
	 -XX:+CMSPermGenPrecleaningEnabled
	 -Dsun.net.inetaddr.ttl=0
	 -Dnetworkaddress.cache.ttl=30
	 -Djava.awt.headless=true
	 -DAllowManagedFieldsInDefaultFetchGroup=true
	 -DAllowMediatedWriteInDefaultFetchGroup=true
	 
	 -Djetty.port=8080
	 -DzkHost=master:2181,slave1:2181,slave2:2181
	 -DzkClientTimeout=15000
	"
{% endhighlight %} 

tomcat 常用命令：

{% highlight bash %}
 #启动
/opt/tomcat/bin/startup.sh
 #查看日志
tail -f /opt	/tomcat/logs/catalina.out
{% endhighlight %} 
查看一下ZooKeeper中的数据状态：

{% highlight bash %}
cd /opt/zookeeper
bin/zkCli.sh -server master:2181
ls /
ls /live_nodes
ls /collections
{% endhighlight %} 

这时候，SolrCloud集群中只有一个活跃的节点，而且默认生成了一个Designer实例，这个实例实际上虚拟的，因为通过web界面无法访问master:8080/solr，看不到任何有关SolrCloud的信息.

#### 同步数据和配置信息，启动其他节点

{% highlight bash %}
scp -r /opt/tomcat/ root@slave1:/opt/
scp -r /opt/tomcat/ root@slave2:/opt/
scp -r /opt/solr/ root@slave1:/opt/
scp -r /opt/solr/ root@slave2:/opt/
{% endhighlight %} 
启动其他Solr服务器节点：
	
`/opt/tomcat/bin/startup.sh`

查看ZooKeeper集群中数据状态：
	
`ls /live_nodes`

这时已经存在3个活跃的节点了，但是SolrCloud集群并没有更多信息.
#### 创建Collection、Shard和Replication
#### 创建Collection及初始Shard
直接通过REST接口来创建Collection:

{% highlight bash %}
[root@master]$ curl 'http://master:8080/solr/admin/collections?action=CREATE&name=Article&numShards=3&replicationFactor=1'
{% endhighlight %} 
如果成功，会输出如下响应内容：
	
{% highlight xml %}
    <?xml version="1.0" encoding="UTF-8"?>
	<response>
		<lst name="responseHeader">
			<int name="status">0</int>
			<int name="QTime">4103</int>
		</lst>
		<lst name="success">
			<lst>
				<lst name="responseHeader">
					<int name="status">0</int>
					<int name="QTime">3367</int>
				</lst>
				<str name="core">Article_shard2_replica1</str>
				<str name="saved">/opt/solr/home/solr.xml</str>
			</lst>
			<lst>
				<lst name="responseHeader">
					<int name="status">0</int>
					<int name="QTime">3280</int>
				</lst>
				<str name="core">Article_shard1_replica1</str>
				<str name="saved">/opt/solr/home/solr.xml</str>
			</lst>
			<lst>
				<lst name="responseHeader">
					<int name="status">0</int>
					<int name="QTime">3690</int>
				</lst>
				<str name="core">Article_shard3_replica1</str>
				<str name="saved">/opt/solr/home/solr.xml</str>
			</lst>
		</lst>
	</response>
{% endhighlight %} 
上面链接中的几个参数的含义，说明如下：

> name                待创建Collection的名称
> 
> numShards           分片的数量
> 
> replicationFactor   复制副本的数量

执行上述操作如果没有异常，已经创建了一个Collection，名称为Article，而且每个节点上存在一个分片。这时，也可以查看ZooKeeper中状态：
	
`ls /collections`
`ls /collections/Article`

可以通过Web管理页面，访问master:8080/solr/#/~cloud，查看SolrCloud集群的分片信息.
我们从master节点可以看到，SOLR的配置文件内容，已经发生了变化，如下所示：

`cat /opt/solr/home/solr.xml`

{% highlight xml %}
	<?xml version="1.0" encoding="UTF-8" ?>
	<solr persistent="true">
		<cores defaultCoreName="Designer" host="${host:}" adminPath="/admin/cores" zkClientTimeout="${zkClientTimeout:15000}" hostPort="${jetty.port:}">
				<core loadOnStartup="true" shard="shard3" instanceDir="Article_shard3_replica1/" transient="false" name="Article_shard3_replica1" collection="Article" />
		</cores>
	</solr>
{% endhighlight %} 

#### 创建Replication
下面对已经创建的初始分片进行复制。 shard1已经在slave1上，我们复制分片到master和slave2上，执行如下命令：

{% highlight bash %}
curl 'http://master:8080/solr/admin/cores?action=CREATE&collection=Aritcle&name=Aritcle_shard1_replica_2&shard=shard1'
	<?xml version="1.0" encoding="UTF-8"?>
	<response>
		<lst name="responseHeader">
			<int name="status">0</int>
			<int name="QTime">1485</int>
		</lst>
		<str name="core">Aritcle_shard1_replica_2</str>
		<str name="saved">/opt/solr/home/solr.xml</str>
	</response>
	
curl 'http://master:8080/solr/admin/cores?action=CREATE&collection=Aritcle&name=Aritcle_shard1_replica_3&shard=shard1'
	<?xml version="1.0" encoding="UTF-8"?>
	<response>
		<lst name="responseHeader">
			<int name="status">0</int>
			<int name="QTime">2543</int>
		</lst>
		<str name="core">Aritcle_shard1_replica_3</str>
		<str name="saved">/opt/solr/home/solr.xml</str>
	</response>
	
curl 'http://master:8080/solr/admin/cores?action=CREATE&collection=Aritcle&name=Aritcle_shard1_replica_4&shard=shard1'
	<?xml version="1.0" encoding="UTF-8"?>
	<response>
		<lst name="responseHeader">
			<int name="status">0</int>
			<int name="QTime">2405</int>
		</lst>
		<str name="core">Aritcle_shard1_replica_4</str>
		<str name="saved">/opt/solr/home/solr.xml</str>
	</response>
{% endhighlight %} 
最后的结果是，slave1上的shard1，在master节点上有2个副本，名称为Article_shard1_replica_2和Article_shard1_replica_3，在slave2节点上有一个副本，名称为Article_shard1_replica_4. 也可以通过查看master和slave2上的目录变化，如下所示：

`ll /opt/solr/conf/`
```
总用量 24

drwxrwxr-x. 4 root root 4096 6月   1 09:58 Designer
drwxrwxr-x. 3 root root 4096 6月   1 15:41 Article_shard1_replica_2
drwxrwxr-x. 3 root root 4096 6月   1 15:42 Article_shard1_replica_3 
drwxrwxr-x. 3 hadoop root 4096 6月   1 15:23 Article_shard3_replica1
```
	
`ll /opt/solr/conf/`
```
总用量 20

drwxrwxr-x. 4 root root 4096 6月   1 14:53 Designer
drwxrwxr-x. 3 root root 4096 6月   1 15:44 Article_shard1_replica_4
drwxrwxr-x. 3 root root 4096 6月   1 15:23 Article_shard2_replica1
```

其中，Article_shard3_replica1和Article_shard2_replica1都是创建Collection的时候自动生成的分片，也就是第一个副本。 通过Web界面，可以更加直观地看到shard1的情况.

我们再次从master节点可以看到，SOLR的配置文件内容，又发生了变化，如下所示：

`cat /opt/solr/home/solr.xml`

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8" ?>
<solr persistent="true">
	<cores defaultCoreName="Designer" host="${host:}" adminPath="/admin/cores" zkClientTimeout="${zkClientTimeout:15000}" hostPort="${jetty.port:}">
		<core loadOnStartup="true" shard="shard3" instanceDir="Article_shard3_replica1/" transient="false" name="Article_shard3_replica1" collection="Article" />
		<core loadOnStartup="true" shard="shard1" instanceDir="Article_shard1_replica_2/" transient="false" name="Article_shard1_replica_2" collection="Article" />
		<core loadOnStartup="true" shard="shard1" instanceDir="Article_shard1_replica_3/" transient="false" name="Article_shard1_replica_3" collection="Article" />
	</cores>
</solr>
{% endhighlight %} 
到此为止，我们已经基于3个物理节点，配置完成了SolrCloud集群。

