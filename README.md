# Prestashop install in Debian 9 with NGINX, PHP7.1-FPM, MariaDB and Lets Encrypt

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

#### Select following options

```
Enter current password for root (enter for none): Just press the Enter
Set root password? [Y/n]: Y
New password: Enter password
Re-enter new password: Repeat password
Remove anonymous users? [Y/n]: Y
Disallow root login remotely? [Y/n]: Y
Remove test database and access to it? [Y/n]:  Y
Reload privilege tables now? [Y/n]:  Y
Restart MariaDB server
```

### Restart Nginx

```
sudo systemctl restart mysql.service
```

## Install PHP 7.1-FPM

```
apt install apt-transport-https lsb-release ca-certificates && wget -O /etc/apt/trusted.gpg.d/php.gpg https://packages.sury.org/php/apt.gpg
sh -c 'echo "deb https://packages.sury.org/php/ $(lsb_release -sc) main" > /etc/apt/sources.list.d/php.list'
apt update && apt install php7.1-fpm php7.1-common php7.1-mbstring php7.1-xmlrpc php7.1-soap php7.1-gd php7.1-xml php7.1-intl php7.1-mysql php7.1-cli php7.1-mcrypt php7.1-ldap php7.1-zip php7.1-curl
```

#### Edit php.ini options

```
vim /etc/php/7.1/fpm/php.ini
```

#### Change FOLLOWING LINES:

```
file_uploads = On
allow_url_fopen = On
memory_limit = 256M
upload_max_file_size = 2G
max_execution_time = 540
cgi.fix_pathinfo = 0
date.timezone = Europe/Madrid
```

### CREATE PRESTASHOP DATABASE

```
mysql -u root -p
CREATE DATABASE prestashopdb;
CREATE USER 'prestashopuser'@'localhost' IDENTIFIED BY 'SUPERPASS';
GRANT ALL ON prestashopdb.* TO 'prestauser'@'localhost' IDENTIFIED BY 'SUPERPASS' WITH GRANT OPTION;
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
    listen [::]:80;   #Use this to enable IPv6
    server_name example.com www.example.com;

    root /var/www/html/prestashop;
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    index index.php index.html;

    location = /favicon.ico {
        log_not_found off;
        access_log off;
    }

    location = /robots.txt {
        auth_basic off;
        allow all;
        log_not_found off;
        access_log off;
    }

#    location ~ /. {
#        deny all;
#        access_log off;
#        log_not_found off;
#    }

    ##
    # Gzip Settings
    ##

    gzip on;
    gzip_disable "msie6";
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 1;
    gzip_buffers 16 8k;
    gzip_http_version 1.0;
    gzip_types application/json text/css application/javascript;

    rewrite ^/api/?(.*)$ /webservice/dispatcher.php?url=$1 last;
    rewrite ^/([0-9])(-[_a-zA-Z0-9-]*)?(-[0-9]+)?/.+.jpg$ /img/p/$1/$1$2$3.jpg last;
    rewrite ^/([0-9])([0-9])(-[_a-zA-Z0-9-]*)?(-[0-9]+)?/.+.jpg$ /img/p/$1/$2/$1$2$3$4.jpg last;
    rewrite ^/([0-9])([0-9])([0-9])(-[_a-zA-Z0-9-]*)?(-[0-9]+)?/.+.jpg$ /img/p/$1/$2/$3/$1$2$3$4$5.jpg last;
    rewrite ^/([0-9])([0-9])([0-9])([0-9])(-[_a-zA-Z0-9-]*)?(-[0-9]+)?/.+.jpg$ /img/p/$1/$2/$3/$4/$1$2$3$4$5$6.jpg last;
    rewrite ^/([0-9])([0-9])([0-9])([0-9])([0-9])(-[_a-zA-Z0-9-]*)?(-[0-9]+)?/.+.jpg$ /img/p/$1/$2/$3/$4/$5/$1$2$3$4$5$6$7.jpg last;
    rewrite ^/([0-9])([0-9])([0-9])([0-9])([0-9])([0-9])(-[_a-zA-Z0-9-]*)?(-[0-9]+)?/.+.jpg$ /img/p/$1/$2/$3/$4/$5/$6/$1$2$3$4$5$6$7$8.jpg last;
    rewrite ^/([0-9])([0-9])([0-9])([0-9])([0-9])([0-9])([0-9])(-[_a-zA-Z0-9-]*)?(-[0-9]+)?/.+.jpg$ /img/p/$1/$2/$3/$4/$5/$6/$7/$1$2$3$4$5$6$7$8$9.jpg last;
    rewrite ^/([0-9])([0-9])([0-9])([0-9])([0-9])([0-9])([0-9])([0-9])(-[_a-zA-Z0-9-]*)?(-[0-9]+)?/.+.jpg$ /img/p/$1/$2/$3/$4/$5/$6/$7/$8/$1$2$3$4$5$6$7$8$9$10.jpg last;
    rewrite ^/c/([0-9]+)(-[.*_a-zA-Z0-9-]*)(-[0-9]+)?/.+.jpg$ /img/c/$1$2$3.jpg last;
    rewrite ^/c/([a-zA-Z_-]+)(-[0-9]+)?/.+.jpg$ /img/c/$1$2.jpg last;
    rewrite ^/images_ie/?([^/]+).(jpe?g|png|gif)$ /js/jquery/plugins/fancybox/images/$1.$2 last;
    rewrite ^/order$ /index.php?controller=order last;
    location /admin-area/ {                           #Change this to your admin folder
        if (!-e $request_filename) {
            rewrite ^/.*$ /admin-area/index.php last; #Change this to your admin folder
        }
    }
    location / {
        if (!-e $request_filename) {
            rewrite ^/.*$ /index.php last;
        }
    }

    location ~ .php$ {
        fastcgi_split_path_info ^(.+.php)(/.*)$;
        try_files $uri =404;
        fastcgi_keep_conn on;
        include /etc/nginx/fastcgi_params;
        fastcgi_pass unix:/var/run/php/php7.1-fpm.sock;  #Change this to your PHP-FPM location
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
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
certbot --authenticator standalone --installer nginx --pre-hook "service nginx stop" --post-hook "service nginx start" -d example.com -d www.example.com
certbot --pre-hook "service nginx stop" --post-hook "service nginx start" --nginx -m mail@mail.com -d example.com -d www.example.com
```

### Check that prestashop file has been correctly edited under end of 80 original conf

```

## This info is autocreated by certbot when generating the first cert

    listen 443 ssl; # managed by Certbot
ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem; # managed by Certbot
ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem; # managed by Certbot
ssl_session_cache shared:le_nginx_SSL:1m; # managed by Certbot
ssl_session_timeout 1440m; # managed by Certbot

ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # managed by Certbot
ssl_prefer_server_ciphers on; # managed by Certbot

ssl_ciphers "ECDHE-ECDSA-AES128-GCM-SHA256 ECDHE-ECDSA-AES256-GCM-SHA384 ECDHE-ECDSA-AES128-SHA ECDHE-ECDSA-AES256-SHA ECDHE-ECDSA-AES128-SHA256 ECDHE-ECDSA-AES256-SHA384 ECDHE-RSA-AES128-GCM-SHA256 ECDHE-RSA-AES256-GCM-SHA384 ECDHE-RSA-AES128-SHA ECDHE-RSA-AES128-SHA256 ECDHE-RSA-AES256-SHA384 DHE-RSA-AES128-GCM-SHA256 DHE-RSA-AES256-GCM-SHA384 DHE-RSA-AES128-SHA DHE-RSA-AES256-SHA DHE-RSA-AES128-SHA256 DHE-RSA-AES256-SHA256 EDH-RSA-DES-CBC3-SHA"; # managed by Certbot

    # Redirect non-https traffic to https
    #if ($scheme != "https") {
    #     return 301 https://$host$request_uri;
    #} # managed by Certbot

## End of config autogenerated by certbot

```

## Restart Nginx

```
systemctl restart nginx.service
```

Access your site!
