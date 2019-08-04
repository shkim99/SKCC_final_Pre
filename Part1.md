# Part I  

## 1. Create a CDH Cluster on AWS
```
[public IP]
13.209.206.100    pmn.skcc.com    pmn
13.209.220.134    pcm.skcc.com    pcm
13.209.242.76     pw1.skcc.com    pw1
13.209.56.213     pw2.skcc.com    pw2
13.209.86.24      pw3.skcc.com    pw3

[private IP]
13.209.206.100    mn.skcc.com    mn
13.209.220.134    cm.skcc.com    cm
13.209.242.76     w1.skcc.com    w1
13.209.56.213     w2.skcc.com    w2
13.209.86.24      w3.skcc.com    w3
```

### a. Linux setup
####  Add the following linux accounts to all nodes
```
# 계정 생성
sudo useradd training
sudo usermod -u 3800 training
sudo passwd training

sudo groupadd skcc
cat /etc/passwd | grep training
sudo usermod -aG skcc training

# training 계정 sudo 권한추가
sudo usermod -aG wheel training
```
![](/img/1-1.PNG)
####  List the your instances by IP address and DNS name
```
/etc/hosts 에 private IP 등록
[centos@cm ~]$ getent hosts
```
![](/img/1-1.PNG)

####  List the Linux release you are using
```
[centos@cm ~]$ cat /etc/os-release
# bit 확인
[centos@cm ~]$ getconf LONG_BIT
```
![](/img/1-1.PNG)
####  List the file system capacity for the first nodes
```
[centos@cm ~]$ df -Th
```
![](/img/1-12.PNG)

####  List the command and output for yum repolist enabled
```
yum repolist all
```
![](/img/1-13.PNG)

#### List the /etc/passwd entries for training and the /etc/group entries for skcc
```
cat /etc/passwd | grep training
cat /etc/group | grep skcc
```
![](/img/1-21.PNG)

#### List output of the following commands
```
getent group skcc
getent passwd training
```
![](/img/1-22.PNG)

### Linux setup (추가)

#### a.2.1 yum update
```
$ sudo yum update
$ sudo yum install -y wget
```
#### a.2.2. firewall check and disable
```
[centos@cm ~]$ systemctl status firewalld
Unit firewalld.service could not be found.
```
![Image 000_1](img/0_1.PNG)
* 해당 시스템은 firewalld 없음 필요시<br>
$ systemctl stop firewalld<br>
$ systemctl disable firewalld<br>

#### a.2.3. Selinux 정지
```
[centos@cm ~]$ sestatus
SELinux status:                 disabled
[centos@cm ~]$ sudo vi /etc/selinux/config
```
![Image 000](img/0_2.PNG)

#### a.2.4. NTP 설정
```
[centos@cm ~]$ sudo yum install -y ntp
[centos@cm ~]$ sudo vi /etc/ntp.conf
[centos@cm ~]$ systemctl start ntpd
[centos@cm ~]$ systemctl enable ntpd
[centos@cm ~]$ ntpq -p
```
![Image 000](./img/0_3.PNG)

#### a.2.5. VM Swappiness 설정
```
cat /proc/sys/vm/swappiness
sudo sysctl -w vm.swappiness=1
```
VM Swappiness permanent
```
sudo vi /etc/sysctl.conf
  =>  vm.swappiness=1
```
#### a.2.6. SSH Connection 설정
```
sudo vi /etc/ssh/sshd_config
# PasswordAuthentication -> yes 로 변경 후 저장

# sshd 재시작 및 상태확인
sudo systemctl restart sshd.service
sudo systemctl status sshd.service
```
#### a.2.7. KEYGEN 설정
```
cd .ssh

ssh-keygen -t rsa

ssh-copy-id -i ~/.ssh/id_rsa.pub mn
ssh-copy-id -i ~/.ssh/id_rsa.pub w1
ssh-copy-id -i ~/.ssh/id_rsa.pub w2
ssh-copy-id -i ~/.ssh/id_rsa.pub w3

ssh mn1 명령실행후 로그인 없이 로그인 여부 확인
```
#### a.2.8. Disable Transparent Hugepage Support
```
sudo vi /etc/rc.d/rc.local
  =>  
echo "never" > /sys/kernel/mm/transparent_hugepage/enabled
echo "never" > /sys/kernel/mm/transparent_hugepage/defrag
sudo chmod +x /etc/rc.d/rc.local
sudo vi /etc/default/grub
   add -> transparent_hugepage=never (on line GRUB_CMDLINE_LINUX )
grub2-mkconfig -o /boot/grub2/grub.cfg
```

#### a.2.9. 필요시   IP V6 disable
```
sudo sysctl -w net.ipv6.conf.all.disable_ipv6=1
sudo sysctl -w net.ipv6.conf.default.disable_ipv6=1
```

## 2. CDH Installation
### 2.1  CM 설치 및 JDK 설치

#### 1)  Import Cloudera manager repository on cm :  계정   trainning
```
sudo wget https://archive.cloudera.com/cm5/redhat/7/x86_64/cm/cloudera-manager.repo -P /etc/yum.repos.d/
sudo sed -i 's/x86_64\/cm\/5/x86_64\/cm\/5.15.2/g' /etc/yum.repos.d/cloudera-manager.repo
sudo rpm --import https://archive.cloudera.com/cm5/redhat/7/x86_64/cm/RPM-GPG-KEY-cloudera
sudo yum install cloudera-manager-daemons cloudera-manager-server
```
![Image 001_b](img/1_b_1.PNG)

#### 2)  Install a supported Oracle JDK
```
# 설치 가능한 jdk list 확인
sudo yum list oracle*
sudo yum install -y oracle-j2sdk1.7
```
* 서버별 java 버전 확인 후 필요 시 설치
```
java -version
```
#### 3)  JDBC Connector 설치  
```
sudo wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.46.tar.gz ~/
tar xvfz ~/mysql-connector-java-5.1.46.tar.gz
sudo mkdir -p /usr/share/java
sudo cp ~/mysql-connector-java-5.1.46/mysql-connector-java-5.1.46-bin.jar /usr/share/java/mysql-connector-java.jar
sudo yum install -y mysql-connector-java
```
![Image 001_b](img/1_b_2.PNG)

#### 4)  Copy to all other nodes (JDBC Connector)
```
scp /usr/share/java/mysql-connector-java.jar mn:~/
scp /usr/share/java/mysql-connector-java.jar w1:~/
scp /usr/share/java/mysql-connector-java.jar w2:~/
scp /usr/share/java/mysql-connector-java.jar w3:~/

ssh mn "sudo mkdir -p /usr/share/java; sudo mv ~/mysql-connector-java.jar /usr/share/java/"
ssh w1 "sudo mkdir -p /usr/share/java; sudo mv ~/mysql-connector-java.jar /usr/share/java/"
ssh w2 "sudo mkdir -p /usr/share/java; sudo mv ~/mysql-connector-java.jar /usr/share/java/"
ssh w3 "sudo mkdir -p /usr/share/java; sudo mv ~/mysql-connector-java.jar /usr/share/java/"
```
![Image 001_b](img/1_b_2.PNG)

### 2.2  MariaDB 설치  (1.b Install a MySQL server)
#### DB 설치
```
sudo yum install -y mariadb-server

sudo systemctl enable mariadb
sudo systemctl start mariadb
# mariadb 상태 확인
sudo systemctl status mariadb
```
#### shows the hostname of your database server
```
mysql --version
```
![Image 001_b](img/1_b_2.PNG)
#### database server version
```
mysql -u root -p
show databases;
```
![Image 001_b](img/1_b_2.PNG)

#### 권한 설정 : 전체 Y 선택
````
sudo /usr/bin/mysql_secure_installation
````
#### Creating Databases for Cloudera Software
```
mysql -u root -p

# db, user 생성
CREATE DATABASE scm DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON scm.* TO 'scm-user'@'%' IDENTIFIED BY 'password';

CREATE DATABASE amon DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON amon.* TO 'amon-user'@'%' IDENTIFIED BY 'password';

CREATE DATABASE rman DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON rman.* TO 'rman-user'@'%' IDENTIFIED BY 'password';

CREATE DATABASE hue DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON hue.* TO 'hue-user'@'%' IDENTIFIED BY 'password';

CREATE DATABASE metastore DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON metastore.* TO 'metastore-user'@'%' IDENTIFIED BY 'password';

CREATE DATABASE sentry DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON sentry.* TO 'sentry-user'@'%' IDENTIFIED BY 'password';

CREATE DATABASE oozie DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON oozie.* TO 'oozie-user'@'%' IDENTIFIED BY 'password';

FLUSH PRIVILEGES;
```
### 2.3  Cloudera Manager Install
#### CM SET
```
#  set CM DB
sudo /usr/share/cmf/schema/scm_prepare_database.sh mysql scm scm-user password
# CM server start
sudo systemctl start cloudera-scm-server
sudo tail -f /var/log/cloudera-scm-server/cloudera-scm-server.log

http://cm:7180
admin / admin
```
#### CHD 클러스터 설치 시작

## 3. Data Handling
## 3.1 문제 1번 항목 마지막 답변 및 문제 2번 항목 답변
#### In you cluster, create a user named "training"
```
5대 training 계정 스크린 샷
id
```
#### In MySQL, create a database and name it “test”
````
mysql -u root -p
create database test;
````

#### Create 2 tables in the test databases: authors and posts
```
# file copy local 에서 cm

scp skcc.pem authors.sql.zip training@15.164.144.197:.
scp skcc.pem posts.sql.zip training@15.164.144.197:.
```
![Image 001_b](img/1_b_2.PNG)
```
# upzip 설치 및 압축 해제
sudo yum install -y unzip

unzip authors.sql.zip
unzip posts.sql.zip
```

```
mysql -u root -p
use test;

# sql
source authors-23-04-2019-02-34-beta.sql
source posts23-04-2019 02-44.sql

select count(*) from authors;
select count(*) from posts;
```
#### Create and grant user “training” with password “training” full access to the test database
```
create user 'training'@'%' identified by 'training';
grant all privileges on *.* to 'training'@'%';
```

## 3.2 Extract tables authors and posts from the database and create Hive tables.
```
# training 계정으로 접속
su training

# sqoop 수행 [cm]
sqoop import \
--connect jdbc:mysql://util.com/test \
--username training \
--password training \
--table posts \
--fields-terminated-by "\t" \
--target-dir /user/training/posts

sqoop import \
--connect jdbc:mysql://util.com/test \
--username training \
--password training \
--table authors \
--fields-terminated-by "\t" \
--target-dir /user/training/authors
```
수행 결과 및 directory 조회 이미지
#### create authors as an external table
```
create external table authors
(
id int,
first_name string,
last_name string,
email string,
birthdate date,
added timestamp
)
row format delimited
fields terminated by '\t'
location '/user/training/authors/.'
```
#### create posts as a managed table
```
create table posts
(
id int,
author_id int,
title string,
description string,
content string,
date date
)
row format delimited
fields terminated by '\t'
location '/user/training/posts/.'
```
# 4. Create and run a Hive/Impala query
####  generate the results dataset
```
select a.id Id, a.first_name fname, a.last_name lname, count(*) num_posts
from authors a, posts p
where a.id = p.author_id
group by a.id, a.first_name, a.last_name;
```
#### The output of the query should be saved in your HDFS home directory.
```
insert overwrite directory '/user/training/results'
row format delimited
fields terminated by '\t'
select a.id Id, a.first_name fname, a.last_name lname, count(*) num_posts
from authors a, posts p
where a.id = p.author_id
group by a.id, a.first_name, a.last_name;
```

# 5. Export the data from above query to MySQL
#### Create a MySQL "results" table under the database "test"
```
create table results
( Id int, fname varchar(500), lname varchar(500), num_posts int);
```
#### export into MySQL the results of your query
```
sqoop export \
--connect jdbc:mysql://util.com/test \
--username training \
--password training \
--table results \
--fields-terminated-by '\t' \
--export-dir hdfs://cm/user/training/results
```
