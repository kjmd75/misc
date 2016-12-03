# Installing Mattermost on a single CentOS/RHEL server

Based on: https://docs.mattermost.com/install/prod-rhel-7.html

## OS update
1. yum update
1. yum upgrade

## MySQL Setup

### MySQL Install
1. wget http://dev.mysql.com/get/mysql57-community-release-el7-7.noarch.rpm
1. yum localinstall mysql57-community-release-el7-7.noarch.rpm
1. yum repolist enabled | grep "mysql.\*-community.\*"
  1. Verify the mysql 5.7 community server is in the list
1. yum install mysql-community-server
1. service mysqld start
1. service mysqld status
1. mysql --version
  1. Verify the mysql version is 5.7.x
1. grep 'temporary password' /var/log/mysqld.log  
  1. Find temp password to use for the next step
1. mysql_secure_installation
  1. Change the root password (from the password above)
  1. Remove the anonymous users
  1. Disable root login remotely
  1. Remove test database
  1. Reload privileges
  
### MySQL database setup for Mattermost
1. Login to the database and run several commands:
  1. mysql -u root -p 
  1. CREATE DATABASE mattermost;
  1. CREATE USER mmuser IDENTIFIED BY "password_here";
  1. GRANT ALL PRIVILEGES ON mattermost.* TO mmuser;

### Move data dir to /data01
1. mkdir /data01/mysql
1. chown mysql:mysql /data01/mysql
1. service mysqld stop
1. mv /var/lib/mysql/* /data01/mysql/.
1. vi /etc/my.cnf
  1. Change datadir to /data01/mysql
1. service mysqld start

### Setup MySQL backups
1. mkdir -p /data01/backup/mysql
1. mkdir /root/scripts
1. vi /root/scripts/mysql_backup.sh
  1. Paste the following into the backup script
  
    ``` bash
    #!/bin/sh

    #----------------------------------------------------
    # Simple MySQL backup script
    #----------------------------------------------------

    # (1) set up all the mysqldump variables
    FILE=/data01/backup/mysql/mysql.sql.`date +"%Y%m%d%H%M%S"`
    USER=root
    PASS=<root_passwd>

    # (2) do the mysql database backup (dump)
    mysqldump --opt --user=${USER} --password=${PASS} --all-databases > ${FILE}

    # (3) gzip the mysql database dump file
    gzip $FILE

    # (4) Delete backup files older than 7 days
    find /data01/backup/mysql/* -mtime +7 -exec rm {} \;

    # (5) show the user the result
    echo "${FILE}.gz was created:"
    ls -l ${FILE}.gz  
    ```
1. chmod 750 mysql_backup.sh
1. Add this to the root crontab (crontab -e):

``` cron
# MySQL Full Backup
0 0 * * * cd /root/scripts; ./mysql_backup.sh > mysql_backup.log 2>&1;
```



## Mattermost setup

### Mattermost install
1. cd /root
1. wget https://releases.mattermost.com/3.5.1/mattermost-3.5.1-linux-amd64.tar.gz
1. tar -xvzf mattermost-3.5.1-linux-amd64.tar.gz 
1. mv mattermost /opt
1. mkdir -p /data01/mattermost/data
1. useradd -r mattermost -U
1. chown -R mattermost:mattermost /data01/mattermost
1. chmod -R g+w /data01/mattermost
1. chown -R mattermost:mattermost /opt/mattermost
1. chmod -R g+w /opt/mattermost
1. usermod -aG mattermost root
1. vi /opt/mattermost/config/config.json
  1. Verify "DriverName" is set to "mysql"
  1. Update "DataSource" to use the correct mmuser password and hit localhost instead of dockerhost

### Mattermost service setup
1. touch /etc/systemd/system/mattermost.service
1. vi /etc/systemd/system/mattermost.service

``` bash
[Unit]
Description=Mattermost
After=syslog.target network.target mysqld

[Service]
Type=simple
WorkingDirectory=/opt/mattermost/bin
User=mattermost
ExecStart=/opt/mattermost/bin/platform
PIDFile=/var/spool/mattermost/pid/master.pid
LimitNOFILE=49152

[Install]
WantedBy=multi-user.target
```
1. chmod 664 /etc/systemd/system/mattermost.service
1. systemctl daemon-reload
1. chkconfig mattermost on
1. systemctl enable mattermost.service
1. systemctl start mattermost.service
1. systemctl status mattermost.service

## NGINX setup



