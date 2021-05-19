# InnoDb-group-replication

# Purge Existing
user@hostname$ apt purge mysql*

user@hostname$ rm -rf /etc/apparmor.d/abstractions/mysql /etc/apparmor.d/cache/usr.sbin.mysqld /etc/mysql /var/lib/mysql /var/log/mysql* /var/log/upstart/mysql.log* /var/run/mysqld updatedb


# Installation server
user@hostname$ apt install mysql-server

# create replication user
mysql > SET SQL_LOG_BIN=0;
mysql > CREATE USER 'repluser'@'%' IDENTIFIED BY 'password' REQUIRE SSL;
mysql > GRANT REPLICATION SLAVE ON *.* TO 'repluser'@'%';
mysql > FLUSH PRIVILEGES;
mysql > SET SQL_LOG_BIN=1;


# Slave replication user configuration
mysql > CHANGE MASTER TO MASTER_USER='repluser', MASTER_PASSWORD='password' FOR CHANNEL 'group_replication_recovery';


# /etc/mysql/my.cnf

[mysqld]

*//General replication settings*

gtid_mode = ON

enforce_gtid_consistency = ON

master_info_repository = TABLE

relay_log_info_repository = TABLE

binlog_checksum = NONE

log_slave_updates = ON

log_bin = binlog

binlog_format = ROW

transaction_write_set_extraction = XXHASH64

loose-group_replication_bootstrap_group = OFF

loose-group_replication_start_on_boot = OFF

loose-group_replication_ssl_mode = REQUIRED

loose-group_replication_recovery_use_ssl = 1

*//Shared replication group configuration*

loose-group_replication_group_name = "56bbe764-c69e-4d34-8e61-e7a124cf3e6c"

loose-group_replication_ip_whitelist = "x.x.x.x, x.x.x.x,  x.x.x.x"

loose-group_replication_group_seeds = "x.x.x.x:port, x.x.x.x:port, x.x.x.x:port"

*//Single or Multi-primary mode? Uncomment these two lines*

 for multi-primary mode, where any host can accept writes
 
loose-group_replication_single_primary_mode = OFF

loose-group_replication_enforce_update_everywhere_checks = ON

*//Host specific replication configuration

//server_id =*
bind-address = "current_node_ip"

report_host = "current_node_ip"

loose-group_replication_local_address = "current_node_ip:6606"


# Replicas

mysql > CHANGE REPLICATION SOURCE TO 
             SOURCE_HOST = "primary_ip",
             SOURCE_PORT = 3306,
             SOURCE_USER = "repluser",
             SOURCE_PASSWORD = "password",
             SOURCE_AUTO_POSITION = 1;

# Empty queries to catch up purged executions

mysql > SET GTID_NEXT='50cde6ea-b7e5-11eb-b44e-a62f21d46def:2';

        BEGIN;COMMIT;
        
        SET GTID_NEXT='AUTOMATIC';
        
# Troubleshooting incompatible execution entries

mysql > show global variables like "gtid_purged"

mysql > show global variables like "gtid_executed"

# Check status
mysql > select * from performance_schema.replication_group_members

# Start bootstrap primary
mysql > SET GLOBAL group_replication_bootstrap_group=ON;

mysql > START GROUP_REPLICATION;

mysql > SET GLOBAL group_replication_bootstrap_group=OFF;

# Each of the remaining
mysql > START GROUP_REPLICATION;

# Load balancing

user@hostname$ mysqlrouter --bootstrap root@localhost:3310 --directory /tmp/myrouter --conf-use-sockets --account routerfriend --account-create always
