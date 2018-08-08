# Harbor HA 安裝手冊

## Harbor HA 架構
![](https://i.imgur.com/ZcNEr5A.png)



| Role     | Node 1 IP     | Node 2 IP     | Node 3 IP    |
| -------- | ------------- | ------------- | ------------ | 
| Harbor   | 100.67.191.10 | 100.67.191.11 | N\A          |
| HarborLB | 100.67.191.119| 100.67.191.118| N\A          |
| HarborVIP| 100.67.191.201|               |              |
| 
| MariaDB  | 100.67.191.121| 100.67.191.122|100.67.191.123|
| MariaLB  | 100.67.191.129| 100.67.191.128| N\A          |
| MariaVIP | 100.67.191.120|               |              |
|
| Redis    | 100.67.191.111| 100.67.191.112| N\A          |
| RedisVIP | 100.67.191.110|               |              |
|
|NFS Server| 100.67.191.8  |               |              |

Harbor 此架構主要由三大項目組成   
#### 1. Harbor Node和其LoadBalancer   
(Active/Active)  
藉由keepalived將流量平均分配到其下Harbor Node  

#### 2. MariaDB Cluster  
(Active/Active)  
由三個MariaDB 透過Galera 建立cluster，在藉由Keepalived將流量平均分配  

#### 3. Redis Cluster
(Active/StandBy)  
透過Redis本身的Master/Slave設定加上Keepalived的腳本設計，達到Redis的高可用架構  

## 軟體版本/需求  


| Software | Version  | Description |
| -------- | -------- | ----------- |
| Harbor   | v1.5.0   | https://storage.googleapis.com/harbor-releases/release-1.5.0/harbor-offline-installer-v1.5.0.tgz |
| Redis    | >= 4.0.6   |             |
| MariaDB  | >= 10.2.14 |             |
| Python   | > v2.7   |             |
| Docker   | > 1.10   |             |
| Docker compose| > 1.6.0 |         |

## Network Ports / Settings  

| Port   | Protocol  | Description |
| -----  | --------  | ----------- |
| 80     | http  | Harbor UI and API will accept requests on this port for http protocol |
| 443    | https | Harbor UI and API will accept requests on this port for https protocol |
| 4443   | https | Connections to the Docker Content Trust service for Harbor, only needed when Notary is enabled|
| 6379   |    | Default redis port |
| 3306   |    | Default MariaDB port |
| 5432   |    | Default Postgress port |

**Keepalived setting**
|role|vid|
|----|---|
|Harbor| 75 |
|Redis| 11 |
|MariaDB| 21 |


---


# 環境建置  
### Redis HA Cluster  

| Role | IP |
| ------- | -------------- |
| Redis 1 | 100.67.191.111 |
| Redis 2 | 100.67.191.112 |
| Redis VIP | 100.67.191.110 |
Keepalived virtual router id : 11

**Redis 1為Master**  
**Redis 2為Slave**  

1. 於Redis 1和Redis 2下載 Redis 和 keepalived  
```bash=
wget http://download.redis.io/releases/redis-4.0.6.tar.gz
tar -zxvf redis-4.0.6.tar.gz
sudo sudo apt-get install keepalived
```

2. Redis 1 and 2 編譯Redis  
```bash=
cd redis-4.0.6
sudo make install
```

3. 設定Redis cluster  

[On Master]  
修改redis.conf  
```
daemonize  yes
bind  0.0.0.0
pidfile /var/run/redis_6379.pid
logfile "/var/log/redis.log"
```

[On Slave]  
修改redis.conf，跟Master不一樣的是Slave多了一行```slaveof <masterIP> <port>```  
```
daemonize  yes
bind  0.0.0.0
pidfile /var/run/redis_6379.pid
logfile "/var/log/redis.log"
slaveof 100.67.191.111 6379
```

啟動Redis  
[On Master and Slave]  
```bash=
cd redis-4.0.6
sudo redis-server ./redis.conf
```

檢查Redis狀態  
[On Master and Slave]  
```bash=
redis-cli info replication
```


4. 設定Keepalived  
#### [On Master]  
edit /etc/keepalived/keepalived.conf  
```
global_defs {
	router_id redis
}
 
vrrp_script chk_redis {
	script "/etc/keepalived/scripts/check_redis.sh"
	interval 4
	weight -5
	fall 3  
	rise 2
}
 
vrrp_instance VI_REDIS {
	state MASTER
	interface eth1
	virtual_router_id 11
	priority 100
	advert_int 1
	nopreempt
 
	authentication {
		auth_type PASS
		auth_pass 1111
	}
 
	virtual_ipaddress {
		192.168.99.10
	}
 
	track_script {
		chk_redis
	}
 
	notify_master /etc/keepalived/scripts/redis_master.sh
	notify_backup /etc/keepalived/scripts/redis_backup.sh
	notify_fault  /etc/keepalived/scripts/redis_fault.sh
	notify_stop   /etc/keepalived/scripts/redsi_stop.sh
}
```
**Edit script for redis HA**  
edit /etc/keepalived/scripts/check_redis.sh for checking master reids  
```
#!/bin/bash  
CHECK=`/usr/local/bin/redis-cli PING`  
if [ "$CHECK" == "PONG" ] ;then  
      echo $CHECK  
      exit 0  
else   
      echo $CHECK  
      service keepalived stop 
      exit 1  
fi  
```

edit /etc/keepalived/scripts/redis_backup.sh for master back online and being slave of new master  
```
#!/bin/bash
REDISCLI="/usr/local/bin/redis-cli"
LOGFILE="/var/log/keepalived-redis-state.log"
echo "[backup]" >> $LOGFILE
date >> $LOGFILE
echo "Being slave...." >> $LOGFILE 2>&1
sleep 15
echo "Run SLAVEOF cmd ..." >> $LOGFILE
$REDISCLI SLAVEOF 192.168.99.12 6379 >> $LOGFILE  2>&1
```

edit /etc/keepalived/scripts/redis_fault.sh  
```
# !/bin/bash
LOGFILE=/var/log/keepalived-redis-state.log 
echo "[fault]" >> $LOGFILE 
date >> $LOGFILE
```

edit /etc/keepalived/scripts/redis_master.sh  
```
#!/bin/bash
REDISCLI="/usr/local/bin/redis-cli"
LOGFILE="/var/log/keepalived-redis-state.log"
echo "[master]" >> $LOGFILE
date >> $LOGFILE
echo "Being master...." >> $LOGFILE 2>&1
echo "Run SLAVEOF cmd ..." >> $LOGFILE
$REDISCLI SLAVEOF 192.168.99.12 6379 >> $LOGFILE  2>&1
sleep 10 
echo "Run SLAVEOF NO ONE cmd ..." >> $LOGFILE
$REDISCLI SLAVEOF NO ONE >> $LOGFILE 2>&1
```

edit /etc/keepalived/scripts/redis_stop.sh  
```
# !/bin/bash
LOGFILE=/var/log/keepalived-redis-state.log 
echo "[stop]" >> $LOGFILE 
date >> $LOGFILE
```

#### [On Slave]  
edit **/etc/keepalived/keeplived.conf** for slave keepalived
```
global_defs {
	router_id redis
}
 
vrrp_script chk_redis {
	script "/etc/keepalived/scripts/check_redis.sh"
	interval 4
	weight -5
	fall 3  
	rise 2
}
 
vrrp_instance VI_REDIS {
	state BACKUP
	interface eth1
	virtual_router_id 11
	priority 99
	advert_int 1
	nopreempt
 
	authentication {
		auth_type PASS
		auth_pass 1111
	}
 
	virtual_ipaddress {
		192.168.99.10
	}
 
	track_script {
		chk_redis
	}
 
	notify_master /etc/keepalived/scripts/redis_master.sh
	notify_backup /etc/keepalived/scripts/redis_backup.sh
	notify_fault  /etc/keepalived/scripts/redis_fault.sh
	notify_stop   /etc/keepalived/scripts/redsi_stop.sh
}
```

**Edit script for redis HA**  
/etc/keepalived/scripts/check_redis.sh for checking slave reids
```
#!/bin/bash  
CHECK=`/usr/local/bin/redis-cli PING`  
if [ "$CHECK" == "PONG" ] ;then  
      echo $CHECK  
      exit 0  
else   
      echo $CHECK  
      service keepalived stop
      exit 1  
fi
```


edit /etc/keepalived/scripts/redis_backup.sh for master back online and being slave of new master  
```
#!/bin/bash
REDISCLI="/usr/local/bin/redis-cli"
LOGFILE="/var/log/keepalived-redis-state.log"
echo "[backup]" >> $LOGFILE
date >> $LOGFILE
echo "Being slave...." >> $LOGFILE 2>&1
sleep 15 
echo "Run SLAVEOF cmd ..." >> $LOGFILE
$REDISCLI SLAVEOF 192.168.99.11 6379 >> $LOGFILE  2>&1
```

edit /etc/keepalived/scripts/redis_fault.sh  
```
# !/bin/bash
LOGFILE=/var/log/keepalived-redis-state.log 
echo "[fault]" >> $LOGFILE 
date >> $LOGFILE
```

edit /etc/keepalived/scripts/redis_master.sh  
```
#!/bin/bash
REDISCLI="/usr/local/bin/redis-cli"
LOGFILE="/var/log/keepalived-redis-state.log"
echo "[master]" >> $LOGFILE
date >> $LOGFILE
echo "Being master...." >> $LOGFILE 2>&1
echo "Run SLAVEOF cmd ..." >> $LOGFILE
$REDISCLI SLAVEOF 192.168.99.11 6379 >> $LOGFILE  2>&1
sleep 10 
echo "Run SLAVEOF NO ONE cmd ..." >> $LOGFILE
$REDISCLI SLAVEOF NO ONE >> $LOGFILE 2>&1
```

edit /etc/keepalived/scripts/redis_stop.sh  
```
# !/bin/bash
LOGFILE=/var/log/keepalived-redis-state.log 
echo "[stop]" >> $LOGFILE 
date >> $LOGFILE
```

編輯完master/slave的scripts後  
記得用chmod a+x 把所有腳本設定為可執行  
並且在redis-server啟用的狀態下執行 check_redis.sh  
檢查腳本以及redis-server運作狀況  

**Start keepalived**  
**[On master and slave]**  
```bash=
sudo systemctl restart keepalived
```

#### Redis cluster測試  
1. Master/Slave寫入測試  

[On Master]  
```bash=
redis-cli set a hello
```  

[On Slave]  
```bash=
redis-cli get a
"hello"
```

2. VIP 寫入測試  
[On other machine]  

```bash=
redis-cli -h 100.67191.110 set b test
> OK
redis-cli -h 100.67191.110 get b
> "test"
```

3. HA 測試  

[On Redis 1]  
關閉redis server  
```bash
sudo pkill redis-server
```

測試針對VIP寫入與讀取，驗證master轉移到Redis 2  
[On other machine]  
```bash
redis-cli -h 100.67191.110 set c test
> OK
redis-cli -h 100.67191.110 get c
> "test"
```

[On Redis 2]  
```bash=
ip a
```
應該看到VIP 已經轉移到Redis 2上面  

回到Redis 1重新啟動Redis server和keepalived，驗證VIP 切換回Redis 1  
[On Redis 1]
```bash=
cd redis-4.0.6
sudo redis-server ./redis.conf
sudo systemctl restart keepalived
```
測試Redis 1是否能夠同步最新資訊  
[On Redis 1]
```bash=
redis-cli get c
> "test"
```
[On Redis 1]  
```bash=
ip a
```
應該看到VIP 已經轉移到Redis 1上面  

到此為止已經完成Redis HA環境的架設  



---


### Maria DB Cluster  





---


### NFS 架設  





---


### Harbor Cluster  