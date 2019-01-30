# 通过DLA连线存取ECS上的MySQL数据库

## 背景信息
[Data Lake Analytics](https://www.aliyun.com/product/datalakeanalytics)（简称DLA） 作为云上数据处理的枢纽，支持通过标准JDBC对关系型数据库（MySQL、MS SQL Server、PostgreSQL）中的数据进行查询和分析，并可以将分析结果回写。

本文档适用于用户自建于ECS上的关系型数据库。本文档将以用户自建于ECS上的MySQL为例，介绍如何使用DLA，通过JDBC连线存取位于同一地域数据中心(例如新加坡)的MySQL数据并进行读写。本文档之方法，也适用于读写MS SQL Server、PostgreSQL数据库。

针对阿里云RDS实例之关系型数据库，请参考[通过DLA读写RDS数据](https://help.aliyun.com/document_detail/98566.html)。

## 注意事项
为了让DLA能访问ECS上的数据库，需要在ECS的安全组规则添加白名单：允许**100.104.0.0/16** IP地址段：![Security Group](http://blog-demo.oss-ap-southeast-1.aliyuncs.com/dla_ecs_mysql/dla_ecs_securitygroup.png "添加安全组存取JDBC白名单")

## 准备工作
通过DLA读写MySQL数据前，您需要远程连接ECS服务器，通过以下操作准备好MySQL数据库以及测试数据：

1. 创建MySQL数据库(例如，dla_db)：
	```
    MySQL [(none)]> CREATE DATABASE dla_db;
    Query OK, 1 row affected (0.00 sec)

    MySQL [(none)]> show databases;
    +--------------------+
    | Database           |
    +--------------------+
    | information_schema |
    | dla_db             |
    | mysql              |
    | performance_schema |
    +--------------------+
	4 rows in set (0.00 sec)
    ```
2. 新增用户(例如，dla_user)，DLA之后将以该用户存取数据库。并将适当的存取权限以及DLA之IP地址段**100.104.0.0/16**加入数据库存取白名单列表：

    ```
    MySQL [(none)]> USE mysql;
    Database changed
    
    MySQL [mysql]> CREATE USER 'dla_user'@'100.104.0.0/255.255.0.0' IDENTIFIED BY 'dla_userpasswd';
    Query OK, 0 rows affected (0.00 sec)
    
    MySQL [mysql]> GRANT SELECT,INSERT,UPDATE,DELETE,CREATE,DROP,ALTER,INDEX on dla_db.* TO 'dla_user'@'100.104.0.0/255.255.0.0';
    Query OK, 0 rows affected (0.00 sec)
    
    MySQL [mysql]> flush privileges;
    Query OK, 0 rows affected (0.00 sec)
    
    MySQL [mysql]> SELECT user,host FROM user;
    +----------+-------------------------+
    | user     | host                    |
    +----------+-------------------------+
    | dla_user | 100.104.0.0/255.255.0.0 |
    | root     | 127.0.0.1               |
    | root     | localhost               |
    +----------+-------------------------+
    3 rows in set (0.00 sec)
    ```
    
3. 创建表并写入测试数据通过以下示例在RDS中创建person表：
    ```
    CREATE TABLE person (
      id int,
      name varchar(1023),
      age int
    );
    ```
    
    向person表中插入数据：
    ```
    INSERT INTO person VALUES 
    (1, 'james', 10), 
    (2, 'bond', 20), 
    (3, 'jack', 30), 
    (4, 'lucy', 40);
    ```

## 通过DLA读取ECS服務器上之MySQL数据
于**DLA**控制台，登陆DMS进入**DMS for Data Lake Analytics**页面，执行以下步骤。

### 步骤一：创建MySQL数据库连接
创建一个底层映射到MySQL的数据库schema连接。
```
CREATE SCHEMA hello_mysql_vpc_rds WITH DBPROPERTIES ( 
    CATALOG = 'mysql', 
    LOCATION = 'jdbc:mysql://192.168.2.141:3306/dla_db', 
    USER = 'dla_user', 
    PASSWORD = 'dla_userpasswd', 
    INSTANCE_ID = 'ecs_instance_id', 
    VPC_ID = 'vpc_id' 
);
```
* LOCATION：输入 jdbc:mysql://<ECS实例私有IP>:3306/<数据库名>
* VPC_ID：输入<ECS实例的专有网络ID>，在ECS**实例详情**页面查看专有网路ID。
* INTANCE_ID：输入<ECS实例ID>，如下图所示i-########8e。
![INSTANCE_ID](http://blog-demo.oss-ap-southeast-1.aliyuncs.com/dla_ecs_mysql/dla_ecs_vpcid.png "VPC ID")

### 步骤二：创建MySQL外表
创建一个外表，名为tbl_person，映射到MySQL数据库person表。
```
CREATE EXTERNAL TABLE hello_mysql_vpc_rds.tbl_person (
	id int,
	name varchar(1023),
	age int
) tblproperties (
	table_mapping = "person"
);
```

### 步骤三：查询MySQL数据
查询外表（tbl_person表）数据。也可以通过MySQL客户端连接DLA中的数据库schema（hello_mysql_vpc_rds），读写MySQL数据。
![SQL QUERY](http://blog-demo.oss-ap-southeast-1.aliyuncs.com/dla_ecs_mysql/dla_sql_query.png "MySQL query")
