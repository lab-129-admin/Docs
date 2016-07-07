#Hadoop and stack 搭建



这篇文档主要是详细介绍hadoop和stack的运行环境的搭建步骤，希望给之后搭建的同学一些帮助。本文将会搭建完全分布式虚拟运行环境，使用7台虚拟机，其中一个节点作为master节点，其余六台作为worker节点。

----------


##hadoop环境搭建


这个部分首先介绍hadoop的搭建过程，分为如下几个步骤

> **hadoop搭建步骤**

> - 新建用户组并分配权限
> - 搭建java的运行环境
> - 配置网络参数
> - 配置hadoop运行参数
> - 运行hadoop环境
###新建用户组并分配权限
搭建hadoop环境最好是新建一个全新的用户，笔者实际搭建时将所有用户都命名为hadoop
在每个主机上都创建hadoop用户：
```shell
# adduser hadoop
```
当然在虚拟机多的情况下所有统一命令可以采用shell脚本做批处理
为了让hadoop用户更好地使用权限，我们最好是为其分配超级权限，其实就是让他获得和root一样的能力，但是每次还是要sudo
```shell
# vi /etc/sudoers
```
找到 root    ALL=(ALL:ALL) ALL这一行，然后在这一行之下再加一行 hadoop   ALL=(ALL:ALL) ALL.
保存退出，这样便可以了，然后就切换到hadoop用户，开始完成java环境的搭建。



###搭建java运行环境
很多人一提到java运行环境都觉得非常麻烦，其实java运行环境的搭建是相对简单的：
首先在如下网址下载jdk1.8：http://www.oracle.com/technetwork/java/javase/downloads/index-jsp-138363.html
然后将下载后的文件解压到一个文件下，个人建议如果用户主空间大的话可以直接放在主空间下，然后重命名为jdk1.8. 重命名代码如下：
```shell
$ mv jdkxxxx jdk1.8
```
接着需要修改本地配置文件
```shell
$ vi ~/.bashrc
```
然后在文件后面添加如下语句：
```shell
export JAVA_HOME=/home/hadoop/jdk1.8
export CLASSPATH=${JAVA_HOME}/lib
export PATH=${JAVA_HOME}/bin:$PATH
```
配置好以后保存退出，然后输入如下命令激活配置：
```shell
$ source ~/.bashrc
```
最后输入
```shell
$ java -version
```
验证是否配置成功，如果成功会输出java版本信息


###配置网络参数
这里配置网络参数主要是主机名，网络host，以及无密码登录的设置
####主机名
首先设置主机名，对于主机名来说，其实就是方便于标识服务器，不用再写那一串繁琐的ip地址。所以对于7台主机，分别命名为master和slaver1-6. 修改方式如下：
```shell
$ sudo vi /etc/hostname
```
然后输入主机名便完成修改，但是但是要注意，修改后要重启虚拟机才会生效，可以用hostname命令验证是否成功：
```shell
$ hostname
```

####配置hosts
配置host的目的是让当前主机可以将其他主机的主机名解析为ip地址，在ssh时就只需要使用主机名就行。
修改方式如下：
```shell
# sudo vi /etc/hosts
```
一个hosts的例子如下：
```shell
172.16.1.11 master
10.123.8.64 slaver1
10.123.8.65 slaver2
10.123.8.66 slaver3
10.123.8.67 slaver4
10.123.8.68 slaver5
10.123.8.69 slaver6
```
添加这些在localhost那一行后面即可，每台主机都需要配置大致相同的hosts（但是要注意slaver节点的master对应的地址必须使用浮动ip，即是10.123.8.62），可以使用scp命令将master结点的hosts文件复制到其余slaver节点上

####免密码登录
免密码登录的目的是因为在hadoop或者spark运行时会远程到slaver传递任务，如果不实现免密码登录会导致每次都要输入密码，相当麻烦。

首先在所有机器的/home/hadoop/目录下建立 .ssh文件夹
```shell
$ mkdir /home/hadoop/.ssh
```
然后在master机器上生成密钥对
```shell
$ ssh-keygen -t rsa （注意：ssh与-keygen之间没有空格）
``` 
一路回车即可。
转到.ssh目录 cd ~/.ssh 可以看到生成了id_rsa,和id_rsa.pub两个文件
然后执行
```shell
$ cp id_rsa.pub authorized_keys
此时master节点自己已经可以实现无密码登录
``` 
之后把Master上面的authorized_keys文件复制到Slave机器的/home/hadoop/.ssh/文件下面
```shell
$ scp authorized_keys slave:~/.ssh
```
修改.ssh目录的权限以及authorized_keys 的权限(这个必须修改，要不然还是需要密码)
```shell
$ sudo chmod 644 ~/.ssh/authorized_keys
$ sudo chmod 700 ~/.ssh
```
正常情况下,到这个地方就可以SSH无密码登录了
输入ssh slaver1-6 进行测试。

###运行hadoop环境
终于到了最后一步了，其实如果之前的步骤完成的好的话，hadoop的搭建基本上来说不会有太大的问题。
首先需要下载一个hadoop的发行版本，选择的原则就是最新的稳定版，本文采用的是hadoop-2.7.2版本。
下载后解压，接着我们要做的就是配置hadoop以及设置hadoop的运行环境。
具体步骤如下：
进入hadoop目录下的etc/hadoop/文件夹
我们第一个需要修改的文件就是 etc/hadoop/core-site.xml
```xml
<configuration>
<property>
        <name>hadoop.tmp.dir</name>
        <value>/home/hadoop/hadoop-2.6.0/tmp</value>
        <description>Abase for other temporary directories.</description>
    </property>
    <property>
        <name>fs.default.name</name>
        <value>hdfs://master:9000</value>
    </property>
</configuration>
```
> ***NOTE：***
> - 需要注意的就是tmp的设置，tmp是用来保存一些状态文件的，所以需要自行mkdir一个，并且将tmp路径填写进来

第二个需要修改的是hadoop/etc/hadoop/hdfs-site.xml
```xml
<property>
    <name>dfs.name.dir</name>
    <value>/home/hadoop/hadoop-2.7.2/dfs/name</value>
    <description>Path on the local filesystem where the NameNode stores the namespace and transactions logs persistently.</description>
</property>
 
<property>
    <name>dfs.data.dir</name>
    <value>/home/hadoop/hadoop-2.7.2/dfs/data</value>
    <description>Comma separated list of paths on the local filesystem of a DataNode where it should store its blocks.</description>
</property>
<property>
    <name>dfs.replication</name>
    <value>2</value>
</property>
```
> ***NOTE：***
> - 这个文件十分重要，没一项都值得细说，第一项是namenode的文件目录，需要用户自行创建（mkdir）一个，并把该文件夹的路径填入，在实际运行中只有master节点会使用到，保存当前hadoop集群的信息
> - 第二项是datanode的文件目录，需要用户自行创建（mkdir），并把文件夹路径填入。实际运行中只有datanode节点会使用到，保存datanode的信息。
> - 第三项replication主要是指定存储数据的冗余度，正常的集群一般是3，就是表示一份数据会有三份便于应对数据丢失的问题。

然后需要修改hadoop/etc/hadoop/mapred-site.xml
```xml
<configuration>
<property>
 <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
    <property>
        <name>mapreduce.jobhistory.address</name>
        <value>master:10020</value>
    </property>
    <property>
        <name>mapreduce.jobhistory.webapp.address</name>
        <value>master:19888</value>
    </property>
</configuration>
```
这个文件没什么可说的，照抄

然后需要修改hadoop/etc/hadoop/hadoop-env.sh
```shell
export JAVA_HOME=/home/hadoop/jdk1.8
```
修改java_home路径，修改完成后,让修改生效
```shell
$ source hadoop-env.sh
```

最后需要修改的是hadoop/etc/hadoop/slaves
这个文件是保存所有slaver节点的主机名或者IP地址的，用于告诉hadoop集群有哪些slaver
```shell
slaver1
slaver2
slaver3
slaver4
slaver5
slaver6
```
添加如上信息就可以完成

最后就是配置hadoop的环境变量，主要是配置/etc/environment的信息
```
PATH="/usr/local/sbin:/home/hadoop/jdk1.8/bin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/home/hadoop/hadoop-2.7.2/bin:/home/hadoop/hadoop-2.7.2/sbin"
``` 
值得注意的是，其实这里就是要把hadoop的执行二进制文件和java路径加入，设置完成后source一下使之生效。然后看看默认提示里面hadoop能不能提示，如果能就说明hadoop完成搭建了

>***NOTE***
> - 这里需要最后注意，在每个集群节点都采用相同的设置，都需要拷贝hadoop的整个目录，可能各个节点的配置的唯一差异就是主机名和hosts了吧

###运行hadoop
下面是运行hadoop环境，这里也是验证之前所有配置是否成功的关键了。首先我们需要格式化namenode节点，这个操作相当于是对整个集群名字空间进行格式化，格式化后集群ID会改变，集群保存的信息也会被更新。但是这也是容易出问题的地方，在之后的Q&A部分，我会总结经验。
```
$ hdfs namenode -format
```
这步在集群不需要重新配置时，只用在启动时执行一次就行
接着
```
$ start-dfs.sh
```
这一步会启动namenode和一些相关的组建
可以通过查看jps命令可以判断是否成功，在master命令行输入jps，这是java查看守护进程的命令。（有可能提示你jps没有，那么就要添加一个java路径，上网查下）
如果看到了NameNode，说明成功
然后
```
$ start-yarn.sh
```
这是启动计算框架的命令，这个启动后在每个slave节点运行jps查看是否有datanode进程，如果有，说明启动成功。
到此时hadoop就算是全部启动完成了，这个时候就可以使用网站查看整个集群的信息了
地址：
> master:50070
登录上述地址就可以验证集群是否成功，注意观察其中的live node个数，如果和期望的个数相同说明启动成功。

## Spark 环境搭建
spark基本算是hadoop平台上的一个工具，它借用hadoop的运行平台利用内存交换实现速度更快的计算操作，如果针对大规模数据，spark是一个理想的平台。其搭建远远比hadoop更简单。
spark的搭建分成如下几个步骤

> **spark搭建步骤**

> - 搭建scala环境
> - 配置spark参数
> - 运行spark
###搭建scala环境
spark是基于scala语言开发的，scala语言可以看作是java语言的一个精简版，它仍然需要在jvm上面运行。
scala安装简单，首先在官网上下载scala的安装包，然后将安装包解压缩，放在一个路径上，然后只需要在.bashrc上配置相关的变量就行。基本与jdk安装配置一样
```
$ vi ~/.bashrc
```
```
export SCALA_HOME=/home/hadoop/scala
export PATH=${JAVA_HOME}/bin:$PATH:$SCALA_HOME/bin
```
增加scala路径变量，并修改path路径，修改完成后source一下，然后在命令行输入scala，看是否能进入scala shell

###配置spark参数
scala配置完成后就该配置spark了。首先需要下载spark，在下载时千万注意，官网的package type那里一定要选 那个最长的就是带有with user-provided Hadoop字样的。不然会各种出问题。(这里注意ubuntu下下载这个可能比较慢，可以在windows下先下载好再考过来）
下载完成后解压，然后放在一个路径下，接着打开文件，进入conf文件夹
然后把spark-env.sh生成一份，并把slaves生成一份
```shell
$ cp spark-env.sh.template spark-env.sh
$ cp slaves.template slaves
```
然后编辑这两个文件
首先是slaves.sh
```shell
slaver1
slaver2
slaver3
slaver4
slaver5
slaver6
```
增加几个slave节点的主机名
然后配置spark-env.sh
```
###jdk安装目录
export JAVA_HOME=/home/hadoop/jdk1.8
###scala安装目录
export SCALA_HOME=/home/hadoop/scala
###spark集群的master节点的ip
export SPARK_MASTER_IP=172.16.1.11
export SPARK_LOCAL_IP=127.0.0.1
###指定的worker节点能够最大分配给Excutors的内存大小
export SPARK_WORKER_MEMORY=2g
###hadoop集群的配置文件目录
export HADOOP_CONF_DIR=/home/hadoop/hadoop-2.7.2/etc/hadoop
###spark类节点
export SPARK_DIST_CLASSPATH=$(hadoop classpath)
```
配置完成后source一下，就可以了。
> ***NOTE：***
> - 由于需要在每个slaver节点开启spark，所以必须将以上对于scala和spark的设置在其余slaver节点再重复一遍。

###运行spark
如果上述配置都完成了，就可以开始启动spark了
进入spark/sbin目录，然后按顺序执行spark-master.sh以及执行spark-slaves.sh，从执行中就可以看出哪些节点启动成功。最后在各个节点输入jps验证进程，matser节点会出现matser进程，slave节点会出现worker进程.并且通过
>master:8080也可以看到spark的管理界面，同样观察live节点的个数

到目前为止，spark和hadoop平台就搭建完成了，下面的章节将会对一些可能出现的问题进行归纳。
##Q＆A
>**Note**
>其实如果你遇到问题第一件事是用jps查看进程，然后查看相关日志，里面有具体问题
在这一部分笔者将会总结一些常见的问题
1.启动hadoop以后没有看见namenode进程，遇到这种问题多半是因为主进程没有执行成功，可能原因如下：
> - 端口绑定到虚拟机浮动IP上面了，但是由于浮动ip无法被虚拟机感知，所以失败，改为绑定真实ip即可
> - 可能是端口被占用，使用如下命令查看端口占用情况
$ netstat -a | grep XXX(你想查看的端口号）
> - 如果日志里面写了说你没有format，那么运行hadoop namenode -format生成环境文件

2.启动hadoop后有节点datanode没有启动
> - 大概率是因为namenode的ID与datanode ID没有统一，解决方法是删除所有节点的tmp文件夹，log文件夹，data文件夹和name文件夹然后重新格式化namenode，重启hadoop。我自己是写了自动脚本完成格式化
> - 可能是hosts配置有问题，注意在slaver节点上的matser节点映射地址不能使用虚拟机的内部ip，必须使用浮动ip
> - 相关环境变量或者文件没有写全或者没有source，重复一下之前的动作



























