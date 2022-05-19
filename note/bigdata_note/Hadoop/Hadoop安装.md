## Hadoop安装

### 一、伪分布式安装

![img](https://img.mukewang.com/wiki/5f3f6cf609888b6e07430686.jpg)

- 这张图代表一台Linux机器（节点）,其中NameNode、Secondary namenode、DataNode是HDFS服务的进程；ResourceManager、NodeManager是Yarn服务的进程

#### 1.配置基础环境

##### （1）设置静态Ip

```shell
vi /etc/sysconfig/network-scripts/ifcfg-ens33
##将BOOTPROTO设置为="static",IPADDR为IP地址，GATEWAY与DNS1设置为网关地址
```

##### （2）设置永久主机名

```shell
vi /etc/hostname
```

##### （3）设置IP与主机(hostname)映射关系

```shell
vi /etc/hosts
##192.168.192.100 bigdata01
```

##### （4）关闭防火墙

```shell
systemctl stop firewalld ##临时关闭
systemctl disable firewalld ##永久关闭
systemctl status firewalld ##查看状态
```

##### （5）ssh免密登录

- ssh是secure shell，安全的shell
- ssh加密有两种，对称加密与非对称加密，非对称加密的解密过程是不可逆的；非对称加密会产生秘钥，秘钥分为公钥和私钥，公钥是对外公开的，私钥是自己持有的
- ssh通信过程：通信时第一台机器会将自己的公钥给到第二台机器，当第一台机器要给第二台机器通信时，第一台机器会给第二台机器发送一个随机字符串，第二台机器会使用公钥对这个字符串进行加密，同时第一台机器会使用自己的私钥也对这个字符串加密，然后也传给第二台机器；这样第二台机器有了两份加密的内容，一份是自己使用公钥加密的，一份是第一台机器使用私钥加密传过来的，第二台机器会对比这两份加密之后的内容，如果匹配则可信，反之认为非法机器

```shell
ssh-keygen -t rsa##连续按回车即可,不需要输入任何内容
##rsa表示一种加密算法
```

```shell
##执行完之后会在~/.ssh产生对应的公钥和私钥
##例如
[root@bigdata01 ~]# ll ~/.ssh/
total 12
-rw-------. 1 root root 1679 Apr  7 16:39 id_rsa
-rw-r--r--. 1 root root  396 Apr  7 16:39 id_rsa.pub
-rw-r--r--. 1 root root  203 Apr  7 16:21 known_hosts
```

```shell
##将公钥拷贝到需要免密登录的机器上面
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```

#### 2.安装jdk

##### （1）创建软件存放目录

```shell
mkdir -p /data/soft##-p:确保目录名称存在，不存在的就建一个
```

##### （2）安装rzsz上传

```shell
yum install lrzsz -y
##rz:从本地上传到服务器
##sz:从服务器上传到本地
```

##### （3）解压安装包

```shell
tar -zxvf 安装包 -C 目录 
##-C：解压到相关目录 
##-z:-z或--gzip或--ungzip 通过gzip指令处理备份文件
##-x:-x或--extract或--get 从备份文件中还原文件 
##-v:-v或--verbose 显示指令执行过程 
##-f:-f<备份文件>或--file=<备份文件> 指定备份文件
##其他指令详见：https://www.runoob.com/linux/linux-comm-tar.html
```

##### （4）配置环境变量JAVA_HOME

```shell
vi /etc/profile
##添加内容
export JAVA_HOME=/data/soft/jdk1.8
export PATH=.:$JAVA_HOME/bin:$PATH
##.:$JAVA_HOME/bin:$PATH:在保留原来的PATH环境变量的基础上增加$JAVA_HOME/bin这个路径作为环境变量；$代表引用变量，.代表当前用户目录，:代表分隔符，类似windowPATH环境变量的;号
```

##### （5）加载环境变量

```shell
source /etc/profile
java -version##验证
```

#### 3.安装hadoop

##### （1）解压安装包与配置环境变量

```shell
tar -zxvf hadoop安装包
vi /etc/profile ##配置环境变量
##增加内容
export JAVA_HOME=/data/soft/jdk1.8
export HADOOP_HOME=/data/soft/hadoop-3.2.0
export PATH=.:$JAVA_HOME/bin:$HADOOP_HOME/sbin:$HADOOP_HOME/bin:$PATH
##加载环境变量
source /etc/profile
##验证
hadoop version

##hadoop下bin目录下脚本主要为了操作hadoop集群中的hdfs和yarn组件
##hadoop下sbin目录下脚本主要是负责启动或者停止集群中的组件
```

##### （2）修改hadoop相关配置文件

```shell
cd /data/soft/hadoop-3.2.0/etc/hadoop ##进入到hadoop安装目录下etc/hadoop目录
```

- 修改hadoop-env.sh文件

```shell
##增加环境变量信息
[root@bigdata01 hadoop]# vi hadoop-env.sh
.......
export JAVA_HOME=/data/soft/jdk1.8
export HADOOP_LOG_DIR=/data/hadoop_repo/logs/hadoop

##JAVA_HOME：指定java的安装位置
##HADOOP_LOG_DIR：hadoop的日志的存放目录
```

- 修改core-site.xml文件

```shell
[root@bigdata01 hadoop]# vi core-site.xml
<configuration>
	#<!--配置NameNode的主机名和端口号-->
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://bigdata01:9000</value>#<!--无论多少节点，此节点是唯一的，作为NameNode主节点,bigdata01为主机名-->
    </property>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/data/hadoop_repo</value>
   </property>
</configuration>
##注意 fs.defaultFS 属性中的主机名需要和配置的主机名保持一致
```

- 修改hdfs-site.xml文件

```shell
[root@bigdata01 hadoop]# vi hdfs-site.xml
<configuration>
	#设置hdfs副本数
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
    #<!--设置其他用户执行操作时会提醒没有权限问题-->
    <property>
         <name>dfs.permissions</name>
         <value>false</value>
    </property>
</configuration>
##把hdfs中文件副本的数量设置为1，因为现在伪分布集群只有一个节点
```

- 修改mapred-site.xml，设置mapreduce使用的资源调度框架

```xml
[root@bigdata01 hadoop]# vi mapred-site.xml
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
</configuration>
```

- 修改yarn-site.xml，设置yarn上支持运行的服务和环境变量白名单

```shell
[root@bigdata01 hadoop]# vi yarn-site.xml
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.nodemanager.env-whitelist</name>
   <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_MAPRED_HOME</value>
    </property>
</configuration>

```

- 修改workers，设置集群中从节点的主机名信息，在这里就一台集群，所以就填写bigdata01即可

```shell
[root@bigdata01 hadoop]# vi workers
bigdata01
```


##### （3）修改sbin目录下脚本

```shell
##修改sbin目录下的start-dfs.sh、stop-dfs.sh、start-yarn.sh、stop-yarn.sh脚本，增加一下内容
[root@bigdata01 hadoop-3.2.0]# cd sbin/
[root@bigdata01 sbin]# vi start-dfs.sh
HDFS_DATANODE_USER=root
HDFS_DATANODE_SECURE_USER=hdfs
HDFS_NAMENODE_USER=root
HDFS_SECONDARYNAMENODE_USER=root
[root@bigdata01 sbin]# vi stop-dfs.sh
HDFS_DATANODE_USER=root
HDFS_DATANODE_SECURE_USER=hdfs
HDFS_NAMENODE_USER=root
HDFS_SECONDARYNAMENODE_USER=root

[root@bigdata01 sbin]# vi start-yarn.sh
YARN_RESOURCEMANAGER_USER=root
HADOOP_SECURE_DN_USER=yarn
YARN_NODEMANAGER_USER=root
[root@bigdata01 sbin]# vi stop-yarn.sh
YARN_RESOURCEMANAGER_USER=root
HADOOP_SECURE_DN_USER=yarn
YARN_NODEMANAGER_USER=root
```

##### （4）格式化HDFS

```shell
hdfs namenode -format
##配置文件修改好了，但是不能直接启动，因为Hadoop中的HDFS是一个分布式的文件系统，文件系统在使用之前是需要先格式化的，就类似买一块新的磁盘，在安装系统之前需要先格式化才可以使用

##注意：格式化操作只能执行一次，如果格式化的时候失败了，可以修改配置文件后再执行格式化，如果格式化成功了就不能再重复执行了，否则集群就会出现问题。

##如果确实需要重复执行，那么需要把/data/hadoop_repo目录中的内容全部删除，再执行格式化
```

##### （5）启动hadoop集群

```shell
start-all.sh
```

##### （6）验证集群是否启动

- HDFS webui界面：http://192.168.192.100:9870
- YARN webui界面：http://192.168.192.100:8088

```shell
##执行jps命令可以查看集群的进程信息，去掉Jps这个进程之外还需要有5个进程才说明集群是正常启动的
[root@bigdata01 hadoop-3.2.0]# jps
3267 NameNode
3859 ResourceManager
3397 DataNode
3623 SecondaryNameNode
3996 NodeManager
4319 Jps
```

### 二、分布式集群安装

- 其他基础环境，Java环境如上，一下是不同的安装地方
- 以三台机器作为基准，三台机器的基础环境、java环境、hadoop环境一致及环境变量都需要配置

#### 1.配置基础环境

##### （1）配置ip与主机映射关系

```shell
[root@bigdata01 ~]# vi /etc/hosts
192.168.192.100 bigdata01
192.168.192.101 bigdata02
192.168.192.102 bigdata03
##其余两台机器也是一样的操作
```

##### （2）集群节点时间同步

```shell
##先下载ntpdata
yum install -y ntpdata
##执行命令
[root@bigdata01 ~]# ntpdate -u ntp.sjtu.edu.cn
##放入到crontab定时器中，每分钟执行一次
[root@bigdata01 ~]# vi /etc/crontab
* * * * * root /usr/sbin/ntpdate -u ntp.sjtu.edu.cn
##其余两台机器也是一样的操作
```

##### （3）集群之间实现免密

```shell
[root@bigdata01 ~]#ssh-keygen -t rsa
[root@bigdata01 ~]#cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
##将公钥发送到其他两台机器上
[root@bigdata01 ~]#scp ~/.ssh/authorized_keys bigdata02:~/
[root@bigdata01 ~]#scp ~/.ssh/authorized_keys bigdata03:~/
##其他两台机器执行
[root@bigdata02 ~]# cat ~/authorized_keys  >> ~/.ssh/authorized_keys
[root@bigdata03 ~]# cat ~/authorized_keys  >> ~/.ssh/authorized_keys
##验证是否可以免密登录
[root@bigdata01 ~]# ssh bigdata02
[root@bigdata01 ~]# ssh bigdata03

##如果想实现节点互相免密登录，重复以上操作
```

#### 2.安装hadoop

##### （1）修改配置文件

- 修改hadoop-env.sh文件

```shell
[root@bigdata01 hadoop]# vi hadoop-env.sh 
export JAVA_HOME=/data/soft/jdk1.8
export HADOOP_LOG_DIR=/data/hadoop_repo/logs/hadoop
```

- 修改core-site.xml文件

```shell
[root@bigdata01 hadoop]# vi core-site.xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://bigdata01:9000</value>
    </property>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/data/hadoop_repo</value>
   </property>
</configuration>
```

- 修改hdfs-site.xml文件

```shell
[root@bigdata01 hadoop]# vi hdfs-site.xml 
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>2</value>
    </property>
    <property>
        <name>dfs.namenode.secondary.http-address</name>
        <value>bigdata01:50090</value>
    </property>
    #<!--设置其他用户执行操作时会提醒没有权限问题-->
    <property>
        <name>dfs.permissions</name>
        <value>false</value>
    </property>
</configuration>
##把hdfs中文件副本的数量设置为2，最多为2，因为现在集群中有两个从节点，还有secondaryNamenode进程所在的节点信息
```

- 修改mapred-site.xml，设置mapreduce使用的资源调度框架

```shell
[root@bigdata01 hadoop]# vi mapred-site.xml
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
</configuration>
```

- 修改yarn-site.xml，设置yarn上支持运行的服务和环境变量白名单

```shell
[root@bigdata01 hadoop]# vi yarn-site.xml
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.nodemanager.env-whitelist</name>
        <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_MAPRED_HOME</value>
    </property>
	<property>
		<name>yarn.resourcemanager.hostname</name>
		<value>bigdata01</value>
	</property>
	##开启日志聚合功能，设置完配置，每个节点需要输入mapred --daemon start historyserver命令才可以查看history
	<property> 
        <name>yarn.log-aggregation-enable</name>  
        <value>true</value>
    </property>
    <property>
        <name>yarn.log.server.url</name>
        <value>http://bigdata01:19888/jobhistory/logs/</value>
    </property>

</configuration>
#注意，针对分布式集群在这个配置文件中还需要设置resourcemanager的hostname，否则nodemanager找不到resourcemanager节点。
```

- 修改workers文件，增加所有从节点的主机名，一个一行

```shell
[root@bigdata01 hadoop]# vi workers
bigdata02
bigdata03
```

- 修改启动脚本

```shell
[root@bigdata01 hadoop]# cd /data/soft/hadoop-3.2.0/sbin
[root@bigdata01 sbin]# vi start-dfs.sh
HDFS_DATANODE_USER=root
HDFS_DATANODE_SECURE_USER=hdfs
HDFS_NAMENODE_USER=root
HDFS_SECONDARYNAMENODE_USER=root

[root@bigdata01 sbin]# vi stop-dfs.sh
HDFS_DATANODE_USER=root
HDFS_DATANODE_SECURE_USER=hdfs
HDFS_NAMENODE_USER=root
HDFS_SECONDARYNAMENODE_USER=root

[root@bigdata01 sbin]# vi start-yarn.sh
YARN_RESOURCEMANAGER_USER=root
HADOOP_SECURE_DN_USER=yarn
YARN_NODEMANAGER_USER=root

[root@bigdata01 sbin]# vi stop-yarn.sh
YARN_RESOURCEMANAGER_USER=root
HADOOP_SECURE_DN_USER=yarn
YARN_NODEMANAGER_USER=root
```

##### （2）hadoop配置发送到其他机器

```shell
[root@bigdata01 sbin]# cd /data/soft/
[root@bigdata01 soft]# scp -rq hadoop-3.2.0 bigdata02:/data/soft/
[root@bigdata01 soft]# scp -rq hadoop-3.2.0 bigdata03:/data/soft/
```

##### （3）主节点格式化HDFS

```shell
[root@bigdata01 soft]# cd /data/soft/hadoop-3.2.0
[root@bigdata01 hadoop-3.2.0]# bin/hdfs namenode -format
```

##### （4）主节点启动集群

```shell
start-all.sh
```

##### （5）验证集群

```shell
[root@bigdata01 hadoop-3.2.0]# jps
6128 NameNode
6621 ResourceManager
6382 SecondaryNameNode

[root@bigdata02 ~]# jps
2385 NodeManager
2276 DataNode

[root@bigdata03 ~]# jps
2326 NodeManager
2217 DataNode
```