# Wordpress on Nginx + PHP-FPM

http://www.howtoforge.com/installing-nginx-with-php5-and-php-fpm-and-mysql-support-on-centos-6.4
https://www.digitalocean.com/community/articles/how-to-create-a-new-user-and-grant-permissions-in-mysql
https://library.linode.com/securing-your-server



STEP 1: update to latest CentOS version
ssh root@106.186.117.160
yum update
check version
cat /etc/redhat-release
setup hostname
echo “HOSTNAME=<yourhostname>” >> /etc/sysconfig/network
hostname “<yourhostname>”
Update /etc/hosts
nano /etc/hosts
add new line: <ip address>    <yourhostname>.example.com    <yourhostname>
add new line: <ipv6 address>    <yourhostname>.example.com    <yourhostname>

STEP 2: get important Repo
install important repo
rpm -ivh http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
rpm -Uvh http://rpms.famillecollet.com/enterprise/remi-release-6.rpm

install yum-priorities for repo config
yum install yum-priorities

// edit [epel] repo and set priority=10 below enable=1
// nano /etc/yum.repos.d/epel.repo

// edit [remi] repo and set priority=10 and set enable=1
// nano /etc/yum.repos.d/remi.repo

STEP 3: install mysql, nginx, php-fpm, memcached
install mysql
yum install mysql mysql-server
chkconfig --levels 235 mysqld on
service mysqld start

check mysqld server in running
netstat -tap | grep mysql

run secure installation (to set password to root)
mysql_secure_installation
set password

now, lets install nginx
yum install nginx
chkconfig --levels 235 nginx on
service nginx start
ifconfig eth0 | grep inet | awk '{ print $2 }'
visit your ip address to check on nginx static page

now, lets install php-fpm
yum install php-fpm php-cli php-mysql php-gd php-imap php-ldap php-odbc php-pear php-xml php-xmlrpc php-magickwand php-magpierss php-mbstring php-mcrypt php-mssql php-shout php-snmp php-soap php-tidy php-pecl-apc sendmail sendmail-cf
edit /etc/php.ini to set cgi.fix_pathinfo=0;
nano /etc/php.ini
=> cgi.fix_pathinfo=0;
edit timezone to your location (Asia/Kuala_Lumpur)
=> date.timezone = “Asia/Kuala_Lumpur”
ln -sf /usr/share/zoneinfo/Asia/Kuala_Lumpur /etc/localtime
chkconfig --levels 235 php-fpm on
service php-fpm start
chkconfig --levels 235 sendmail on
service sendmail start

now, lets install memcached
yum install memcached php-memcached
nano /etc/sysconfig/memcached
=> OPTIONS="-l 127.0.0.1”
chkconfig --levels 235 memcached on
service memcached start

STEP 4: new user for FTP and SSH
useradd <username>
passwd <username>
cd /srv
mkdir www
cd www
mkdir <website>
mkdir <website>/html
chown -R user:usergroup <website>

STEP 5: configure nginx, php-fpm, php session to memcached
Edit nginx configuration file
nano /etc/nginx/nginx.conf
=> worker_processes    8;
=> keeplive_timeout    2;
easier, follow this format http://paste.laravel.com/15PT
here, we set some configuration for php-fpm to run on socket
Then, we edit the default virtual host configuration
nano /etc/nginx/conf.d/default.conf
follow this format http://paste.laravel.com/162K
now, add those global/… config files
cd /etc/nginx
mkdir global
cd global
nano restrictions.conf => http://paste.laravel.com/15PY
nano wordpress.conf => http://paste.laravel.com/162I
nano w3-total-cache.conf => http://paste.laravel.com/15Q6
service nginx restart

now, we edit php-fpm configuration
nano /etc/php-fpm.d/www.conf
=> listen = /tmp/php-fpm.sock
=> user = <username>
=> group = <username>
=> php_value[session.save_handler] = memcached
=> php_value[session.save_path] = “127.0.0.1:11211"
service php-fpm restart

service memcached restart

STEP 6: install FTP (vsftpd)
yum install vsftpd
nano /etc/vsftpd/vsftpd.conf
=> anonymous_enable=NO
=> chroot_local_user=YES
add => user_config_dir=/etc/vsftpd/vsftpd_user_conf
add => use_localtime=YES
save
mkdir /etc/vsftpd/vsftpd_user_conf
nano /etc/vsftpd/vsftpd_user_conf/<username>
=> dirlist_enable=YES
=> download_enable=YES
=> local_root=/srv/www/<website>
=> write_enable=YES
save
service vsftpd restart

STEP 7: install phpMyAdmin
yum install phpmyadmin
now, create new mysql user since root has been denied
mysql -u root -p
CREATE USER ‘<username>'@'localhost' IDENTIFIED BY ‘<password>’;
GRANT ALL PRIVILEGES ON * . * TO ‘<username>'@'localhost’;
FLUSH PRIVILEGES;
exit

STEP 8: Grant <username> to sudoers
usermod -a -G wheel <username>
visudo
uncomment %wheel lines
add new line below root   ALL=(ALL)  ALL
<username>    ALL=(ALL)   ALL 
ESC key
:wq

STEP 9: install WordPress files
su <username>
cd /srv/www/<website>/html
wget http://wordpress.org/latest.tar.gz
tar -xzvf latest.tar.gz
mv wordpress/* ./
rmdir wordpress
rm latest.tar.gz
visit website and install

STEP 10: install NewRelic
rpm -Uvh http://yum.newrelic.com/pub/newrelic/el5/x86_64/newrelic-repo-5-3.noarch.rpm
yum install newrelic-php5
newrelic-install install
=> 4baf55072ceb5cdc23a70d717317c3e296328acc
yum install newrelic-sysmond
nrsysmond-config --set license_key=4baf55072ceb5cdc23a70d717317c3e296328acc
nano /etc/php.d/newrelic.ini
=> newrelic.appname = "RTKY.com Server”
service php-fpm restart
service newrelic-sysmond start

STEP 11: SSH Key Pair Authentication
ssh-keygen
scp ~/.ssh/id_rsa.pub <username>@<ip-address>:
mkdir .ssh
mv id_rsa.pub .ssh/authorized_keys
chown -R <username>:<username> .ssh
chmod 700 .ssh
chmod 600 .ssh/authorized_keys

STEP 11: Disable root SSH login and change Port 22 to Port 215
nano /etc/ssh/sshd_config
=> Port 215
=> PermitRootLogin no
service sshd restart

STEP 12: Enable firewall settings
iptables -L
nano /etc/iptables.firewall.rules
fill up with this http://paste.laravel.com/164b
iptables-restore < /etc/iptables.firewall.rules
/sbin/service iptables save
you will face raw nat filter error (http://blog.btnotes.com/articles/606.html)
mv /etc/init.d/iptables /etc/init.d/iptables.bkp
nano /etc/init.d/iptables
paste this content http://paste.laravel.com/164n
SAVE
service iptables restart

STEP 13: install Fail2Ban for failed login access (bruteforce)
yum install fail2ban
configure setting: http://www.fail2ban.org/wiki/index.php/MANUAL_0_8#Configuration











