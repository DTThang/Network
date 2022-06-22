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


        # nội dung
        date.timezone = Asia/Ho_Chi_Minh
        memory_limit = 512M
        max_execution_time = 60    

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




