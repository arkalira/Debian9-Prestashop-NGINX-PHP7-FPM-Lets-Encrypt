# Prestashop install in Debian 9 with NGINX, PHP7.2-FPM, MariaDB and Lets Encrypt

## Update and install software
```
apt update && apt install -y vim mc uml-utilities ntp qemu-guest-agent \
htop sudo curl git-core etckeeper molly-guard apt-transport-https ca-certificates \
bridge-utils gettext-base jq -y < /dev/null
```
### Install Oh-my-zsh

```
apt-get install zsh -y < /dev/null
usermod root -s /bin/zsh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
sed -ri 's/ZSH_THEME="robbyrussell"/ZSH_THEME="agnoster"/g' .zshrc
sed -ri 's/plugins=\(git\)/plugins=\(debian apt systemd docker zsh-navigation-tools\)/g' .zshrc
echo 'export VTYSH_PAGER=more' >> /etc/zsh/zshenv
source .zshrc
```

### Add backports repo and install kernel

```
echo "deb http://ftp.debian.org/debian jessie-backports main"  >> /etc/apt/sources.list.d/debian-backports.list
apt update
apt install linux-image-amd64 linux-headers- -t jessie-backports -y < /dev/null
```

### Uninstall old kernel

```
apt remove --purge $(dpkg --list | grep linux-image-3 | cut -d " " -f 3) -y
shutdown -r now
```

### System tunning

```
cat > /etc/sysctl.d/local.conf << EOL
fs.file-max = 2097152
fs.nr_open = 2097152
net.ipv4.ip_nonlocal_bind = 1
EOL

cat > /etc/security/limits.d/local.conf <<EOL
*         hard    nofile      999999
root      hard    nofile      999999
*         soft    nofile      999999
root      soft    nofile      999999
EOL

```

## Install Sysdig
### Sysdig for monitoring

```
curl -s https://s3.amazonaws.com/download.draios.com/DRAIOS-GPG-KEY.public | apt-key add -
curl -s -o /etc/apt/sources.list.d/draios.list http://download.draios.com/stable/deb/draios.list
apt-get -qq update < /dev/null
apt-get -qq -y install linux-headers-$(uname -r) < /dev/null || kernel_warning
apt-get -qq -y install sysdig < /dev/null
echo "sysdig-probe" >> /etc/modules-load.d/modules.conf
modprobe sysdig-probe
```

## Install Prestashop
### INSTALL NGINX

```
apt install nginx -y
systemctl stop nginx.service && systemctl start nginx.service && systemctl enable nginx.service
```

### INSTALL MARIADB

```
apt-get install mariadb-server mariadb-client -y
systemctl stop mysql.service && systemctl start mysql.service && systemctl enable mysql.service
mysql_secure_installation
```

---
Enter current password for root (enter for none): Just press the Enter
Set root password? [Y/n]: Y
New password: Enter password
Re-enter new password: Repeat password
Remove anonymous users? [Y/n]: Y
Disallow root login remotely? [Y/n]: Y
Remove test database and access to it? [Y/n]:  Y
Reload privilege tables now? [Y/n]:  Y
Restart MariaDB server
---

```
sudo systemctl restart mysql.service
```

### Install PHP7.1-FPM

```
apt install apt-transport-https lsb-release ca-certificates && wget -O /etc/apt/trusted.gpg.d/php.gpg https://packages.sury.org/php/apt.gpg
sh -c 'echo "deb https://packages.sury.org/php/ $(lsb_release -sc) main" > /etc/apt/sources.list.d/php.list'
apt update && apt install php7.2-fpm php7.2-common php7.2-mbstring php7.2-xmlrpc php7.2-soap php7.2-gd php7.2-xml php7.2-intl php7.2-mysql php7.2-cli php7.1-mcrypt php7.2-ldap php7.2-zip php7.2-curl
```
---

vim /etc/php/7.2/fpm/php.ini

--- Change FOLLOWING LINES:

file_uploads = On
allow_url_fopen = On
memory_limit = 256M
upload_max_file_size = 2G
max_execution_time = 540
cgi.fix_pathinfo = 0
date.timezone = Europe/Madrid

----

### CREATE PRESTASHOP DATABASE

```
mysql -u root -p
CREATE DATABASE prestadb;
CREATE USER 'prestauser'@'localhost' IDENTIFIED BY 'prestadb_n4r4n';
GRANT ALL ON prestadb.* TO 'prestauser'@'localhost' IDENTIFIED BY 'prestadb_n4r4n' WITH GRANT OPTION;
FLUSH PRIVILEGES;
EXIT;
```

### DOWNLOAD PRESTASHOP LATEST RELEASE

```
cd /tmp && curl -O https://download.prestashop.com/download/releases/prestashop_1.7.2.4.zip
unzip prestashop_1.7.2.4.zip
mkdir -p /var/www/html/prestashop
unzip prestashop.zip -d /var/www/html/prestashop/
chown -R www-data:www-data /var/www/html/prestashop/
chmod -R 755 /var/www/html/prestashop/
```

### Configure NGINX P80

```
vim /etc/nginx/sites-available/prestashop

server {
    listen 80;
    listen [::]:80;
    root /var/www/html/prestashop;
    index  index.php index.html index.htm;
    server_name  example.com www.example.com;

    location / {
    rewrite ^/api/?(.*)$ /webservice/dispatcher.php?url=$1 last;
    rewrite ^/([0-9])(-[_a-zA-Z0-9-]*)?(-[0-9]+)?/.+\.jpg$ /img/p/$1/$1$2.jpg last;
    rewrite ^/([0-9])([0-9])(-[_a-zA-Z0-9-]*)?(-[0-9]+)?/.+\.jpg$ /img/p/$1/$2/$1$2$3.jpg last;
    rewrite ^/([0-9])([0-9])([0-9])(-[_a-zA-Z0-9-]*)?(-[0-9]+)?/.+\.jpg$ /img/p/$1/$2/$3/$1$2$3$4.jpg last;
    rewrite ^/([0-9])([0-9])([0-9])([0-9])(-[_a-zA-Z0-9-]*)?(-[0-9]+)?/.+\.jpg$ /img/p/$1/$2/$3/$4/$1$2$3$4$5.jpg last;
    rewrite ^/([0-9])([0-9])([0-9])([0-9])([0-9])(-[_a-zA-Z0-9-]*)?(-[0-9]+)?/.+\.jpg$ /img/p/$1/$2/$3/$4/$5/$1$2$3$4$5$6.jpg last;
    rewrite ^/([0-9])([0-9])([0-9])([0-9])([0-9])([0-9])(-[_a-zA-Z0-9-]*)?(-[0-9]+)?/.+\.jpg$ /img/p/$1/$2/$3/$4/$5/$6/$1$2$3$4$5$6$7.jpg last;
    rewrite ^/([0-9])([0-9])([0-9])([0-9])([0-9])([0-9])([0-9])(-[_a-zA-Z0-9-]*)?(-[0-9]+)?/.+\.jpg$ /img/p/$1/$2/$3/$4/$5/$6/$7/$1$2$3$4$5$6$7$8.jpg last;
    rewrite ^/([0-9])([0-9])([0-9])([0-9])([0-9])([0-9])([0-9])([0-9])(-[_a-zA-Z0-9-]*)?(-[0-9]+)?/.+\.jpg$ /img/p/$1/$2/$3/$4/$5/$6/$7/$8/$1$2$3$4$5$6$7$8$9.jpg last;
    rewrite ^/c/([0-9]+)(-[_a-zA-Z0-9-]*)(-[0-9]+)?/.+\.jpg$ /img/c/$1$2.jpg last;
    rewrite ^/c/([a-zA-Z-]+)(-[0-9]+)?/.+\.jpg$ /img/c/$1.jpg last;
    rewrite ^/([0-9]+)(-[_a-zA-Z0-9-]*)(-[0-9]+)?/.+\.jpg$ /img/c/$1$2.jpg last;
    try_files $uri $uri/ /index.php?$args;        
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php7.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

}
```
### Enable Nginx Site

```
ln -s /etc/nginx/sites-available/prestashop /etc/nginx/sites-enabled/
```

### Lets Encrypt Nginx

```
apt-get update
apt-get install python-certbot-nginx
```

### Cert for new sites

```
certbot --authenticator webroot --installer nginx
certbot --authenticator standalone --installer nginx --pre-hook "service nginx stop" --post-hook "service nginx start" -d hipnoticmediaclub.com -d www.hipnoticmediaclub.com
certbot --pre-hook "service nginx stop" --post-hook "service nginx start" --nginx -m arkalira@gmail.com -d hipnoticmediaclub.com -d www.hipnoticmediaclub.com
```

### Check that prestashop file has been correctly edited under end of 80 original conf

```
listen 443 ssl; # managed by Certbot
ssl_certificate /etc/letsencrypt/live/hipnoticmediaclub.com/fullchain.pem; # managed by Certbot
ssl_certificate_key /etc/letsencrypt/live/hipnoticmediaclub.com/privkey.pem; # managed by Certbot
ssl_session_cache shared:le_nginx_SSL:1m; # managed by Certbot
ssl_session_timeout 1440m; # managed by Certbot

ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # managed by Certbot
ssl_prefer_server_ciphers on; # managed by Certbot

ssl_ciphers "ECDHE-ECDSA-AES128-GCM-SHA256 ECDHE-ECDSA-AES256-GCM-SHA384 ECDHE-ECDSA-AES128-SHA ECDHE-ECDSA-AES256-SHA ECDHE-ECDSA-AES128-SHA256 ECDHE-ECDSA-AES256-SHA384 ECDHE-RSA-AES128-GCM-SHA256 ECDHE-RSA-AES256-GCM-SHA384 ECDHE-RSA-AES128-SHA ECDHE-RSA-AES128-SHA256 ECDHE-RSA-AES256-SHA384 DHE-RSA-AES128-GCM-SHA256 DHE-RSA-AES256-GCM-SHA384 DHE-RSA-AES128-SHA DHE-RSA-AES256-SHA DHE-RSA-AES128-SHA256 DHE-RSA-AES256-SHA256 EDH-RSA-DES-CBC3-SHA"; # managed by Certbot

# Redirect non-https traffic to https
#if ($scheme != "https") {
#     return 301 https://$host$request_uri;
#} # managed by Certbot
```

## Restart Nginx

```
systemctl restart nginx.service
```

Access your site!
