---
layout: post
title:  "Phishing Frenzy Ubuntu 18.04"
date:   2019-05-28 23:59:59 +0800
categories: phishing
tags: [phishing, Ubuntu 18.04]
---

更新倉庫

    sudo apt-get update

安裝以下

    sudo apt-get install apache2 php mysql-server libmysqlclient-dev git curl

設定 mysql root 密碼

先查看檔案 /etc/mysql/debian.cnf

    sudo cat /etc/mysql/debian.cnf

內容大概是這樣

    # Automatically generated for Debian scripts. DO NOT TOUCH!
    [client]
    host     = localhost
    user     = debian-sys-maint
    password = zJjv2pLf2kQ1OESD
    socket   = /vat/run/mysqld/mysqld.sock
    [mysql_upgrade]
    host     = localhost
    user     = debian-sys-maint
    password = zJjv2pLf2kQ1OESD
    socket   = /vat/run/mysqld/mysqld.sock

使用檔案中的使用者 debian-sys-maint 與密碼 zJjv2pLf2kQ1OESD 登入 mysql

    mysql -u debian-sys-maint -p

下一行會出現 Enter password: 在這裡輸入密碼，可以用複製貼上

    mysql> use mysql;
    mysql> UPDATE user SET plugin=’mysql_native_password’ WHERE User=’root’;
    mysql> UPDATE mysql.user SET authentication_string=PASSWORD(‘password’) WHERE USER=’root’;
    mysql> FLUSH PRIVILEGES;
    mysql> exit;

這樣就將 mysql 的 root 密碼設為 password

下載 Phishing Frenzy

sudo git clone https://github.com/pentestgeek/phishing-frenzy.git /var/www/phishing-frenzy

安裝 rvm

    \curl -sSL https://get.rvm.io | bash

將 rvm 啟用加到 .bashrc

    echo "source ~/.rvm/scripts/rvm" >> .bashrc

以 rvm 安裝 ruby 2.3.0

    rvm install 2.3.0

過程中可能會需要輸入密碼

安裝 Rails

    rvm all do gem install --no-document rails

安裝 Passenger

    rvm all do gem install --no-document passenger

安裝 passenger apache2 module

    passenger-install-apache2-module

過程中會選擇語言，選 ruby，接著會提示系統中缺少那些東西，依照提示安裝即可

    sudo spt-get install libcurl4-openssl-dev
    sudo spt-get install apache2-dev
    sudo spt-get install libapr1-dev
    sudo spt-get install libaprutil1-dev
    sudo spt-get install libssl-dev

再次安裝 passenger apache2 module

    passenger-install-apache2-module

這會從原始碼編譯需要一些時間

安裝好之後會出現訊息

    Almost there!

    Please eidt your Apache configuration file, and add these lines:

       LoadModule passenger_module /home/sean/.rvm/gems/ruby-2.3.0/gems/passenger-6.0.2/buildout/apache2/mod_passenger.so
       <IfModule mod_passenger.c>
         PassengerRoot /home/sean/.rvm/gems/ruby-2.3.0/gems/passenger-6.0.2
         PassengerDefaultRuby /home/sean/.rvm/gems/ruby-2.3.0/wrappers/ruby
       </IFModule>

    After you restart Apache, you are ready to deploy any number of web applications on Apache, with a minimum amount of configuration!

按下 Enter 後會再出現這個訊息

    You did not specify 'LoadModule passenger_module' in any of your Apache configuration files. Please paste the configuration snippet that this installer printed earlier, into one of your Apache configuration files, such as /etc/apache2/apache2.conf

在 /etc/apache2/apache2.conf 檔案中加入上述的設定

    sudo nano /etc/apache2/apache2.conf

新增 Apache 設定檔

    sudo nano /etc/apache2/sites-enabled/pf.conf

內容如下

    <VirtualHost *:80>
      ServerName 127.0.0.1
      # !!! Be sure to point DocumentRoot to 'public'!
      DocumentRoot /var/www/phishing-frenzy/public
      RailsEnv development
      <Directory /var/www/phishing-frenzy/public>
        # This relaxes Apache security settings.
        AllowOverride all
        # MultiViews must be turned off.
        Options -MultiViews
      </Directory>
    </VirtualHost>

設定 MySQL Database

首先確定啟動 MySQL

    sudo service mysql start

登入

    mysql -u root -p

建立 database pf_dev 與使用者 pf_dev@localhost

    mysql> create database pf_dev;
    mysql> grant all privileges on pf_dev.* to 'pf_dev'@'localhost' identified by 'password';

安裝 Redis

    wget http://download.redis.io/releases/redis-stable.tar.gz
    tar xzf redis-stable.tar.gz
    cd redis-stable/
    sudo make
    sudo make install
    cd utils/
    sudo ./install_server.sh

安裝 Required Gems

    cd /var/www/phishing-frenzy/
    bundle install
    rvmsudo bundle exec rake db:migrate
    rvmsudo bundle exec rake db:seed

Sidekiq Configuration

    mkdir -p /var/www/phishing-frenzy/tmp/pids

啟動 Sidekiq

    rvmsudo sidekiq -C config/sidekiq.yml

這裡可以用 Ctrl+Z 之後搭配指令 `bg %1` 讓 Sidekiq 在背景運作。

設定 sudo 權限給 www-data

    sudo visudo

在最下面加入

    www-data ALL=(ALL) NOPASSWD: /etc/init.d/apache2 reload

Load the Efax and Intel default templates for PF using the rake helper

    rvmsudo bundle exec rake templates:load

擁有者設定

    sudo chown -R www-data:www-data /var/www/phishing-frenzy/
    sudo chmod -R 755 /var/www/phishing-frenzy/public/uploads/
    sudo chown -R www-data:www-data /etc/apache2/sites-enabled/
    sudo chmod 755 /etc/apache2/sites-enabled/
    sudo apachectl restart

打開瀏覽器連線到 http://127.0.0.1/ 測試是否成功