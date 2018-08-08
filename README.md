# Harbor HA 安裝手冊

## Harbor HA 架構
![](https://i.imgur.com/ZcNEr5A.png)



| Role     | Node 1 IP     | Node 2 IP     | Node 3 IP    |
| -------- | ------------- | ------------- | ------------ | 
| Harbor   | 100.67.191.10 | 100.67.191.11 | N\A          |
| HarborLB | 100.67.191.119| 100.67.191.118| N\A          |
| HarborVIP| 100.67.191.201|               |              |
||||
| MariaDB  | 100.67.191.121| 100.67.191.122|100.67.191.123|
| MariaLB  | 100.67.191.129| 100.67.191.128| N\A          |
| MariaVIP | 100.67.191.120|               |              |
||||
| Redis    | 100.67.191.111| 100.67.191.112| N\A          |
| RedisVIP | 100.67.191.110|               |              |
||||
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
[Link to redis cluster setup guide](https://github.com/mJace/HarborHA/blob/master/redis)  


---


### Maria DB Cluster  






---


### NFS 架設  





---


### Harbor Cluster  
