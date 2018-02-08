BOX_OS = "centos/7"
BOX_VERSION = "1710.01"

$cmdPhpSetup = <<SCRIPT
yum -y update

systemctl stop firewalld
systemctl disable firewalld
yum -y remove firewalld

cat <<EOF > /etc/selinux/config
SELINUX=disabled
SELINUXTYPE=targeted
EOF

yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
yum install http://rpms.remirepo.net/enterprise/remi-release-7.rpm

yes | yum install -y curl unzip gcc epel-release nginx yum-utils git

yum-config-manager --enable remi-php71

yes | yum install php php-mcrypt php-cli php-gd php-curl \
  php-mysql php-ldap php-zip php-fileinfo php-fpm php-pdo \
  php-pdo_mysql php-dom php-devel php-pear php-mbstring

systemctl enable nginx.service
systemctl start nginx.service

cat <<EOF > /etc/yum.repos.d/mongodb-org.repo
[mongodb-org-3.4]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/3.4/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-3.4.asc
EOF

yes | yum install -y mongodb-org

systemctl enable mongod.service
systemctl start mongod.service

cat <<EOF > /etc/nginx/conf.d/laravel.conf
server {
listen  80;
server_name localhost 127.0.0.1 10.240.0.201;
access_log /var/www/laravel/logs/access.log;
error_log /var/www/laravel/logs/error.log;

root /var/www/laravel;
index index.php;

location / {
    try_files $uri $uri/ /index.php?$args;
}

location ~ \.php$ {
    try_files $uri =404;
    include /etc/nginx/fastcgi_params;
    fastcgi_pass unix:/var/run/php-fpm/php-fpm.sock;
    fastcgi_index index.php;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
EOF

cat <<EOF > /etc/nginx/conf.d/xhgui.conf
server {
listen  81;
server_name localhost 127.0.0.1 10.240.0.201;
access_log /var/www/xhgui/logs/access.log;
error_log /var/www/xhgui/logs/error.log;

root /var/www/xhgui/webroot;
index index.php;

location / {
    try_files $uri $uri/ /index.php?$args;
}

location ~ \.php$ {
    try_files $uri =404;
    include /etc/nginx/fastcgi_params;
    fastcgi_pass unix:/var/run/php-fpm/php-fpm.sock;
    fastcgi_index index.php;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
EOF

cat <<EOF > /etc/nginx/conf.d/demosite.conf
server {
listen  82;
server_name localhost 127.0.0.1 10.240.0.201;
access_log /var/www/demo/logs/access.log;
error_log /var/www/demo/logs/error.log;

root /var/www/demo/public;
index index.php;

location / {
    try_files $uri $uri/ /index.php?$args;
}

location ~ \.php$ {
    try_files $uri =404;
    include /etc/nginx/fastcgi_params;
    fastcgi_pass unix:/var/run/php-fpm/php-fpm.sock;
    fastcgi_index index.php;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
EOF

mkdir -p /var/www/xhgui/logs
mkdir -p /var/www/demo/logs
mkdir -p /var/www/demo/laravel

pushd /tmp
curl -k -sS https://getcomposer.org/installer | php
chmod +x composer.phar
mv composer.phar /usr/bin/composer
popd

pushd /var/www/laravel
curl -Ls https://github.com/laravel/laravel/archive/v4.2.11.zip -o v4.2.11.zip
unzip v4.2.11.zip
yes | mv laravel-4.2.11/* .
yes | mv laravel-4.2.11/.* .
rmdir laravel-4.2.11/
composer install
popd

pushd /var/www/demo
git clone https://github.com/symfony/demo .
composer install
popd

pushd /tmp
mkdir xhprof
cd xdprof
git clone https://github.com/tideways/php-xhprof-extension .
phpize
./configure
make
make install
sed -ie "\$aextension=tideways_xhprof.so" /etc/php.ini
sed -rie "s|(auto_prepend_file =)[ ]*$|\1 /var/www/xhgui/external/header.php|" /etc/php.ini
popd

pushd /var/www/xhgui
git clone https://github.com/perftools/xhgui .
chmod -R 0777 cache
sed -rie "s|(.*)(mongodb://)(127.0.0.1:27017)(.*)|\1\3\4|" ./config/config.default.php
sed -rie "s|(.*)(return rand(1, 100) === 42;)(.*)|\1return true;\3|" ./config/config.default.php
mongo --eval "db = db.getSiblingDB('xhprof'); \
  db.results.ensureIndex({'meta.SERVER.REQUEST_TIME': -1}); \
  db.results.ensureIndex({'profile.main().wt': -1}); \
  db.results.ensureIndex({'profile.main().mu': -1}); \
  db.results.ensureIndex({'profile.main().cpu': -1}); \
  db.results.ensureIndex({'meta.url': 1}); \
  db.results.ensureIndex({'meta.simple_url': 1});"
php install.php
popd

chown -R nginx:nginx /var/www
systemctl restart nginx.service
systemctl restart php-fpm.service

reboot
SCRIPT

Vagrant.configure("2") do |config|
  config.vm.define "php" do |box|
    box.vm.box = BOX_OS
    box.vm.box_version = BOX_VERSION
    box.vm.provider "virtualbox" do |v|
      v.memory = 1000
      v.cpus = 1
    end
    box.vm.hostname = "phphost"
    box.vm.network "public_network", bridge: "Microsoft KM-TEST Loopback Adapter", ip: "10.240.0.201"
    # box.vm.network "private_network", ip: "192.168.56.11"
    # box.vm.network "forwarded_port", guest: 8080, host: 8088
    box.vm.synced_folder ".", "/vagrant", disabled: false, type: "rsync", rsync__auto: true
    box.vm.provision "shell", inline: $cmdPhpSetup
  end
end
