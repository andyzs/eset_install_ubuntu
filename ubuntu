#!/bin/bash

## Define variable
eraserver=https://download.eset.com/com/eset/apps/business/era/server/linux/latest/server-linux-x86_64.sh
eraagent=https://download.eset.com/com/eset/apps/business/era/agent/latest/agent-linux-x86_64.sh
rdsensor=https://download.eset.com/com/eset/apps/business/era/rdsensor/latest/rdsensor-linux-x86_64.sh
webconsole=https://download.eset.com/com/eset/apps/business/era/webconsole/latest/era.war
esetdb=127.0.0.1

## Setting locale
export DEBIAN_FRONTEND=noninteractive
locale-gen --purge en_US.UTF-8
echo -e 'LANG="en_US.UTF-8"\nLANGUAGE="en_US:en"\n' > /etc/default/locale
## Setting Repositories, Update and Upgrade system
sed -i 's/http://id.archive.ubuntu.com/http://kartolo.sby.datautama.net.id/g' /etc/apt/sources.list
apt -y update
apt -y upgrade
## Install  neccesary package
apt install -y dirmngr debconf wget default-jdk tomcat8 unixodbc libodbc1 xvfb cifs-utils libqtwebkit4 krb5-user winbind snmp ldap-utils libsasl2-modules-gssapi-mit samba squid ufw

## Install MySQL
if [[ -z "$esetdb" ]]; then
    echo "Variable not defined, db setup will be skipped" >&2
else
sudo apt-key adv --keyserver pool.sks-keyservers.net --recv-keys 5072E1F5
debconf-set-selections <<< "mysql-community-server mysql-server/default-auth-override select Use Legacy Authentication Method (Retain MySQL 5.x Compatibility)"
echo "deb http://repo.mysql.com/apt/ubuntu $(lsb_release -sc) mysql-5.7" | \
tee /etc/apt/sources.list.d/mysql.list
apt update
debconf-set-selections <<< \
  "mysql-community-server mysql-community-server/root-pass password eraadmin"
debconf-set-selections <<< \
  "mysql-community-server mysql-community-server/re-root-pass password eraadmin"
debconf-set-selections <<< \
  "mysql-community-server mysql-server/default-auth-override select Use Legacy Authentication Method (Retain MySQL 5.x Compatibility)"
DEBIAN_FRONTEND=noninteractive apt install -y mysql-server
fi
## Setup ODBC
if [ -e "mysql-connector-odbc-5.3.10-linux-glibc2.12-x86-64bit.tar.gz" ]; then
    echo 'File already exists' >&2
else
    wget https://downloads.mysql.com/archives/get/p/10/file/mysql-connector-odbc-5.3.10-linux-glibc2.12-x86-64bit.tar.gz  \
&& tar xvzf mysql-connector-odbc-5.3.10-linux-glibc2.12-x86-64bit.tar.gz
cp mysql-connector-odbc-5.3.10-linux-glibc2.12-x86-64bit/lib/libmyodbc5* /usr/lib/x86_64-linux-gnu/odbc/
cp -r mysql-connector-odbc-5.3.10-linux-glibc2.12-x86-64bit/bin/ /usr/local/bin
fi
mv /etc/odbcinst.ini /etc/odbcinst.ini.bak
cat <<EOF>> /etc/odbcinst.ini
[MySQL]
Description = ODBC for MySQL
Driver = /usr/lib/x86_64-linux-gnu/odbc/libmyodbc5w.so
Setup = /usr/lib/x86_64-linux-gnu/odbc/libodbcmyS.so
FileUsage = 1
EOF
/usr/local/bin/myodbc-installer -a -d -n "MySQL ODBC 5.3 Driver" -t "Driver=/usr/lib/x86_64-linux-gnu/odbc/libmyodbc5w.so"
/usr/local/bin/myodbc-installer -a d -n "MySQL ODBC 5.3" -t "Driver=/usr/lib/x86_64-linux-gnu/odbc/libmyodbc5a.so"
systemctl restart mysql
## Config MySQL
mv /etc/mysql/mysql.conf.d/mysqld.cnf /etc/mysql/mysql.conf.d/mysqld.bak
cat <<EOF>> /etc/mysql/mysql.conf.d/mysqld.cnf
# The MySQL database server configuration file.
[mysqld_safe]
socket          = /var/run/mysqld/mysqld.sock
nice            = 0
[mysqld]
# * Basic Settings
user            = mysql
pid-file        = /var/run/mysqld/mysqld.pid
socket          = /var/run/mysqld/mysqld.sock
port            = 3306
basedir         = /usr
datadir         = /var/lib/mysql
tmpdir          = /tmp
lc-messages-dir = /usr/share/mysql
skip-external-locking
# Instead of skip-networking the default is now to listen only on
# localhost which is more compatible and is not less secure.
bind-address            = 127.0.0.1
allow-suspicious-udfs = false
local_infile = 0
skip-grant-tables = false
skip_symbolic_links = yes
sql_mode ="STRICT_ALL_TABLES,[other_values]"
sql_mode = Prevents GRANT statement from User with blank password can"NO_AUTO_CREATE_USER,[other_values]"

# * Fine Tuning
key_buffer_size         = 16M
max_allowed_packet      = 500M
innodb_log_file_size = 100M
innodb_log_files_in_group = 4
innodb_lock_wait_timeout=600
thread_stack            = 192K
thread_cache_size       = 8
# This replaces the startup script and checks MyISAM tables if needed
# the first time they are touched
myisam-recover-options  = BACKUP
#max_connections        = 100
#table_open_cache       = 64
#thread_concurrency     = 10
# * Query Cache Configuration
query_cache_limit       = 1M
query_cache_size        = 16M
# * Logging and Replication
# Both location gets rotated by the cronjob.
# Be aware that this log type is a performance killer.
# As of 5.1 you can enable the log at runtime!
#general_log_file        = /var/log/mysql/mysql.log
#general_log             = 1
# Error log - should be very few entries.
log_error = /var/log/mysql/error.log
log_error_verbosity = 3
log-warnings = 2
log-raw = off
# Here you can see queries with especially long duration
#slow_query_log         = 1
#slow_query_log_file    = /var/log/mysql/mysql-slow.log
#long_query_time = 2
#log-queries-not-using-indexes

# The following can be used as easy to replay backup logs or for replication.
# note: if you are setting up a replication slave, see README.Debian about
#       other settings you may need to change.
#server-id              = 1
#log_bin                        = /var/log/mysql/mysql-bin.log
expire_logs_days        = 10
max_binlog_size   = 100M
EOF

systemctl restart mysql 

##Config squid
mv /etc/squid/squid.conf /etc/squid/squid.bak


cat <<EOF>> /etc/squid/squid.conf
# _____Squid config for eset_____
acl SSL_ports port 443
acl Safe_ports port 80 # http
acl Safe_ports port 21 # ftp
acl Safe_ports port 443 # https
acl Safe_ports port 70 # gopher
acl Safe_ports port 210 # wais
acl Safe_ports port 1025-65535 # unregistered ports
acl Safe_ports port 280 # http-mgmt
acl Safe_ports port 488 # gss-http
acl Safe_ports port 591 # filemaker
acl Safe_ports port 777 # multiling http
acl Safe_ports port 53
acl CONNECT method CONNECT
# Deny requests to certain unsafe ports
http_access deny !Safe_ports
# Deny CONNECT to other than secure SSL ports
http_access deny CONNECT !SSL_ports
# Only allow cachemgr access from localhost
http_access allow localhost manager
http_access deny manager
#detection Update
acl allowed dstdomain .eset.com .eset.sk .amazonaws.com .mailshell.net .eset.eu .trafficmanager.net .azure.com .esetsoftware.com
http_access allow allowed
http_access deny all
# Squid normally listens to port 3128
http_port 3128
# Uncomment and adjust the following to add a disk cache directory.
cache_dir ufs /var/spool/squid 5000 16 256 max-size=10000000
cache_mem 1000 MB
# Leave coredumps in the first cache dir
coredump_dir /var/spool/squid
# Add any of your own refresh_pattern entries above these.
refresh_pattern ^ftp: 1440 20% 10080
refresh_pattern ^gopher: 1440 0% 1440
refresh_pattern -i (/cgi-bin/|\?) 0 0% 0
refresh_pattern (Release|Packages(.gz)*)$ 0 20% 2880
# example lin deb packages
#refresh_pattern (\.deb|\.udeb)$ 129600 100% 129600
refresh_pattern . 0 20% 4320
# _____End of Squid Config_____
EOF
systemctl enable squid && systemctl restart squid

## Download ESET and others needed Package
if [ -e "eraserver.sh" ]; then
    echo 'File already exists' >&2
else
    wget -O eraserver.sh $eraserver && chmod +x eraserver.sh 
fi
if [ -e "eraagent.sh" ]; then
    echo 'File already exists' >&2
else
    wget -O eraagent.sh $eraagent && chmod +x eraagent.sh
fi
if [ -e "rdsensor.sh" ]; then
    echo 'File already exists' >&2
else
    wget -O rdsensor.sh $rdsensor && chmod +x rdsensor.sh
fi
if [ -e "era.war" ]; then
    echo 'File already exists' >&2
else
    wget -O era.war $webconsole 
fi

## Setup Tomcat
mv ./era.war /var/lib/tomcat8/webapps/
systemctl restart tomcat8


## Install Eraserver
bash ./eraserver.sh --skip-license --db-driver=MySQL --db-hostname=127.0.0.1 --db-port=3306 --db-admin-username=root --db-admin-password=eraadmin --server-root-password=eraadmin --db-user-username=root --db-user-password=eraadmin --cert-hostname="*" --enable-imp-program

## Install Agent
bash ./eraagent.sh --skip-license --enable-imp-program --hostname=127.0.0.1 --port=2222 --webconsole-hostname=127.0.0.1 --webconsole-port=2223 --webconsole-user=administrator --webconsole-password="eraadmin" --cert-auto-confirm
bash ./rdsensor.sh --skip-license

## Hardening
ufw enabled
ufw allow 1717/tcp
ufw allow 2222/tcp
ufw allow 2223/tcp
ufw allow 139/tcp
ufw allow 145/tcp
ufw allow 3306/tcp
ufw allow 3128/tcp
sed -i 's/#Port 22/Port 1717/g' /etc/ssh/sshd_config
systemctl restart sshd
