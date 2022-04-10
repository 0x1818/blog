# Hadoop集群HA高可用部署[2NN2RM]



### 1、服务器环境及角色分配

| hostname | IP           | 配置          | **HDFS**                                           | **YARN**                     | zookeeper |
| -------- | ------------ | ------------- | -------------------------------------------------- | ---------------------------- | --------- |
| hadoop1  | 172.16.1.210 | 2c4g100gsata3 | NameNode、DataNode、JournalNode                    | NodeManager、ResourceManager | ZK        |
| hadoop2  | 172.16.1.211 | 2c4g100gsata3 | NameNode、DataNode、SecondaryNameNode、JournalNode | NodeManager、ResourceManager | zk        |
| hadoop3  | 172.16.1.212 | 2c4g100gsata3 | DataNode、JournalNode                              | NodeManager                  | zk        |

- 1.1、系统和各软件版本约定

| 软件      | 版本号                     |
| --------- | -------------------------- |
| Hadoop    | hadoop-3.3.1               |
| zookeeper | apache-zookeeper-3.8.0-bin |
| os        | CentOS Stream 8            |

- 1.2、各节点jps进程

| hostname | jps进程                                                      |
| -------- | ------------------------------------------------------------ |
| hadoop1  | QuorumPeerMain<br/>DFSZKFailoverController<br/>JournalNode<br/> Jps<br/>NodeManager<br/>NameNode<br/>DataNode<br/>ResourceManager |
| hadoop2  | NodeManager<br/>ResourceManager<br/> JournalNode<br/>Jps<br/>NameNode<br/>QuorumPeerMain<br/>DataNode<br/>DFSZKFailoverController |
| hadoop3  | NodeManager<br/> JournalNode<br/>QuorumPeerMain<br/>DataNode<br/>Jps |



### 2、服务器免密配置


# 生成ssh 密钥对,一路回车，不输入密码
[root@hadoop1 ~]# ssh-keygen -t rsa

# 把本地的ssh公钥文件安装到远程主机对应的账户
ssh-copy-id -i .ssh/id_rsa.pub hadoop1
ssh-copy-id -i .ssh/id_rsa.pub hadoop2
ssh-copy-id -i .ssh/id_rsa.pub hadoop3

# 因为namenode仲裁时需要用到sshfence，所以各服务器之间需要免密登录

scp /root/.ssh/id_rsa hadoop1:/root/.ssh/id_rsa
scp /root/.ssh/id_rsa hadoop2:/root/.ssh/id_rsa
scp /root/.ssh/id_rsa hadoop3:/root/.ssh/id_rsa

# 3台服务器配置主机名

cat >> /etc/hosts <<EOF
172.16.1.210 hadoop1
172.16.1.211 hadoop2
172.16.1.212 hadoop3
EOF

# 配置Hadoop环境变量

vi /etc/profile

export HADOOP_HOME=/data/hadoop-3.3.1
export PATH=$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH
export HADOOP_OPTS="-Djava.library.path=$HADOOP_HOME/lib:$HADOOP_HOME/lib/native"

source /etc/profile


### 3、安装JDK


# 此软件会提供jps命令，如果不确定什么软件包可提供此命令，可以使用yum provides jps或yum provides */jps
yum install java-1.8.0-openjdk-devel -y

yum install java-11-openjdk-devel -y

# 验证Java安装是否正常
java -version

# 安装psmisc，因为namenode仲裁时需要用到sshfence，所以此软件的fuser配合。

yum -y install psmisc




### 4、安装zookeeper集群

```bash
cd /data

wget https://dlcdn.apache.org/zookeeper/zookeeper-3.8.0/apache-zookeeper-3.8.0-bin.tar.gz

tar xf apache-zookeeper-3.8.0-bin.tar.gz

# 创建zookeeper数据目录和第一次启动的myid文件,每个节点的id值不一样，节点1的myid为1、节点2的myid为2、以此类推。

mkdir -p /data/apache-zookeeper-3.8.0-bin/data

echo 1 >/data/apache-zookeeper-3.8.0-bin/data/myid

# 修改zookeeper配置文件
cd  /data/apache-zookeeper-3.8.0-bin/conf

cp zoo_sample.cfg zoo.cfg

vi zoo.cfg

dataDir=/data/apache-zookeeper-3.8.0-bin/data
dataLogDir=/data/apache-zookeeper-3.8.0-bin/logs
# 3台节点
server.1=hadoop1:2888:3888
server.2=hadoop2:2888:3888
server.3=hadoop3:2888:3888


# 加入zookeeper到环境变量
export PATH=$PATH:/data/apache-zookeeper-3.8.0-bin/bin
source /etc/profile

# 拷贝zookeeper到其他节点，注意要修改myid的值

cd /data

scp -r apache-zookeeper-3.8.0-bin  hadoop2:/$(pwd)
echo 2 >/data/apache-zookeeper-3.8.0-bin/data/myid

scp -r apache-zookeeper-3.8.0-bin  hadoop2:/$(pwd)
echo 3 >/data/apache-zookeeper-3.8.0-bin/data/myid

# zookeeper启动和停止以及leader查看命令
启动：zkServer.sh start
停止： zkServer.sh stop
查看leader状态：zkServer.sh status

[root@hadoop1 data]# zkServer.sh status
/usr/bin/java
ZooKeeper JMX enabled by default
Using config: /data/apache-zookeeper-3.8.0-bin/bin/../conf/zoo.cfg
Client port found: 2181. Client address: localhost. Client SSL: false.
Mode: follower/leader  【判断此节点是follower还是leader节点】
```



### 5、下载Hadoop二进制软件包并解压

```bash
cd /data

wget https://dlcdn.apache.org/hadoop/common/hadoop-3.3.1/hadoop-3.3.1.tar.gz

tar xf hadoop-3.3.1.tar.gz

```



### 6、配置Hadoop并启动，需要修改的配置文件为workers、core-site.xml、[hdfs](https://so.csdn.net/so/search?q=hdfs&spm=1001.2101.3001.7020)-site.xml、mapred-site.xml、yarn-site.xml

```bash
# 进入Hadoop配置目录

cd /data/hadoop-3.3.1/etc/hadoop

# 1、修改hadoop-env.sh 添加如下内容

export  JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.322.b06-2.el8_5.x86_64

export HDFS_NAMENODE_USER=root
export HDFS_DATANODE_USER=root
export HDFS_SECONDARYNAMENODE_USER=root
export YARN_RESOURCEMANAGER_USER=root
export YARN_NODEMANAGER_USER=root
export HDFS_JOURNALNODE_USER=root
export HDFS_ZKFC_USER=root

# 2、修改核心模块配置文件core-site.xml

<configuration>


	<property>
    		<name>fs.defaultFS</name>
    		<value>hdfs://ns</value>
    		<!-- 这里定义参与federation的子集群别名 -->
	</property>


	<!-- 指定hadoop运行时产生文件的存储目录 -->
	<property>
		<name>hadoop.tmp.dir</name>
		<value>/data/hdata/tmp</value>
	</property>
	<!-- 指定hadoop WEB UI 用户身份 -->
        <property>
                <name>hadoop.http.staticuser.user</name>
                <value>root</value>
        </property>

	<!-- 设置Hadoop的代理用户 -->
	<property>
		<name>hadoop.proxyuser.root.hosts</name>
		<value>*</value>
	</property>

        <property>
                <name>hadoop.proxyuser.root.groups</name>
                <value>*</value>
        </property>

        <!-- 文件系统垃圾桶保存时间 -->
        <property>
                <name>fs.trash.interval</name>
                <value>1440</value>
        </property>

 	<!--指定zookeeper地址-->
 	<property>
      		<name>ha.zookeeper.quorum</name>
      		<value>hadoop1:2181,hadoop2:2181,hadoop3:2181</value>
 	</property>

</configuration>

# 3、修改hdfs文件系统模块配置：hdfs-site.xml

<configuration>

	<!-- 指定HDFS副本的数量，默认为3个 -->
	<property>
		<name>dfs.replication</name>
		<value>3</value>
	</property>

	<property>
    		<name>dfs.nameservices</name>
    		<value>ns</value>
    		<!-- 这里定义参与federation的子集群别名与core-site.xml对应 -->
	</property>
	
	<property>
    		<name>dfs.ha.namenodes.ns</name>
    		<value>nn1,nn2</value>
    		<!-- 这里定义ns的namenode名称，自定义与下面相对应 -->
  	</property>

	<property>
    		<name>dfs.namenode.rpc-address.ns.nn1</name>
    		<value>hadoop1:8020</value>
	</property>

	<property>
    		<name>dfs.namenode.http-address.ns.nn1</name>
    		<value>hadoop1:9870</value>
	</property>

        <property>
                <name>dfs.namenode.rpc-address.ns.nn2</name>
                <value>hadoop2:8020</value>
        </property>

        <property>
                <name>dfs.namenode.http-address.ns.nn2</name>
                <value>hadoop2:9870</value>
        </property>

	<!-- 开启NameNode故障时自动切换 -->
    	<property>
        	<name>dfs.ha.automatic-failover.enabled</name>
        	<value>true</value>
    	</property>
    	<!-- 配置失败自动切换实现方式 -->
    	<property>
            	<name>dfs.client.failover.proxy.provider.ns</name>
            	<value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
    	</property>
    	<!-- 配置隔离机制，如果ssh是默认22端口，value直接写sshfence即可【这里为了保险加了两个，测试过sshfence，不能故障转移】 -->
    	<property>
            	<name>dfs.ha.fencing.methods</name>
             	<value>
					sshfence
					shell(/bin/true)
				</value>
    	</property>


	<!-- 使用隔离机制时需要ssh免登陆 -->
	<property>
		<name>dfs.ha.fencing.ssh.private-key-files</name>
		<value>/root/.ssh/id_rsa</value>
	</property>



    	<!-- 指定NameNode的元数据在JournalNode上的存放位置，建议奇数个节点和zookeeper一样 -->
    	<property>
         	<name>dfs.namenode.shared.edits.dir</name>
         	<value>qjournal://hadoop1:8485;hadoop2:8485;hadoop3:8485/ns</value>
    	</property>
    	<!-- 指定JournalNode在本地磁盘存放数据的位置 -->
    	<property>
        	<name>dfs.journalnode.edits.dir</name>
        	<value>/data/hdata/journal</value>
    	</property>

</configuration>

# 4、修改MapReduce模块配置：mapred-site.xml

<configuration>

	<property>
		<name>mapreduce.framework.name</name>
		<value>yarn</value>
	</property>
	
	<!-- yarn的环境变量 -->

	<property>
		<name>yarn.app.mapreduce.am.env</name>
		<value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
	</property>

	<property>

		<name>mapreduce.map.env</name>
		<value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
	
	</property>

	<property>		
		<name>mapreduce.reduce.env</name>
		<value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
	
	</property>

</configuration>

# 5、修改yarn模块配置：yarn-site.xml

<configuration>

<!-- Site specific YARN configuration properties -->


        <property>
                <name>yarn.nodemanager.aux-services</name>
                <value>mapreduce_shuffle</value>
        </property>


        <!--是否对容器实施物理内存限制 -->
        <property>
                <name>yarn.nodemanager.pmem-check-enabled</name>
                <value>false</value>
        </property>

        <!-- 是否对容器实施虚拟内存限制 -->

        <property>
                <name>yarn.nodemanager.vmem-check-enabled</name>
                <value>false</value>
        </property>

        <!-- 开启日志聚集 -->

        <property>
                <name>yarn.log-aggregation-enable</name>
                <value>true</value>
        </property>


        <!-- 历史日志保留的时间7天 -->

        <property>
                <name>yarn.log-aggregation.retain-seconds</name>
                <value>604800</value>
        </property>


	<!-- yarn ha 高可用配置，是否开启高可用 -->
	<property>
　　　　	<name>yarn.resourcemanager.ha.enabled</name>
　　　　	<value>true</value>
　　	</property>
　　	
　　	<!-- yarn ha 高可用配置，设置集群id -->
　　	<property>
　　　　	<name>yarn.resourcemanager.cluster-id</name>
　　　　	<value>cluster1</value>
　　	</property>
　　	
　　	<!-- yarn ha 高可用配置，设置rm id ,自定义，与下面相对应 -->
　　	<property>
　　　　	<name>yarn.resourcemanager.ha.rm-ids</name>
　　　　	<value>rm1,rm2</value>
　　	</property>
　　	<property>
　　　　	<name>yarn.resourcemanager.hostname.rm1</name>
　　　　	<value>hadoop1</value>
　　	</property>
　　	<property>
　　　　	<name>yarn.resourcemanager.hostname.rm2</name>
　　　　	<value>hadoop2</value>
　　	</property>
　　	<property>
　　　　	<name>yarn.resourcemanager.webapp.address.rm1</name>
　　　　	<value>hadoop1:8088</value>
　　	</property>
　　	<property>
　　　　	<name>yarn.resourcemanager.webapp.address.rm2</name>
　　　　	<value>hadoop2:8088</value>
　　	</property>
　　	
　　	<!-- yarn ha 高可用配置，zookeeper 配置 -->
　　	<property>
　　　　	<name>hadoop.zk.address</name>
　　　　	<value>hadoop1:2181,hadoop2:2181,hadoop3:2181</value>
　　	</property>

</configuration>
# 6、修改workers文件，配置任务运行机器
[root@hadoop1 hadoop]# vi workers 
hadoop1
hadoop2
hadoop3

```



### 7、Hadoop的初次启停命令【有顺序】

```bash
1、首先启动各个节点的Zookeeper，在各个节点上执行以下命令【jps查看进程为：QuorumPeerMain】：
zkServer.sh start

2、在某一个namenode节点执行如下命令，创建命名空间

hdfs zkfc -formatZK

3、在每个journalnode节点用如下命令启动journalnode【按规划是三台节点都启动，jps查看进程为：JournalNode】

hdfs --daemon start journalnode

4、在主namenode节点格式化namenode和journalnode目录

hdfs namenode -format ns

5、在主namenode节点启动namenode进程，【jps查看进程为：NameNode】

hdfs --daemon start namenode  

6、在备namenode节点执行第一行命令，这个是把备namenode节点的目录格式化并把元数据从主namenode节点copy过来，并且这个命令不会把journalnode目录再格式化了！然后用第二个命令启动备namenode进程！

hdfs namenode -bootstrapStandby
hdfs --daemon start namenode

7、在两个namenode【hadoop1和Hadoop2】节点都执行以下命令，【jps查看进程为：DFSZKFailoverController】

hdfs --daemon start zkfc

8、在所有datanode节点都执行以下命令启动datanode，【jps查看进程为：DataNode】

hdfs --daemon start datanode

9、在两个resourcemanager节点启动yarn，【jps查看进程为：ResourceManager】

yarn --daemon start resourcemanager

yarn --daemon stop resourcemanager

```



### 8、Hadoop模块启停命令


```bash
# 1、启动所有模块命令【会启动除zookeeper进程以外的所有Hadoop服务】： start-all.sh

# 2、停止所有模块命令【会停止除zookeeper进程以外的所有Hadoop服务】：stop-all.sh

# 3、分模块启停命令：
hdfs模块启停（start-dfs.sh/stop-dfs.sh）
yarn模块启停（start-yarn.sh/stop-yarn.sh）

# 4、Hadoop历史服务器启动，可以查看MapReduce的执行历史
mr-jobhistory-daemon.sh start historyserver
mr-jobhistory-daemon.sh stop historyserver

# 暴力停止Hadoop集群【不推荐，可能会丢失数据】：

jps |grep -iv jps |awk '{print $1}' |xargs kill -9

```



### 9、Hadoop集群测试

- hdfs测试

  ```bash
  # 上传测试：
  hadoop  fs -put CentOS-8.5.2111-x86_64-boot.iso  /
  
  # 查看
  [root@hadoop1 data]# hadoop fs -ls /
  Found 3 items
  -rw-r--r--   3 root supergroup  827326464 2022-03-18 14:52 /CentOS-8.5.2111-x86_64-boot.iso
  drwx------   - root supergroup          0 2022-03-19 10:29 /tmp
  drwxr-xr-x   - root supergroup          0 2022-03-07 10:23 /user
  
  # 下载测试：
  hadoop fs -get /CentOS-8.5.2111-x86_64-boot.iso  .
  
  ```

  

- MapReduce测试

  ```bash
  # 利用Hadoop自带的测试用例测试 pi 圆周率
  cd  /data/hadoop-3.3.1/share/hadoop/mapreduce
  
  # MapReduce测试命令：
  hadoop jar hadoop-mapreduce-examples-3.3.1.jar  pi 10  10 
  ```

- namenode HA 故障转移测试

  ```bash
  # 查看namenode的状态：
  [root@k8s-node1 home]# hdfs haadmin -getServiceState nn1
  active
  [root@k8s-node1 home]# hdfs haadmin -getServiceState nn2
  standby
  
  [root@hadoop1]# hdfs --daemon start namenode 【杀掉主namenode的进程】
  
  # 查看从namenode的状态
  [root@k8s-node1 home]# hdfs haadmin -getServiceState nn2
  active
  
  # 启动namenode进程
  
  hdfs --daemon start namenode
  
  [root@k8s-node1 home]# hdfs haadmin -getServiceState nn1
  standby
  ```

- yarn  HA 故障转移测试

  ```bash
  # 查看yarn的resourcemanager状态
  [root@hadoop1]# yarn rmadmin -getServiceState rm1
  active
  [root@hadoop1]# yarn rmadmin -getServiceState rm2
  standby
  
  # 停止“active”主机的 resourcemanager：
  yarn --daemon stop resourcemanager
  
  # 查看yarn的resourcemanager状态
  [root@hadoop1]# yarn rmadmin -getServiceState rm2
  active
  
  # 启动resourcemanager
  
  [root@hadoop1]# yarn --daemon start resourcemanager
  [root@hadoop1]# yarn rmadmin -getServiceState rm1
  standby
  
  
  ```

  

### 10、Hadoop WEB页面访问

```bash
# 1、hdfs WEB界面访问地址：http://172.16.1.210:9870
# 2、yarn WEB界面访问地址：http://172.16.1.210:8088


feiyu:1OqFKWaY0AIRWwVf
```

### 11、优化相关

- hdfs数据块大小设置依据【数据块的大小主要影响寻址】，生产建议3副本.

- hdfs块大小参考设置

  | 磁盘读写     | hdfs数据块大小设置 |
  | ------------ | ------------------ |
  | 100MB/S      | 128M               |
  | 200-300MB/s  | 256M               |
  | 400-1000MB/s | 512M               |

  

### 附录：

1、Hadoop生态圈介绍：https://blog.csdn.net/yue_2018/article/details/89084266

2、Hadoop相关服务端口：https://blog.csdn.net/qq_31454379/article/details/105439752

3、服务器端口与各配置文件参数详解：https://www.cnblogs.com/yoyo1216/p/12848926.html
