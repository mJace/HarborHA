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
[On MariaDB 1,2,3]  

```bash=
sudo apt-key adv --recv-keys --keyserver hkp://keyserver.ubuntu.com:80 0xF1656F24C74CD1D8
sudo add-apt-repository 'deb [arch=amd64,i386,ppc64el] http://ftp.utexas.edu/mariadb/repo/10.1/ubuntu xenial main'
sudo apt update -y
sudo apt-get install mariadb-server rsync
```
安裝完成後 執行secure MariaDB安裝 且允許root remote登入  
```bash=
sudo mysql_secure_installation
```

#### 將個別DB載入 Harbor的schema  
嘗試過在MariaDB cluster架設起來之後在載入schema，但是一直會失敗  
測試到現在發現可行的方法就是在建立galera cluster之前就個別載入schema  

[On MariaDB 1,2,3]  
```bash=
wget https://raw.githubusercontent.com/goharbor/harbor/release-1.5.0/make/photon/db/registry.sql
mysql -u root -p < registry.sql
```


#### 設定MariaDB  
[On MariaDB 1]  
Edit [/etc/mysql/conf.d/galera.cnf](https://github.com/mJace/HarborHA/blob/master/mariaDB/MariaDB_1/galera.cnf)  
```
[mysqld]
binlog_format=ROW
default-storage-engine=innodb
innodb_autoinc_lock_mode=2
bind-address=0.0.0.0
# Galera Provider Configuration
wsrep_on=ON
wsrep_provider=/usr/lib/galera/libgalera_smm.so
# Galera Cluster Configuration
wsrep_cluster_name="galera_cluster"
wsrep_cluster_address="gcomm://100.67.191.121,100.67.191.122,100.67.191.123"

# Galera Synchronization Configuration
wsrep_sst_method=rsync

# Galera Node Configuration
wsrep_node_address="100.67.191.121"
wsrep_node_name="Node1"
```


[On MariaDB 2]  
Edit [/etc/mysql/conf.d/galera.cnf](https://github.com/mJace/HarborHA/blob/master/mariaDB/MariaDB_2/galera.cnf)  
```
[mysqld]
binlog_format=ROW
default-storage-engine=innodb
innodb_autoinc_lock_mode=2
bind-address=0.0.0.0
# Galera Provider Configuration
wsrep_on=ON
wsrep_provider=/usr/lib/galera/libgalera_smm.so
# Galera Cluster Configuration
wsrep_cluster_name="galera_cluster"
wsrep_cluster_address="gcomm://100.67.191.121,100.67.191.122,100.67.191.123"

# Galera Synchronization Configuration
wsrep_sst_method=rsync

# Galera Node Configuration
wsrep_node_address="100.67.191.122"
wsrep_node_name="Node2"
```


[On MariaDB 3]
Edit [/etc/mysql/conf.d/galera.cnf](https://github.com/mJace/HarborHA/blob/master/mariaDB/MariaDB_3/galera.cnf)  
```
[mysqld]
binlog_format=ROW
default-storage-engine=innodb
innodb_autoinc_lock_mode=2
bind-address=0.0.0.0
# Galera Provider Configuration
wsrep_on=ON
wsrep_provider=/usr/lib/galera/libgalera_smm.so
# Galera Cluster Configuration
wsrep_cluster_name="galera_cluster"
wsrep_cluster_address="gcomm://100.67.191.121,100.67.191.122,100.67.191.123"

# Galera Synchronization Configuration
wsrep_sst_method=rsync

# Galera Node Configuration
wsrep_node_address="100.67.191.123"
wsrep_node_name="Node3"
```

**Starting the Galera Cluster**  

先關閉所有Node上的mysql server  
[On MariaDB 1,2,3]  

```bash=
sudo systemctl stop mysql
```

當所有Node的mysql server都關閉後，在Node 1上啟動cluster  
[On MariaDB 1]  
```bash=
sudo galera_new_cluster
```

到此Node 1會建立一個cluster，可用以下指令查看cluster資訊  
[On MariaDB 1]  
```bash=
mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_cluster_size'"
```

接著再到Node 2,3啟動mysql server  
[On MariaDB 2,3]  
```bash=
sudo systemctl start mysql
```

啟動後用同樣指令可看到目前cluster的size為3  
[On MariaDB 2,3]  
```bash=
mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_cluster_size'"
```

#### 測試MariaDB功能
在任一點上執行  
```bash=
mysql -u root -p
> CREATE DATABASE tr_test;
```

在另一點上執行  
```bash=
mysql -u root -p
show databases;
```

#### 架設Keepalived for MariaDB cluster  

下載keepalived
[On Maria LB 1,2]  
```bash=
sudo apt-get install keepalived
```

[On Maria LB 1,2]  
Edit [/etc/keepalived/keepalived.conf](https://github.com/mJace/HarborHA/blob/master/mariaDB/keepalived_cnf/keepalived.conf)  
Maria LB 1,2兩台用的keepalived設定是相同的

```
global_defs {
  router_id haborlb
}
vrrp_sync_groups VG1 {
  group {
    VI_1
  }
}
#Please change "ens160" to the interface name on you loadbalancer hosts.
#In some case it will be eth0, ens16xxx etc.
vrrp_instance VI_1 {
  interface enp0s8

  track_interface {
    enp0s8
  }

  state MASTER
  virtual_router_id 21
  priority 10

  virtual_ipaddress {
    100.67.191.120/16
  }
  advert_int 1
  authentication {
    auth_type PASS
    auth_pass d0cker
  }

}

virtual_server 100.67.191.120 3306 {
  delay_loop 15
  lb_algo rr
  lb_kind DR
  protocol TCP
  nat_mask 255.255.0.0
  persistence_timeout 10

  real_server 100.67.191.121 3306 {
    weight 10
    TCP_CHECK {
        connect_timeout 3
        connect_port 3306
    }
  }

  real_server 100.67.191.122 3306 {
    weight 10
    TCP_CHECK {
        connect_timeout 3
        connect_port 3306
    }
  }

  real_server 100.67.191.123 3306 {
    weight 10
    TCP_CHECK {
        connect_timeout 3
        connect_port 3306
    }
  }
}
```

重啟keepalived使其套用最新設定  
```bash=
sudo systemctl restart keepalived
```
