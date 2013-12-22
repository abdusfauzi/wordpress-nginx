# Wordpress on Nginx + PHP-FPM
This is my personal steps in preparing my VPS/Dedicated server for running WordPress installation. Due to the nature of Nginx, .htaccess is not supported. We will look into configuration to imimate the how .htaccess normally works.

All files given on http://paste.laravel.com has been put into its respective files in __etc__ folder above.

__Make sure to replace \<username\> or \<website\> with your own__

### STEP 1: update to latest CentOS version
* ssh root@your-ip-address
* yum update
* __Check version__
* cat /etc/redhat-release
* __Cetup hostname__
* echo "HOSTNAME=\<yourhostname\>" >> /etc/sysconfig/network
* hostname "\<yourhostname\>"
* __Update /etc/hosts__
* nano /etc/hosts
* add new line: \<ip address\>    \<yourhostname\>.example.com    \<yourhostname\>
* add new line: \<ipv6 address\>    \<yourhostname\>.example.com    \<yourhostname\>

### STEP 2: get important Repo
* __Install important repo__
* rpm -ivh http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
* rpm -Uvh http://rpms.famillecollet.com/enterprise/remi-release-6.rpm
* __Install yum-priorities for repo config__
* yum install yum-priorities


### STEP 3: install mysql, nginx, php-fpm, memcached
* __Install mysql__
* yum install mysql mysql-server
* chkconfig --levels 235 mysqld on
* service mysqld start
* __Check mysqld server in running__
* netstat -tap | grep mysql
* __Run secure installation (to set password to root)__
* mysql_secure_installation
* __Set password__
* __Now, lets install nginx__
* yum install nginx
* chkconfig --levels 235 nginx on
* service nginx start
* ifconfig eth0 | grep inet | awk '{ print $2 }'
* __Visit your ip address to check on nginx static page__
* __Now, lets install php-fpm__
* yum install php-fpm php-cli php-mysql php-gd php-imap php-ldap php-odbc php-pear php-xml php-xmlrpc php-magickwand php-magpierss php-mbstring php-mcrypt php-mssql php-shout php-snmp php-soap php-tidy php-pecl-apc sendmail sendmail-cf
* __edit /etc/php.ini to set cgi.fix_pathinfo=0;__
* nano /etc/php.ini
* > cgi.fix_pathinfo=0;
* __Edit timezone to your location (Asia/Kuala_Lumpur)__
* > date.timezone = "Asia/Kuala_Lumpur"
* ln -sf /usr/share/zoneinfo/Asia/Kuala_Lumpur /etc/localtime
* chkconfig --levels 235 php-fpm on
* service php-fpm start
* chkconfig --levels 235 sendmail on
* service sendmail start
* __Now, lets install memcached__
* yum install memcached php-memcached
* nano /etc/sysconfig/memcached
* > OPTIONS="-l 127.0.0.1”
* chkconfig --levels 235 memcached on
* service memcached start

### STEP 4: new user for FTP and SSH
* useradd \<username\>
* passwd \<username\>
* cd /srv
* mkdir www
* cd www
* mkdir \<website\>
* mkdir \<website\>/html
* chown -R user:usergroup \<website\>

### STEP 5: configure nginx, php-fpm, php session to memcached
* __Edit nginx configuration file__
* nano /etc/nginx/nginx.conf
* > worker_processes    8;
* > keeplive_timeout    2;
* __Easier, follow this format http://paste.laravel.com/15PT__
* __Here, we set some configuration for php-fpm to run on socket__
* __Then, we edit the default virtual host configuration__
* nano /etc/nginx/conf.d/default.conf
* __Follow this format http://paste.laravel.com/162K__
* __Now, add those global/… config files__
* cd /etc/nginx
* mkdir global
* cd global
* nano restrictions.conf => http://paste.laravel.com/15PY
* nano wordpress.conf => http://paste.laravel.com/162I
* nano w3-total-cache.conf => http://paste.laravel.com/15Q6
* service nginx restart
* __Now, we edit php-fpm configuration__
* nano /etc/php-fpm.d/www.conf
* > listen = /tmp/php-fpm.sock
* > user = \<username\>
* > group = \<username\>
* > php_value[session.save_handler] = memcached
* > php_value[session.save_path] = “127.0.0.1:11211"
* service php-fpm restart
* service memcached restart

### STEP 6: install FTP (vsftpd)
* yum install vsftpd
* nano /etc/vsftpd/vsftpd.conf
* > anonymous_enable=NO
* > chroot_local_user=YES
* add => user_config_dir=/etc/vsftpd/vsftpd_user_conf
* add => use_localtime=YES
* __Save__
* mkdir /etc/vsftpd/vsftpd_user_conf
* nano /etc/vsftpd/vsftpd_user_conf/<username>
* > dirlist_enable=YES
* > download_enable=YES
* > local_root=/srv/www/<website>
* > write_enable=YES
* __Save__
* service vsftpd restart

### STEP 7: install phpMyAdmin
* yum install phpmyadmin
* __Now, create new mysql user since root has been denied__
* mysql -u root -p
* CREATE USER ‘\<username\>'@'localhost' IDENTIFIED BY ‘\<password\>’;
* GRANT ALL PRIVILEGES ON * . * TO ‘\<username\>'@'localhost’;
* FLUSH PRIVILEGES;
* exit

### STEP 8: Grant <username> to sudoers
* usermod -a -G wheel \<username\>
* visudo
* __Uncomment %wheel lines__
* __Add new line below root   ALL=(ALL)  ALL__
* \<username\>    ALL=(ALL)   ALL 
* __ESC key__
* :wq

### STEP 9: install WordPress files
* su \<username\>
* cd /srv/www/\<website\>/html
* wget http://wordpress.org/latest.tar.gz
* tar -xzvf latest.tar.gz
* mv wordpress/* ./
* rmdir wordpress
* rm latest.tar.gz
* __Visit website and install__

### STEP 10: install NewRelic
* rpm -Uvh http://yum.newrelic.com/pub/newrelic/el5/x86_64/newrelic-repo-5-3.noarch.rpm
* yum install newrelic-php5
* newrelic-install install
* yum install newrelic-sysmond
* nano /etc/php.d/newrelic.ini
* > newrelic.appname = "\<website name\>"
* service php-fpm restart
* service newrelic-sysmond start

###vSTEP 11: Disable root SSH login and change Port 22 to Port 215
* nano /etc/ssh/sshd_config
* > Port 215
* > PermitRootLogin no
* service sshd restart

### STEP 12: Enable firewall settings
* iptables -L
* nano /etc/iptables.firewall.rules
* __Fill up with this http://paste.laravel.com/164b__
* iptables-restore < /etc/iptables.firewall.rules
* /sbin/service iptables save
* __You will face raw nat filter error (http://blog.btnotes.com/articles/606.html)__
* mv /etc/init.d/iptables /etc/init.d/iptables.bkp
* nano /etc/init.d/iptables
* __Paste this content http://paste.laravel.com/164n__
* __SAVE__
* service iptables restart

### STEP 13: install Fail2Ban for failed login access (bruteforce)
* yum install fail2ban
* configure setting: http://www.fail2ban.org/wiki/index.php/MANUAL_0_8#Configuration

----

## References
* [Installing Nginx with PHP5 and PHP-FPM and MySQL support on Centos 6.4](http://www.howtoforge.com/installing-nginx-with-php5-and-php-fpm-and-mysql-support-on-centos-6.4)
* [How To Create A New User & Grant Permissions in MySQL](https://www.digitalocean.com/community/articles/how-to-create-a-new-user-and-grant-permissions-in-mysql)
* [Securing Your Server](https://library.linode.com/securing-your-server)







