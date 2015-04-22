---
layout: post
title: Setup Hadoop cluster with Cloudera Repository
---

Note
====

## Facebook

Using Hive 

2008: 200GB per day 
2009: 20+TB per day
2+TB (compressed) raw data per day

2012: Hadoop cluster with 100PB (world lagest cluster)

## Twitter

Using Pig

2010: 7TB per day, 2+TB per year (~ 10,000 CD-ROM per day)




Install HDFS Cluster
====================

##### Install CDH 5 Repository

```
wget http://archive.cloudera.com/cdh5/one-click-install/redhat/6/x86_64/cloudera-cdh-5-0.x86_64.rpm
sudo yum --nogpgcheck localinstall cloudera-cdh-5-0.x86_64.rpm
```

##### Install ZooKeeper

```
sudo yum install zookeeper
sudo yum install zookeeper-server
sudo service zookeeper-server init -- After new installation only
sudo service zookeeper-server start
```

##### Install Resource Manager (MapReduce node)

```
sudo yum clean all; sudo yum install hadoop-yarn-resourcemanager
```

##### Install Name Node 

```
sudo yum clean all; sudo yum install hadoop-hdfs-namenode
```

##### Install Secondary NameNode host (if used)

```
sudo yum clean all; sudo yum install hadoop-hdfs-secondarynamenode
```

##### Install All cluster hosts except the Resource Manager

```
sudo yum clean all; sudo yum install hadoop-yarn-nodemanager hadoop-hdfs-datanode hadoop-mapreduce
```

##### One host in the cluster

```
sudo yum clean all; sudo yum install hadoop-mapreduce-historyserver hadoop-yarn-proxyserver
```

##### Install Hadoop client on machine that connect to hadoop

```
sudo yum clean all; sudo yum install hadoop-client
```

Config HDFS
===========

### Change hadoop default conf directory (using `alternatives`)

1. Copy the default configuration to your custom directory:

```
 sudo cp -r /etc/hadoop/conf.empty /etc/hadoop/conf.my_cluster
 ```

2. CDH uses the alternatives setting to determine which Hadoop configuration to use. 
Set alternatives to point to your custom directory, as follows.
To manually set the configuration on Red Hat-compatible systems:

```
sudo alternatives --install /etc/hadoop/conf hadoop-conf /etc/hadoop/conf.my_cluster 50 
sudo alternatives --set hadoop-conf /etc/hadoop/conf.my_cluster
```

### Config cluster

Update your conf directory following below steps and upload to your host

#### Config Namenode address and permission (on every host)

1. core-site.xml

`fs.defaultFS`: Namenode address, port

```
<property>
 <name>fs.defaultFS</name>
 <value>hdfs://namenode-host.company.com:8020</value>
</property>
```

2. hdfs-site.xml

`dfs.permissions.superusergroup` Specifies the UNIX group containing users that will be treated as superusers by HDFS. You can stick with the value of 'hadoop' or pick your own group depending on the security policies at your site

```
<property>
 <name>dfs.permissions.superusergroup</name>
 <value>hadoop</value>
</property>
```

#### Config Namenode data directory

1. `hdfs-site.xml` on the NameNode

`dfs.name.dir` or `dfs.namenode.name.dir` This property specifies the URIs of the directories where the NameNode stores its metadata and edit logs. Cloudera recommends that you specify at least two directories. One of these should be located on an NFS mount point, unless you will be using a High Availability (HA) configuration.
*Ensure that configured directory are existing and accessible (drwx------) on corresponding host*

```
<property>
 <name>dfs.namenode.name.dir</name>
 <value>file:///data/1/dfs/nn,file:///nfsmount/dfs/nn</value>
</property>
```

2. `hdfs-site.xml` on each DataNode

`dfs.data.dir` or `dfs.datanode.data.dir` This property specifies the URIs of the directories where the DataNode stores blocks. Cloudera recommends that you configure the disks on the DataNode in a JBOD configuration, mounted at /data/1/ through /data/N, and configure dfs.data.dir or dfs.datanode.data.dir to specify file:///data/1/dfs/dn through file:///data/N/dfs/dn/.
*Ensure that configured directory are existing and accessible (drwx------) on corresponding host*

```
<property>
 <name>dfs.datanode.data.dir</name>
 <value>file:///data/1/dfs/dn,file:///data/2/dfs/dn,file:///data/3/dfs/dn,file:///data/4/dfs/dn</value>
</property>
```

#### Formatting Namenode host

```
sudo -u hdfs hdfs namenode -format
```

#### Enabling WEBHDFS on Namenode

1. Update or add following to `hdfs-site.xml`

```
<property>
  <name>dfs.webhdfs.enabled</name>
  <value>true</value>
</property>
```

### Start DFS

##### Push your custom directory (for example /etc/hadoop/conf.my_cluster) to each node in your cluster; for example:

```
scp -r /etc/hadoop/conf.my_cluster myuser@myCDHnode-<n>.mycompany.com:/etc/hadoop/conf.my_cluster
```

##### Manually set alternatives on each node to point to that directory, as follows.

```
sudo alternatives --verbose --install /etc/hadoop/conf hadoop-conf /etc/hadoop/conf.my_cluster 50 
sudo alternatives --set hadoop-conf /etc/hadoop/conf.my_cluster
```

##### Start

```
for x in `cd /etc/init.d ; ls hadoop-hdfs-*` ; do sudo service $x start ; done
```

*Make sure your hdfs `/tmp` directory exists and has correct permission*

```
sudo -u hdfs hadoop fs -mkdir /tmp
sudo -u hdfs hadoop fs -chmod -R 1777 /tmp
```

Config & deploy YARN
====================

##### Start ResourceManager service (on ResourceManager host)

```
sudo service hadoop-yarn-resourcemanager start
```

##### Start NodeManager (on every DataNode)

```
sudo service hadoop-mapreduce-historyserver start
```

##### Create hdfs home directory for each user (LINUX user)

```
sudo -u hdfs hadoop fs -mkdir  /user/{user}
sudo -u hdfs hadoop fs -chown {user} /user/{user}
```


Config hadoop daemmon start at boot
===================================

[link] http://www.cloudera.com/content/cloudera-content/cloudera-docs/CDH5/latest/CDH5-Installation-Guide/cdh5ig_init_configure.html?scroll=topic_27_2

```
sudo chkconfig zookeeper-server on
sudo chkconfig hadoop-hdfs-namenode on
sudo chkconfig hadoop-yarn-resourcemanager on
sudo chkconfig hadoop-hdfs-secondarynamenode on
sudo chkconfig hadoop-yarn-nodemanager on
sudo chkconfig hadoop-hdfs-datanode on
sudo chkconfig hadoop-mapreduce-historyserver on
```

Install Supporting Components
=============================


## Apache Hive 

### Installation

*Installing Hive*

```
sudo yum install hive hive-metastore hive-server2 hive-hbase
```

### Configuration

Hive supports Local, Embeded and Remote mode. Below is the configuration for Remote Mode

#### Configure hive to connect to mysql 

`/usr/lib/hive/conf/hive-site.xml`:

```
<property>
  <name>javax.jdo.option.ConnectionURL</name>
  <value>jdbc:mysql://myhost/metastore</value>        <!-- jdbc:postgresql://myhost/metastore -->
  <description>the URL of the MySQL database</description>
</property>

<property>
  <name>javax.jdo.option.ConnectionDriverName</name>
  <value>com.mysql.jdbc.Driver</value>                <!-- org.postgresql.Driver -->
</property>

<property>
  <name>javax.jdo.option.ConnectionUserName</name>
  <value>{mysql user}</value>
</property>

<property>
  <name>javax.jdo.option.ConnectionPassword</name>
  <value>{mysql pass}</value>
</property>

<property>
  <name>datanucleus.autoCreateSchema</name>
  <value>false</value>
</property>

<property>
  <name>datanucleus.fixedDatastore</name>
  <value>true</value>
</property>

<property>
  <name>datanucleus.autoStartMechanism</name> 
  <value>SchemaTable</value>
</property> 

<property>
  <name>hive.metastore.uris</name>
  <value>thrift://{ip or hostname of metastore host}:9083</value>
  <description>IP address (or fully-qualified domain name) and port of the metastore host</description>
</property>
```

*Configuration Mysql (OPTIONAL)*

```
sudo yum install mysql-server
sudo yum install mysql-connector-java
sudo ln -s /usr/share/java/mysql-connector-java.jar /usr/lib/hive/lib/mysql-connector-java.jar
sudo service mysqld start
sudo /usr/bin/mysql_secure_installation
sudo chkconfig mysqld on
```

Init Hive Metastore Schema

```
mysql -u root -p
> create database metastore;
> grant all on metastore.* to {hiveuser} identified by 'hive password';
> USE metastore;
> SOURCE /usr/lib/hive/scripts/metastore/upgrade/mysql/hive-schema-0.12.0.mysql.sql; 
```

#### Create hive HDFS directory

```
sudo -u hdfs hadoop fs -mkdir -p /user/hive/warehouse
sudo -u hdfs hadoop fs -chmod 1777 /user/hive/warehouse
```

### Start service

```
sudo service hive-metastore start
sudo chkconfig hive-metastore on
sudo service hive-server2 start
sudo chkconfig hive-server2 on
```

### Connect Hive using Beeline

```
beeline
```

```
!connect jdbc:hive2://localhost:10000 qunguyen qunguyen org.apache.hive.jdbc.HiveDriver
```

## Oozie

```
sudo yum install oozie
```

### Configuration

##### Use one of following configuration to tell Oozie what kind of Hadoop MapReduce that is will running

1. To use YARN (without SSL):

```
sudo alternatives --set oozie-tomcat-conf /etc/oozie/tomcat-conf.http
```

2. To use YARN (with SSL):

```
sudo alternatives --set oozie-tomcat-conf /etc/oozie/tomcat-conf.https
```

3. To use MRv1(without SSL) :

```
sudo alternatives --set oozie-tomcat-conf /etc/oozie/tomcat-conf.http.mr1
```

4. To use MRv1(with SSL) :

```
sudo alternatives --set oozie-tomcat-conf /etc/oozie/tomcat-conf.https.mr1
```


##### Config Oozie with MySQL

Copy MySQL Connector library to Oozie library directory

```
sudo ln -s /usr/share/java/mysql-connector-java.jar /var/lib/oozie/mysql-connector-java.jar
```

Config Oozied to use MySQL in `oozie-site.xml`

```
<property>
    <name>oozie.service.JPAService.jdbc.driver</name>
    <value>com.mysql.jdbc.Driver</value>
</property>
<property>
    <name>oozie.service.JPAService.jdbc.url</name>
    <value>jdbc:mysql://localhost:3306/oozie</value>
</property>
<property>
    <name>oozie.service.JPAService.jdbc.username</name>
    <value>oozie</value>
</property>
<property>
    <name>oozie.service.JPAService.jdbc.password</name>
    <value>oozie</value>
</property>
```

##### Create Oozie database 

```
sudo -u oozie /usr/lib/oozie/bin/ooziedb.sh create -run
```

##### Installing the Oozie Shared Library in Hadoop HDFS

```
sudo -u hdfs hadoop fs -mkdir -p /user/oozie
sudo -u hdfs hadoop fs -chown oozie:oozie /user/oozie
sudo oozie-setup sharelib create -fs hdfs://lab:8020 -locallib /usr/lib/oozie/oozie-sharelib-yarn.tar.gz
```

##### Configuring Support for Oozie Uber JARs

An uber JAR is a JAR that contains other JARs with dependencies in a lib/ folder inside the JAR. You can configure the cluster to handle uber JARs properly for the MapReduce action (as long as it does not include any streaming or pipes) by setting the following property in the oozie-site.xml file:

```
<property>
  <name>oozie.action.mapreduce.uber.jar.enable</name>
  <value>true</value>
</property>
```

When this property is set, users can use the oozie.mapreduce.uber.jar configuration property in their MapReduce workflows to notify Oozie that the specified JAR file is an uber JAR.

##### Configuring Oozie to Run against a Federated Cluster

To run Oozie against a federated HDFS cluster using ViewFS, configure the oozie.service.HadoopAccessorService.supported.filesystems property in oozie-site.xml as follows:

```
<property>
  <name>oozie.service.HadoopAccessorService.supported.filesystems</name>
  <value>hdfs,viewfs</value>
</property>
```

##### Start Oozie Server

```
sudo service oozie start
sudo chkconfig oozie on
```

Add following variable to environment variable to not to specify it when using Oozie command line

```
export OOZIE_URL=http://localhost:11000/oozie
```

## Hue

HDFS and Oozie must be installed before installing Hue

### Installation

```
sudo yum install hue
```

### Configuration

#### WebHDFS or HttpFS Configuration

##### For WebHDFS only:

`hdfs-site.xml`

```
<property>
  <name>dfs.webhdfs.enabled</name>
  <value>true</value>
</property>
```

`core-site.xml`

```
<!-- Hue WebHDFS proxy user setting -->
<property>
  <name>hadoop.proxyuser.hue.hosts</name>
  <value>*</value>
</property>
<property>
  <name>hadoop.proxyuser.hue.groups</name>
  <value>*</value>
</property>

<!-- to make diferrent temp directory for hue user in  -->
<!-- XXX: MAY CAUSE ERROR NEED FURTHER INVESTIGATION -->
<property>
  <name>hadoop.tmp.dir</name>
  <value>/tmp/hadoop-${user.name}-${hue.suffix}</value>
</property>
```


`hue.ini`

```
[hadoop]
[[hdfs_clusters]]
[[[default]]]
webhdfs_url=http://FQDN:50070/webhdfs/v1/
```


### Start service

```
 sudo service hue start
```









