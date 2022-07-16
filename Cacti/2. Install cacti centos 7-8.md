# 1. Chuẩn bị
**Nếu gặp lỗi**
```
CentOS Linux 8 - AppStream                             64  B/s |  38  B     00:00
Error: Failed to download metadata for repo 'appstream': Cannot prepare internal mirrorlist: No URLs in mirrorlist`
```

Nhập:
```
cd /etc/yum.repos.d/
sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*
yum update -y
```
- Gói cacti có sẵn trong EPEL repository for Centos 8

        dnf install -y epel-release
        dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm

- Cài đặt Apache

        dnf install httpd -y

- Cài đặt SNMP và RRDTool

        dnf install -y net-snmp net-snmp-utils net-snmp-libs rrdtool

- Cài đặt mariadb database cho base repository

        dnf install -y mariadb-server mariadb

- Cài đặt PHP và các gói PHP yêu cầu

        dnf install -y php php-xml php-session php-sockets php-ldap php-gd php-json php-mysqlnd php-gmp php-mbstring php-posix php-snmp php-intl

- Khởi động và cho phép dịch vụ khởi chạy khi hệ thống start-up
       
```
systemctl start httpd
systemctl start snmpd
systemctl start mariadb
systemctl enable httpd
systemctl enable snmpd
systemctl enable mariadb
```

# 2. Cấu hình database
- Nên thay đổi cài đặt biến của Mysql để có hiệu suất tốt hơn. Chỉnh sửa file cấu hình tùy thuộc vào hệ điều hành 

- Chỉnh sửa file /etc/my.cnf.d/mariadb-server.cnf  

        vim /etc/my.cnf.d/mariadb-server.cnf  

```
# Thêm biến vào trường [mysqld]
collation-server=utf8mb4_unicode_ci
character-set-server=utf8mb4
max_heap_table_size=32M
tmp_table_size=32M
join_buffer_size=64M
# 25% Of Total System Memory
innodb_buffer_pool_size=1GB
# pool_size/128 for less than 1GB of memory
innodb_buffer_pool_instances=10
innodb_flush_log_at_timeout=3
innodb_read_io_threads=32
innodb_write_io_threads=16
innodb_io_capacity=5000
innodb_io_capacity_max=10000
```
- Khởi động lại dịch vụ  

        systemctl restart mariadb

# 3. Tạo Cacti database
- Nhập  
```
# mysql -u root -p
```
```
create database cacti;
GRANT ALL ON cacti.* TO cactiuser@localhost IDENTIFIED BY 'cactipassword';
flush privileges;
exit

```

- cactiuser phải có quyền truy cập vào bảng mysql.time_zone_name. Để làm điều đó, nhập mysql_test_data_timezone.sql vào cơ sở dữ liệu mysql.

        mysql -u root -p mysql < /usr/share/mariadb/mysql_test_data_timezone.sql
        


Sau đó login vào mysql 

        mysql -u root -p

Cấp quyền cho cactiuser
```
GRANT SELECT ON mysql.time_zone_name TO cactiuser@localhost;
flush privileges;
exit
```

# 4. Cài đặt và cấu hình cacti

- Cài đặt cacti

        dnf install cacti -y 

- Nhập database mặc định vào cacti database       

        mysql cacti < /usr/share/doc/cacti/cacti.sql -u cactiuser -p
```
user password : cactipassword
```

- Chỉnh sửa file cấu hình để chỉ định loại database, name, hostname, user và password

        vim /usr/share/cacti/include/config.php

```
/*
* Make sure these values reflect your actual database/host/user/password
*/
$database_type = 'mysql';
$database_default = 'cacti';
$database_hostname = 'localhost';
$database_username = 'cactiuser';
$database_password = 'cactipassword';
$database_port = '3306';
```
- Chỉnh sửa file  /etc/cron.d/cacti  để quét thông tin mỗi ​​năm phút một lần.

        vim /etc/cron.d/cacti

```        
*/5 * * * *     apache  /usr/bin/php /usr/share/cacti/poller.php > /dev/null 2>&1
```

- Chỉnh sửa file  cấu hình Apache để thực hiện cài đặt từ xa.

        vim /etc/httpd/conf.d/cacti.conf
        
```
Alias /cacti    /usr/share/cacti
<Directory /usr/share/cacti/>
<IfModule mod_authz_core.c>
# httpd 2.4
Require all granted  #
</IfModule>
<IfModule !mod_authz_core.c>
# httpd 2.2
Order deny,allow
Deny from all
Allow from all   #
</IfModule>
</Directory>
```
- Thiết lập time zone 

        vim /etc/php.ini

```
date.timezone = Asia/Ho_Chi_Minh
memory_limit = 512M
max_execution_time = 60    
```
- Khởi động lại dịch vụ  
    
        systemctl restart httpd
        systemctl restart php-fpm

- Cấu hình firewall

```
firewall-cmd --permanent --add-service=http
firewall-cmd --reload
```
- Cấu hình SELinux

        setenforce 0

# 5. Thiết lập web interface
- Truy cập cacti thông qua web interfice

Đường dẫn http://ip-address/cacti

user/pass: admin/admin




# Install on centos 7

- Update System

                yum update -y

- Install Apache 

                yum -y install httpd httpd-devel

- Restart Apache service and Enable Apache auto startup when server startup.
```
systemctl restart httpd
systemctl enable httpd
```
- Install Net-SNMP and RRDTools

                yum -y install net-snmp net-snmp-utils net-snmp-libs rrdtool

- Restart Apache service and Enable Apache auto startup when server startup.
```
systemctl restart snmpd
systemctl enable snmpd
```
- Install PHP and PHP Extensions

The current Cacti version requires PHP 7.2+, so we need to install from Remi repository because by default the PHP version available in base OS repository is version 5.4.   Create Remi Repository with the below command.

                yum install -y https://rpms.remirepo.net/enterprise/remi-release-7.rpm

  - Install PHP 7.3 from the Remi repository.

                yum install -y --enablerepo=remi-php73 php php-xml php-session php-sockets php-ldap php-gd php-gmp php-intl php-mbstring php-mysqlnd php-pdo php-process php-snmp

  - PHP recommended changing the following settings in /etc/php.ini file. Before making a change please verify the timezone on our system.

                ls -l /etc/localtime

                vi /etc/php.ini
```
date.timezone = Asian/Phnom_Penh
memory_limit = 512M
max_execution_time = 60
```
- Install MariaDB

By default, MariaDB v5.4 is available in the based Centos repository. The current Cacti version recommended MariaDB v5.6+. We will install MariaDB v10+, so we need to create a new repository for MariaDB. Create a new repository.

        vim /etc/yum.repos.d/mariadb.repo
```
[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/10.4/centos7-amd64
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1
```
- Install MariaDB with the below commands.

                yum install -y MariaDB-server MariaDB-client

Restart MariaDB and enable to auto-start when the server starts.
```
systemctl restart mariadb
systemctl enable mariadb
```
- Create Database

  - The password is not set for MariaDB, so we need to so secure it with the command below.

                mysqladmin -u root password your_strong_password

  - Login into our database with a password just set at the moment.

                mysql -u root -p

  - Create a database and username for Cacti.
  ```
  create database cacti;
  GRANT ALL ON cacti.* TO cactiuser@localhost IDENTIFIED BY 'p@$$w0rd';
  flush privileges;
  exit;
  ```

  - we need to grant access to the MySQL TimeZone database for user Cacti so that the database is populated with global TimeZone information. Import database to mysql_test_data_timezone.sql first.

                mysql -u root -p mysql < /usr/share/mysql/mysql_test_data_timezone.sql
                password: p@$$w0rd
  - Login into our database again.

                mysql -u root -p

  - Grant permission to cacti user.
  ```
  GRANT SELECT ON mysql.time_zone_name TO cactiuser@localhost;
  flush privileges;
  exit;
  ```
- Optimize Database

To improve performance Cacti recommended to optimize some setting in database. Edit /etc/my.cnf.d/server.cnf file.

                vi /etc/my.cnf.d/server.cnf 

Add the below lines under the [mysqld] section.
```
# this is only for the mysqld standalone daemon
[mysqld]
collation-server = utf8mb4_unicode_ci
character-set-server=utf8mb4
max_heap_table_size = 64M
tmp_table_size = 64M
join_buffer_size = 64M
innodb_file_format = Barracuda
innodb_large_prefix = 1
innodb_flush_log_at_timeout = 3
innodb_buffer_pool_size = 1GB
innodb_buffer_pool_instances = 10
innodb_read_io_threads = 32
innodb_write_io_threads = 16
innodb_io_capacity = 5000
innodb_io_capacity_max = 10000
```
- Install and Configure Cacti

  - Install Cacti with yum command, it will browse for the latest version of Cacti from the repository.

                yum -y install cacti

  - Import the default database to the cacti database with the password we have set for our database.

                mysql -u root -p cacti < /usr/share/doc/cacti-*/cacti.sql

  - Edit "include/config.php" and specify the database type, name, host user, and password for our Cacti configuration.

                vim /usr/share/cacti/include/config.php
```
/* make sure these values reflect your actual database/host/user/password */
$database_type = "mysql";
$database_default = "cacti";
$database_hostname = "localhost";
$database_username = "cactiuser";
$database_password = "p@$$w0rd";
$database_port = "3306";
$database_ssl = false;
```
- Create a Cron Job

  - Cacti need to run its query script to collect information from end devices that we monitor regularly. Mostly, we schedule Cacti to run the poller script every 5 minutes. If we install Cacti from yum command file in /etc/cron.d/cacti is already there by default.

                vim /etc/cron.d/cacti

  - Uncomment the following line.

                */5 * * * * apache /usr/bin/php /usr/share/cacti/poller.php > /dev/null 2>&1

  - Allow remote access

    - When we install Cacti from yum command, it allows the only the localhost to access. To allow remote access we need to change Apache configuration.

                vim /etc/httpd/conf.d/cacti.conf

    Change  “Require host localhost” to “Require all granted” and “Allow from localhost” to “Allow from all.”

```
Alias /cacti /usr/share/cacti
<Directory /usr/share/cacti/>
            <IfModule mod_authz_core.c>
                         # httpd 2.4
                         Require all granted
            </IfModule>
            <IfModule !mod_authz_core.c>
                         # httpd 2.2
                         Order deny,allow
                         Deny from all
                         Allow from all
            </IfModule>
</Directory>
```
-  Allow Firewalld

```
firewall-cmd --permanent --add-service=http
firewall-cmd --reload
```
- Disable SELinux

```
sed -i 's/enforcing/disabled/g' /etc/selinux/config
setenforce 0
```
- Setup Cacti

Open favorite web browser and type http://server_ip/cacti (our service IP, or domain name). The default username and password of Cacti are (user = admin, password = admin).