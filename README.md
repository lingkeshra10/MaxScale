# MaxScale

MariaDB MaxScale has been supported as database load balancer. MariaDB MaxScale allows for master/slave deployments with high availability, automatic failover, manual switchover and automatic rejoin. If the master fails, MariaDB MaxScale can automatically reconfigure it as a slave to the new master. In addition, administrator can perform a manual switchover to change the master on demand.

Functions of the MariaDB MaxScale mechanism

•	Master Detection
The monitor is now less likely to suddenly change the master server, even if another server has more slaves than the current master. The DBA can force a master reselection by setting the current master read-only, or by removing all it slaves if the master is down.

•	Switchover New Master Auto selection
The switchover command can now with just the monitor instance name as parameter. In this case the monitor will automatically select a server for promotion.

•	Replication Lag Detection
The replication lag measurement now simply reads the Seconds_Behind_Master- field of the slave status output slaves. The slave calculates this value by comparing the timestamp in the binlog event the slave is currently processing to the slave’s own clock. If a slave has multiple slave connections, the smallest lag is used.

•	Automatic Switchover After Low Disk Space Detection
With the recent MariaDB server versions, the monitor can now check the disk space on the backend and detect if the server is running low. When this happens, the monitor can be set automatically switchover from a master low on disk space. Slaves can also be set to maintenance mode. The disk space is also factor which is considered when selecting which new master to promote.

•	Replication Reset Feature
The reset-replication monitor command delete all slave connections and binary logs and sets up replication. Useful when data is sync but gtid are not.

•	Scheduled events handling in Failover/Switchover/Rejoin
Server events launched by the event scheduler thread are now handled during cluster modification operations.

•	External Master Support
The monitor can detect if a server in the cluster is replicating form an external master (a server that is not being monitored by the MaxScale monitor). If the replicating server is the cluster master server, then the cluster itself is considered to have an external master. If a failover/switchover happens, the new master is set to replicate from the cluster external master server. The Username and password for the replication are defined in replication_user and replication_password. The address and port used are the ones shown by SHOW ALL SLAVE Status on the old cluster master server. In the case of switchover, the old master also stops replicating from the external server to preserve the topology. After the failover the new master is replicating from the external master. If the failed old master comes back online, it is also replicating from the external server. To normalize the situation, either have auto_rejoin or manually execute a rejoin. This will redirect the old master to the current cluster master.

Steps to install the MaxScale in CentOS/AlmaLinux

1. First, we need to install the MaxScale on the server.
```
$ yum install maxscale
```

2. Next need to create the MaxScale config file which is maxscale.cnf on the **/etc** location.
```
$ cd /etc
$ touch /etc/ maxscale.cnf
```

3. There are two configurations one is multi-master replication setup configuration and master-slave replication setup configuration.

•	Multi-Master Replication

```
//Global parameters Master-Master Replication
[maxscale]
threads = auto
log_augmentation = 1
ms_timestamp = 1
syslog = 1

[Splitter-Service]
type=service
router=readwritesplit
servers=master1,master2
user=maxscaleuser //The username that will monitor the DB
password=maxscalepassword //The monitor username's password

[Splitter-Listener]
type=listener
service=Splitter-Service
protocol=MySQLClient
port=3306

[Replication-Monitor]
type=monitor
module=mariadbmon
servers=master1,master2
user=maxscaleuser //The username that will monitor the DB
password=maxscalepassword //The monitor username's password
monitor_interval=200
detect_stale_master=1
detect_replication_lag=true

[master1]
type=server
address=<monitor_server_ip_address_1> //The monitor server IP address
port=3306
protocol=MySQLBackend

[master2]
type=server
address=<monitor_server_ip_address_2> //The monitor server IP address
port=3306
protocol=MySQLBackend
```

•	Master-Slave Replication

```
// Global parameters Master-Slave Replication
[maxscale]
threads = auto
log_augmentation = 1
ms_timestamp = 1
syslog = 1

// Server definitions
[server1]
type=server
address=<monitor_server_ip_address_1> //The monitor server IP address
port=3306
protocol=MariaDBBackend

[server2]
type=server
address=<monitor_server_ip_address_2>  //The monitor server IP address
port=3306
protocol=MariaDBBackend

// Monitor for the servers
[MariaDB-Monitor]
type=monitor
module=mariadbmon
servers=server1,server2
user=maxscaleuser //The username that will monitor the DB
password=maxscalepassword //The monitor username's password
monitor_interval=2000

// Service definitions
[Read-Only-Service]
type=service
router=readconnroute
servers=server2
user=maxscaleuser // The username that will monitor the DB
password=maxscalepassword // The monitor username's password
router_options=slave

[Read-Write-Service]
type=service
router=readwritesplit
servers=server1
user=maxscaleuser // The username that will monitor the DB
password=maxscalepassword // The monitor username's password

// Listener definitions for the services
[Read-Only-Listener]
type=listener
service=Read-Only-Service
protocol=MariaDBClient
port=4008

[Read-Write-Listener]
type=listener
service=Read-Write-Service
protocol=MariaDBClient
port=4006
```

4. Next, we need to execute below SQL commands in the monitoring Database.
```
mariadb[none]> CREATE USER 'maxscaleuser'@'%' IDENTIFIED BY 'maxscalepassword';
mariadb[none]> GRANT SELECT ON mysql.user TO 'maxscaleuser'@'%';
mariadb[none]> GRANT SELECT ON mysql.db TO 'maxscaleuser'@'%';
mariadb[none]> GRANT SELECT ON mysql.tables_priv TO 'maxscaleuser'@'%';
mariadb[none]> GRANT SELECT ON mysql.roles_mapping TO 'maxscaleuser'@'%';
mariadb[none]> GRANT SHOW DATABASES ON *.* TO 'maxscaleuser'@'%';
mariadb[none]> GRANT SUPER, RELOAD, PROCESS, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO maxscaleuser@'%';
mariadb[none]> GRANT SELECT ON mysql.* TO 'maxscaleuser'@'%';
```

5. After successfully add monitor user on the database, we need to start the maxscale services.
```
// To enable the maxscale service
$ systemctl enable maxscale
// To start the maxscale service
$ systemctl start maxscale
// To check the maxscale status
$ systemctl status maxscale
```

6.	After starting the maxscale services, we can monitor the Databases that have been added inside the maxscale monitoring service.

```
$ maxctrl list servers
```



