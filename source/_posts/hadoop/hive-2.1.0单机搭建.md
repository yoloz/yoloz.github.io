---
title: hive
comments: false
toc: false
date: 2019-09-18 16:56:50
categories: hadoop
tags:
---

Requirements:jdk, hadoop

`tar -zxvf apache-hive-2.1.0-bin.tar.gz -C ~`

## 修改配置文件

* hive-env.sh

`cp $HIVE_HOME/conf/hive-env.sh.template $HIVE_HOME/conf/hive-env.sh`
主要是配置hadoop的路径, 找到"HADOOP_HOME=="，填写hadoop路径

* hive-site.xml

`cp $HIVE_HOME/conf/hive-default.xml.template $HIVE_HOME/conf/hive-site.xml`
主要是配置metadata存储及hive数据存储在hdfs的位置, 可以直接使用默认的derby，如下使用mysql:

``` xml
<configuration>
<property>
<name>javax.jdo.option.ConnectionURL</name>
<value>jdbc:mysql://localhost:3306/hive_basic?createDatabaseIfNotExist=true&amp;characterEncoding=UTF-8&amp;useSSL=false</value>
<description>hive_basic为要创建的数据库名,注意字符集设置</description>
</property>
<property>
<name>javax.jdo.option.ConnectionDriverName</name>
<value>com.mysql.jdbc.Driver</value>
</property>
<property>
<name>javax.jdo.option.ConnectionUserName</name>
<value>root</value>
<description>登录账户名</description>
</property>
<property>
<name>javax.jdo.option.ConnectionPassword</name>
<value>123456</value>
<description>登录密码</description>
</property>
<property>
<name>hive.metastore.warehouse.dir</name>
<value>/user/hive/warehouse</value>
<description>hive表在hdfs的位置</description>
</property>
</configuration>  
```

> 如果使用mysql, 需要将对应的jdbc驱动jar移到lib下

## 初始化

``` sh
#hadoop已启动
export HADOOP_HOME=<hadoop-install-dir>
$HIVE_HOME/bin/schematool -dbType <db type> -initSchema  # use "derby" as db type or "mysql"
```

## 启动

``` sh
$HIVE_HOME/bin/hiveserver2 &  #"&" 表示后台运行
$HIVE_HOME/bin/beeline
# beeline> !connect jdbc:hive2://localhost:10000/default
# Connecting to jdbc:hive2://localhost:10000/default
# Enter username for jdbc:hive2://localhost:10000/default:(直接回车)
# Enter password for jdbc:hive2://localhost:10000/default:(直接回车)
# Error: Failed to open new session: java.lang.RuntimeException: org.apache.hadoop.ipc.RemoteException(org.apache.hadoop.security.authorize.AuthorizationException): User: ylz is not allowed to impersonate anonymous (state=,code=0)
# beeline>
```

> default localhost:10000
> 默认没有用户密码, hive-site.xml中hive.server2.authentication为NONE

对于上述的Error, 修改hadoop的core-site.xml加入:

``` xml
<property>
<name>hadoop.proxyuser.ylz.hosts</name>
<value>*</value>
</property>
<property>
<name>hadoop.proxyuser.ylz.groups</name>
<value>*</value>
</property>
```

重启hadoop

``` sh
$HADOOP_HOME/sbin/hadoop-daemon.sh stop namenode
$HADOOP_HOME/sbin/hadoop-daemon.sh stop secondarynamenode
$HADOOP_HOME/sbin/hadoop-daemon.sh stop datanode
$HADOOP_HOME/sbin/hadoop-daemon.sh start namenode
$HADOOP_HOME/sbin/hadoop-daemon.sh start secondarynamenode
$HADOOP_HOME/sbin/hadoop-daemon.sh start datanode
```

## JDBC

使用原生方式如下:

``` java
@Test
public void hiveTest() {
    String url = "jdbc:hive2://127.0.0.1:10000/default";
    try {
        Properties properties = new Properties();
        Class.forName("org.apache.hive.jdbc.HiveDriver");
        try (Connection conn = DriverManager.getConnection(url, properties);
            Statement stmt = conn.createStatement();
            ResultSet rs = stmt.executeQuery("show databases")) {
            while (rs.next()) {
                System.out.println(rs.getString(1));
            }
        }
    } catch (Exception e) {
        e.printStackTrace();
    }
}
@Test
public void hiveKerberosTest() {
    String url = "jdbc:hive2://127.0.0.1:10000/default;principal=hive/cdh1.com@CDHKDC.COM";
    try {
        Properties properties = new Properties();
        org.apache.hadoop.conf.Configuration conf = new org.apache.hadoop.conf.Configuration();
        conf.set("hadoop.security.authentication", "Kerberos");
        UserGroupInformation.setConfiguration(conf);
        UserGroupInformation.loginUserFromKeytab("wufang", "/tmp/wufang.keytab");
        Class.forName("org.apache.hive.jdbc.HiveDriver");
        try (Connection conn = DriverManager.getConnection(url, properties);
                Statement stmt = conn.createStatement();
                ResultSet rs = stmt.executeQuery("show databases")) {
            while (rs.next()) {
                System.out.println(rs.getString(1));
            }
        }
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

> jdbc:hive2://127.0.0.1:10000/default  

jdbc:hive2://127.0.0.1:10000/default; principal=hive/cdh1.com@CDHKDC. COM  
jdbc:hive2://hdp1:2181, hdp2:2181, hdp3:2181/; serviceDiscoveryMode=zooKeeper; zooKeeperNamespace=hiveserver2  

* dependencies

commons-collections-3.2.2.jar  
commons-configuration-1.6.jar  
commons-lang-2.6.jar  
hadoop-auth-2.6.0-cdh5.8.3.jar  
hadoop-common-2.7.3.jar  
hive-exec-1.1.0-cdh5.8.3.jar  
hive-jdbc-1.1.0-cdh5.8.3.jar  
hive-metastore-1.1.0-cdh5.8.3.jar  
hive-serde-1.1.0-cdh5.8.3.jar  
hive-service-1.1.0-cdh5.8.3.jar  
httpclient-4.4.jar  
httpcore-4.4.jar  
libthrift-0.9.3.jar  

cdh版也可以使用如下方式

``` java
@Test
public void cdhHiveTest() {
    String url = "jdbc:hive2://127.0.0.1:10000/default;AuthMech=1;" +
            "KrbRealm=CDHKDC.COM;KrbHostFQDN=cdh1.com;" +
            "KrbServiceName=hive;KrbAuthType=2";
    try {
        Properties properties = new Properties();
        Class.forName("com.cloudera.hive.jdbc41.HS2Driver");
        try (Connection conn = DriverManager.getConnection(url, properties);
                Statement stmt = conn.createStatement();
                ResultSet rs = stmt.executeQuery("show databases")) {
            while (rs.next()) {
                System.out.println(rs.getString(1));
            }
        }
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

>[Installing Cloudera JDBC and ODBC Drivers on Clients in CDH](https://www.cloudera.com/documentation/enterprise/latest/topics/hive_jdbc_odbc_driver_install.html#hive_installing_jdbc_odbc_drivers)  
[Cloudera JDBC latest Driver Documentation for Apache Hive](https://www.cloudera.com/documentation/other/connectors/hive-jdbc/latest.html)
