### Maria DB Cluster  

| Role | IP |
| ------- | -------------- |
| MariaDB 1 | 100.67.191.121 |
| MariaDB 2 | 100.67.191.122 |
| MariaDB 3 | 100.67.191.123 |
| MariaDB LB1 | 100.67.191.129 |
| MariaDB LB2 | 100.67.191.128 |
| MariaVIP 1| 100.67.191.120 |

Keepalived virtual router id : 21  


1. 安裝MariaDB  
[On MariaDB]  
```
#vim /etc/yum.repos.d/mariadb.repo
[mariadb-main]
name = MariaDB Server
baseurl = https://downloads.mariadb.com/MariaDB/mariadb-10.3.9/yum/rhel/$releasever/$basearch
gpgkey = https://downloads.mariadb.com/MariaDB/RPM-GPG-KEY-MariaDB
gpgcheck = 1
enabled = 1

[mariadb-tools]
name = MariaDB Tools
baseurl = https://downloads.mariadb.com/Tools/rhel/$releasever/$basearch
gpgkey = https://downloads.mariadb.com/Tools/MariaDB-Enterprise-GPG-KEY
gpgcheck = 1
enabled = 1

#yum -y install MariaDB-server galera MariaDB-client
#systemctl start mariadb
#systemctl enable mariadb
#systemctl status mariadb
#mysql --version
```
安裝完成後 執行secure MariaDB安裝 且允許root remote登入  
```bash=
sudo mysql_secure_installation
```

#### 將個別DB載入 Harbor的schema  
嘗試過在MariaDB cluster架設起來之後在載入schema，但是一直會失敗  
測試到現在發現可行的方法就是在建立galera cluster之前就個別載入schema  

[On MariaDB]  
```bash=
wget https://raw.githubusercontent.com/goharbor/harbor/release-1.5.0/make/photon/db/registry.sql
mysql -u root -p < registry.sql
```

**Starting the Galera Cluster**  

#### 測試MariaDB功能
在任一點上執行  
```bash=
mysql -u root -p
> CREATE DATABASE tr_test;
>show databases;

```
