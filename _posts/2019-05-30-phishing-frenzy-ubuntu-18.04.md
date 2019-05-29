---
layout: post
title:  "Phishing Frenzy Ubuntu 18.04"
date:   2019-05-30 23:59:59 +0800
categories: phishing
tags: [phishing, Ubuntu 18.04]
---

這個安裝過程大部分是參考 Phishing Frenzy 網站上的 [Installing Phishing Frenzy on Ubuntu Linux](https://www.phishingfrenzy.com/resources/install_ubuntu_linux)
，不過網站上的教學是 **Ubuntu Server 16.04.2 LTS x64 bit edition**，我這裡用的是 **Ubuntu Desktop 18.04.2 LTS amd64**

這裡的安裝順序與 Phishing Frenzy 網站上的教學是一樣的，不過有些步驟會有版本上的差異，如18.04 與 16.04 的倉庫內的 MySQL 版本不同等。

## 安裝套件

首先更新倉庫

    sudo apt-get update

安裝以下

    sudo apt-get install apache2 php mysql-server libmysqlclient-dev git curl

## 設定 MySQL root 密碼

在 Ubuntu 16.04 安裝 mysql 的過程中會有提示要使用者設定 root 密碼，在 18.04 版中沒有這個過程，需要自己設定 mysql root 密碼

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
    mysql> UPDATE user SET plugin='mysql_native_password' WHERE User='root';
    mysql> UPDATE mysql.user SET authentication_string=PASSWORD('password') WHERE USER='root';
    mysql> FLUSH PRIVILEGES;
    mysql> exit;

這樣就將 mysql 的 root 密碼設為 password

## 下載 Phishing Frenzy

    sudo git clone https://github.com/pentestgeek/phishing-frenzy.git /var/www/phishing-frenzy

## 安裝 RVM 與 Ruby

    \curl -sSL https://get.rvm.io | bash

將 rvm 啟用加到 .bashrc

    echo "source ~/.rvm/scripts/rvm" >> .bashrc

以 rvm 安裝 ruby 2.3.0

    rvm install 2.3.0

過程中可能會需要輸入密碼

安裝 Rails Gem

    rvm all do gem install --no-document rails

安裝 Passenger Gem

    rvm all do gem install --no-document passenger

## 安裝 Passenger

執行 passenger 安裝腳本

    passenger-install-apache2-module

過程中會選擇語言，選 ruby，接著會提示系統中缺少那些東西，依照提示安裝即可

    sudo apt-get install libcurl4-openssl-dev
    sudo apt-get install apache2-dev
    sudo apt-get install libapr1-dev
    sudo apt-get install libaprutil1-dev
    sudo apt-get install libssl-dev

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

    You did not specify 'LoadModule passenger_module' in any of your Apache configuration files.
    Please paste the configuration snippet that this installer printed earlier,
    into one of your Apache configuration files, such as /etc/apache2/apache2.conf

在 /etc/apache2/apache2.conf 檔案中加入上述的設定 (LoadModule passenger_module ... 那段)

    sudo nano /etc/apache2/apache2.conf

其中路徑的部分是自己的路徑，我的使用者是 sean ，所以路徑都是 `/home/sean/...`

## Apache VHOST 設定

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

其中 `ServerName 127.0.0.1` 這個會影響 Apache 如何判斷進來的連線是要連到哪個 VHOST，設定成 127.0.0.1 的時候就不能從外部訪問這個 VHOST，設定成實體 IP 或是域名可以使這個 VHOST 對外服務。

## 設定 MySQL Database

首先確定啟動 MySQL

    sudo service mysql start

登入

    mysql -u root -p

建立 database pf_dev 與使用者 pf_dev@localhost

    mysql> create database pf_dev;
    mysql> grant all privileges on pf_dev.* to 'pf_dev'@'localhost' identified by 'password';

## 安裝 Redis

    wget http://download.redis.io/releases/redis-stable.tar.gz
    tar xzf redis-stable.tar.gz
    cd redis-stable/
    sudo make
    sudo make install
    cd utils/
    sudo ./install_server.sh

## 安裝 Phishing Frenzy 需要的 Gems

    cd /var/www/phishing-frenzy/
    bundle install
    rvmsudo bundle exec rake db:migrate
    rvmsudo bundle exec rake db:seed

## 設定 Sidekiq

建一個資料夾給 Sidekiq pid

    mkdir -p /var/www/phishing-frenzy/tmp/pids

啟動 Sidekiq

    rvmsudo bundle exec sidekiq -C config/sidekiq.yml

這裡可以用 Ctrl+Z 之後搭配指令 `bg %1` 讓 Sidekiq 在背景運作。

如果是用桌面版的 Ubuntu 也可以直接開另外一個終端機繼續下面的步驟。

## 系統設定

設定 sudo 權限給 www-data

    sudo visudo

在最下面加入

    www-data ALL=(ALL) NOPASSWD: /etc/init.d/apache2 reload

用 rake helper 幫 Phishing Frenzy 載入預設的 Efax 模板與 Intel模板

    rvmsudo bundle exec rake templates:load

## 所有權設定

    sudo chown -R www-data:www-data /var/www/phishing-frenzy/
    sudo chmod -R 755 /var/www/phishing-frenzy/public/uploads/
    sudo chown -R www-data:www-data /etc/apache2/sites-enabled/
    sudo chmod 755 /etc/apache2/sites-enabled/

## 重啟 Apache Server

    sudo apachectl restart

## 測試

打開瀏覽器連線到 http://127.0.0.1/ 測試是否成功

## 預設帳號密碼

    username: admin
    password: Funt1me!
