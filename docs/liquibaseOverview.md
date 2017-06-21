
# Understanding Liquibase

<!-- TOC depthFrom:2 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [Why Liquibase?](#why-liquibase)
- [Installing MySQL](#installing-mysql)
- [Create a new sample database](#create-a-new-sample-database)
- [Getting Started with Liquibase](#getting-started-with-liquibase)
- [Installing Liquibase](#installing-liquibase)
- [Creating a changesetlog](#creating-a-changesetlog)
- [Adding new changes to the changelog](#adding-new-changes-to-the-changelog)

<!-- /TOC -->

## Why Liquibase?

With projects increasing in size, complexity and delivery times becoming continually shorter, its always helpful to find the right kind of automation to help speed the work of consistently managing application environments. [Liquibase](http://www.liquibase.org/)  is a helpful tool that manages the migration of database updates in a declarative fashion.


---

## Installing MySQL

Let's get started... First we'll need a database we can work on... we'll use [MySQL](http://www.mysql.com) and start with a refresher by getting it installed, up and running, and make a simple schema we can work with. Let's start by installing MySQL

```bash
$ apt-get update && apt-get upgrade -y
$ sudo apt-get install mysql-server
```

This will install the database server and the required dependencies. During the installation you will need to enter (and confirm) a password for the MySQL root user. Now MySQL should be installed!


Now that the server's installed and running, let's enable the service and restart the server.

```bash

ubuntu@server:~$ sudo systemctl restart mysql
ubuntu@server:~$ sudo systemctl enable mysql
Synchronizing state of mysql.service with SysV init with /lib/systemd/systemd-sysv-install...
Executing /lib/systemd/systemd-sysv-install enable mysql

```


Login to the database using the root password you just set. Let's check the version...

```sql
mysql> select version();
+-------------------------+
| version()               |
+-------------------------+
| 5.7.16-0ubuntu0.16.04.1 |
+-------------------------+
1 row in set (0.00 sec)
```

... and see what databases were created by default in our new server.

```sql
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.00 sec)

mysql>
```


## Create a new sample database
Since we're going do some tests on our server, we'll need a few things... We'll start by creating a database, a new user, and we'll give the user privileges to the database.

```sql
mysql>
mysql> create database sampledb;
Query OK, 1 row affected (0.00 sec)

mysql> create user ubuntu@localhost identified by "Password$123";
Query OK, 0 rows affected (0.00 sec)

mysql> grant all privileges on sampledb.* to ubuntu@localhost;
Query OK, 0 rows affected (0.00 sec)

mysql> exit
Bye

```


We'll log back in as our new user, and validate that we've correctly permissioned by connecting to the database.

```sql
ubuntu@server:~$ mysql -uubuntu -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 17
Server version: 5.7.16-0ubuntu0.16.04.1 (Ubuntu)

Copyright (c) 2000, 2016, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| sampledb           |
+--------------------+
2 rows in set (0.00 sec)


mysql>
mysql> use sampledb;
Database changed
```


Great! - Looks like everything's working! Now we have installed the MySQL database on ubuntu v16.04, created a new user, and created a database. Now we have a working environment that we can begin to push changes to using Liquibase.

---

## Getting Started with Liquibase

What is Liquibase? [Liquibase](http://www.http://www.liquibase.org) is an open source database-independent library for tracking, managing and applying database schema changes. It's great because it make managing databases declaritive, and it adheres to the guidelines around "Infrastracture as Code". The changeset file you use to manage a database is essentially a text file that can be stored in git along with all the other code artifacts. It's probably easier to walk through a simple example so lets do that next...


## Installing Liquibase
Liquibase is written in java... so, first lets make sure the java jre is installed

```bash
ubuntu@server:~/lbase$
ubuntu@server:~/lbase$ java -version
openjdk version "9-internal"
OpenJDK Runtime Environment (build 9-internal+0-2016-04-14-195246.buildd.src)
OpenJDK 64-Bit Server VM (build 9-internal+0-2016-04-14-195246.buildd.src, mixed mode)

```


Liquibase is really easy to install. Download and unzip it, and make sure its in your path.

```bash
$ cd ~
$ pwd
/home/ubuntu
$
$ mkdir bin-liquibase
$ cd bin-liquibase
$ wget https://github.com/liquibase/liquibase/releases/download/liquibase-parent-3.5.3/liquibase-3.5.3-bin.tar.gz
$
$ tar -zxf liquibase-3.5.3-bin.tar.gz
$
$ ls
lib          liquibase                   liquibase.bat  liquibase.spec  sdk
LICENSE.txt  liquibase-3.5.3-bin.tar.gz  liquibase.jar  README.txt
$
$ export PATH=$PATH:/home/ubuntu/bin-liquibase
```


Now that we have Liquibase installed and in our path, let's create a workspace, download the MySQL JDBC driver.

```bash
$ cd ~
$ mkdir lbase
$ cd lbase
```


Also, lets create the config file so that Liquibase knows how to connect to the MySQL database we just installed. For our convenience with MySQL, we'll add the flag "createDatabaseIfNotExist=true" to the JDBC url so when we run Liquibase, it will create the database if it's not already available.

```bash

driver: com.mysql.jdbc.Driver
classpath: /home/ubuntu/lbase/mysql-connector-java-5.1.40-bin.jar
url: jdbc:mysql://server.mysql/sampledb?useSSL=false&createDatabaseIfNotExist=true
username: ubuntu
password: Password$123

```

---


We'll also create a simple table as a starting point. As with most databases, we'll assume there's been some pre-existing work that we have to consume and update. Let's create a file called 'food.sql' and enter the following.

```sql
CREATE TABLE IF NOT EXISTS food (
  food_id       INT(11) NOT NULL AUTO_INCREMENT,
  food_desc    VARCHAR(20) DEFAULT NULL
)
```

Then we'll create the table by executing the sql script from the commandline

```sql
mysql>
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| sampledb           |
+--------------------+
2 rows in set (0.01 sec)

mysql> use sampledb;
Database changed
mysql>
mysql> source /home/ubuntu/lbase/food.sql;
Query OK, 0 rows affected (0.01 sec)

mysql>
```

---

## Creating a changesetlog
With Liquibase installed and a running mysql database with a simple table, now we'll create our first changeset. We could create a changeset from scratch, but instead we'll generate our changeset log based on the table structure(s) currently in the database. With that, we'll can modify it to add additional updates to the database if desired.

```bash
ubuntu@server:~/lbase$ ls
coffee.lb    liquibase                   liquibase.properties                 sdk
food.sql     liquibase-3.5.3-bin.tar.gz  liquibase.spec
lib          liquibase.bat               mysql-connector-java-5.1.40-bin.jar
LICENSE.txt  liquibase.jar               README.txt
ubuntu@server:~/lbase$
ubuntu@server:~/lbase$
ubuntu@server:~/lbase$ ./liquibase --changeLogFile="./sampledb.xml"  generateChangeLog
Liquibase 'generateChangeLog' Successful
ubuntu@server:~/lbase$
```

When we review the changeset that was generated we can see the table and fields we created by executing the sql file.


```xml
<?xml version="1.1" encoding="UTF-8" standalone="no"?>
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog" xmlns:ext="http://www.liquibase.org/xml/ns/dbchangelog-ext" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog-ext http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-ext.xsd http://www.liquibase.org/xml/ns/dbchangelog http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.5.xsd">
    <changeSet author="ubuntu (generated)" id="1482608299217-1">
        <createTable tableName="food">
            <column name="food_id" type="INT"/>
            <column name="food_desc" type="VARCHAR(20)"/>
        </createTable>
    </changeSet>
</databaseChangeLog>
```


And for good measure... we'll create a git repository to keep track of the changes we've made over time. This will be helpful, particularly during continuous integration.

```bash
$ git init
Initialized empty Git repository in /home/ubuntu/lbase/.git/
$ git add sampledb.xml
$ git commit -m "liquibase: initial commit"
[master (root-commit) 949ca67] liquibase: initial commit
 1 file changed, 9 insertions(+)
 create mode 100644 sampledb.xml
```


---

## Adding new changes to the changelog

Next, let's add a new field to our new table. We'll make another changelog which captures the details of how to roll forward and make the change, as well as how to rollback the update to the target table.

```xml
   <changeSet id="update-food-table" author="irvingr">

        <addColumn
            schemaName="sampledb"
            tableName="food">
           <column name="comment" type="varchar(50)"/>
        </addColumn>

        <rollback>
          <dropColumn
            columnName="comment"
            schemaName="sampledb"
            tableName="food"/>
        </rollback>

    </changeSet>
```


We'll run the update to push the changes to the table in the database.

```bash
$ liquibase --changeLogFile=./food-update.xml update
Liquibase Update Successful
```

We can just as easily rollback whatever changes we made. Though we made a single change, we can use a different rollbackCount value to rollback the last change, the last 2 changes, going all the way back to the earliest change listed in the database.

```bash
$ liquibase --changeLogFile=./food-update.xml rollbackCount 1
Liquibase Rollback Successful
```

Each changeset has a rollback associated with it, so we can rollback the table we created too...

```bash
$ liquibase --changeLogFile=./food-table.xml rollbackCount 1
Liquibase Rollback Successful
```


When you look in the database change log, you can see the details of what updates are being made.

```bash
mysql> select id, description, author, filename, orderexecuted from DATABASECHANGELOG;
+-------------------+----------------------------+---------+------------------+---------------+
| id                | description                | author  | filename         | orderexecuted |
+-------------------+----------------------------+---------+------------------+---------------+
| add-food-table    | createTable tableName=food | irvingr | ./food-table.xml |             1 |
| update-food-table | addColumn tableName=food   | irvingr | ./food-table.xml |             2 |
+-------------------+----------------------------+---------+------------------+---------------+
2 rows in set (0.00 sec)
```

## Summary:
Now we've done a few things... we've
* established a running MySQL service
* installed Liquibase and created a config file to connect to the database
* added a table to the newly created database
* extracted the changes into our initial changelog
* added some new changes with a different changelog
* rolled back all of the changes we created

---

###### [irvingr2017|tutorial]
