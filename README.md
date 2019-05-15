# LinOTP-mariaDB-HA
LinOTP mariaDB replication

  APP setup

1.
Add LinOTP repo /etc/yum.repos.d/linotp.repo
[linotp]
name=KeyIdentity LinOTP Packages for Enterprise Linux 7 - $basearch
baseurl=http://linotp.org/rpm/el7/linotp/x86_64
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-LINOTP-7

[linotp-dependencies]
name=KeyIdentity LinOTP Packages required for Enterprise Linux 7
baseurl=http://linotp.org/rpm/el7/dependencies/x86_64
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-LINOTP-7

2.
yum install LinOTP_mariadb mariadb-server yum-plugin-versionlock LinOTP_apache --nogpgcheck -y

3.
yum versionlock python-repoze-who

4.
cp /etc/linotp2/ssl_linotp.conf.template /etc/linotp2/ssl_linotp.conf
cp /etc/linotp2/linotp.ini.example /etc/linotp2/linotp.ini
htdigest /etc/linotp2/admins "LinOTP2 admin area" admin
dd if=/dev/urandom of=/etc/linotp2/encKey bs=1 count=128
chown linotp /etc/linotp2/encKey && chmod 640 /etc/linotp2/encKey
copy encKey to second server


  DB setup

1.
mysql_secure_installation

2.
mysql -u root -p -e "CREATE DATABASE IF NOT EXISTS LINOTP; grant all privileges on LINOTP.* to linotp@'localhost' identified by 'linotpegcp'; flush privileges"

3.
On primary-server
GRANT REPLICATION SLAVE ON *.* TO 'repl2_user'@'primary-server' IDENTIFIED BY 'marija';
FLUSH PRIVILEGES;
On secundary-server
GRANT REPLICATION SLAVE ON *.* TO 'repl1_user'@'secundary-server' IDENTIFIED BY 'marija';
FLUSH PRIVILEGES;

4.
Add to my.sql conf...
On primary-server
# replication
server-id = 1
replicate-same-server-id = 0
auto-increment-increment = 2
auto-increment-offset = 1
replicate-do-db = LINOTP
log-bin = /var/log/mariadb/mysql-bin.log
binlog-do-db = LINOTP
relay-log = /var/lib/mysql/slave-relay.log
relay-log-index = /var/lib/mysql/slave-relay-log.index
On secundary-server
server-id = 2
replicate-same-server-id = 0
auto-increment-increment = 2
auto-increment-offset = 2
replicate-do-db = LINOTP
log-bin = /var/log/mariadb/mysql-bin.log
binlog-do-db = LINOTP
relay-log = /var/lib/mysql/slave-relay.log
relay-log-index = /var/lib/mysql/slave-relay-log.index

5.
Restart mariaDB on primary-server and secundary-server...
USE LINOTP;
FLUSH TABLES WITH READ LOCK;
SHOW MASTER STATUS;

+------------------+----------+--------------+------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| mysql-bin.000001 |      245 | LINOTP       |                  |
+------------------+----------+--------------+------------------+

UNLOCK TABLES;

6.
on primary-server...
stop slave;
change master to master_host='secundary-server', master_user='repl1_user', master_password='marijarep',
master_log_file='mysql-bin.000001', master_log_pos=245;
start slave;
on secundary-server...
stop slave;
show slave status\G$
change master to master_host='primary-server', master_user='repl2_user', master_password='marijarep',
master_log_file='mysql-bin.000001', master_log_pos=245;
start slave;

show slave status\G$

7.
on primary-server...
https://primary-server/system/setConfig?linotp.enableReplication=true&session=1c0ca3e419a9bd96cb2db7616cc7f2c3e8689df075aed2a6e88d044780f13db7
