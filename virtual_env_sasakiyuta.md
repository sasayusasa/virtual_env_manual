# 環境構築手順書
作成日 ： 2021年4月27日  
作成者 ： 佐々木 祐太

## 〜環境〜
| Name | Version |
| :- | -: |
| OS | CentOS7 |
| Nginx | 1.19.10 |
| MySQL | 5.7 |
| PHP | 7.3 |
| Laravel | 6.0 |

## 〜環境構築〜
### 1. Vagrant用作業ディレクトリの作成
#### 作業用ディレクトリを作成し、そのディレクトリ内で使うBOXを宣言。
```
mkdir ディレクトリ名
cd ディレクトリ名
vagrant init centos/7
```
### 2. Vagrantfileの編集
#### Vagrantfile内のポート、IP、同期ファイルに関する記述を編集。
```
config.vm.network "forwarded_port", guest: 80, host: 8080 ← #を外す。
config.vm.network "private_network", ip: "192.168.33.19" ← #を外す。

変更前 config.vm.synced_folder "../data", "/vagrant_data"
変更後 config.vm.synced_folder "./", "/vagrant", type:"virtualbox"
```
### 3. Vagrantプラグインのインストール/ゲストOSの起動/ログイン
#### 正常に動作するようvagrant-vbguestプラグインをインストールする。  
vagrant-vbguestプラグインはGuest AdditionsのバージョンをVirtualBoxのバージョンに合わせて最新化してくれる。
##### インストール
```
vagrant plugin install vagrant-vbguest
```
##### バージョン確認
```
vagrant plugin list

上記コマンドで現在インストールされているプラグインの確認ができる。
実行して表示された中に『vagrant-vbguest』があればインストールが成功している。
```
##### 起動とログイン
```
vagrant up
vagrant ssh
```
### 4. 必要なパッケージ/PHP/Composerのインストール
#### パッケージインストール  
下記のコマンドはGit等の開発に必要なパッケージを一括でインストールしてくれる。
```
sudo yum -y groupinstall "development tools"
```
#### PHPインストール  
yumコマンドでのインストールは古いバージョンのPHPがインストールされてしまうので、  
外部パッケージツールをダウンロードして、そこからPHPをインストールする。  
※Laravel6.0を動作させるため、PHPのバージョンは7.2以上が必要。  
(今回使用するバージョンは7.3)
```
sudo yum -y install epel-release wget
sudo wget http://rpms.famillecollet.com/enterprise/remi-release-7.rpm
sudo rpm -Uvh remi-release-7.rpm
sudo yum -y install --enablerepo=epel,remi,remi-php73 php php-devel php-mbstring php-pdo php-gd php-xml php-mcrypt
```
##### PHPバージョン確認
```
php -v

上記実行し、下記のような表示になればOK。

PHP 7.3.28 (cli) (built: Apr 27 2021 13:57:06) ( NTS )
Copyright (c) 1997-2018 The PHP Group
Zend Engine v3.3.28, Copyright (c) 1998-2018 Zend Technologies
```
#### Composerインストール  
PHPのパッケージ管理ツールであるComposerをインストール。
```
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php composer-setup.php
php -r "unlink('composer-setup.php');"
sudo mv composer.phar /usr/local/bin/composer
```
##### Composerバージョン確認
```
composer -v

上記実行し、下記のような表示になればOK。
   ______
  / ____/___  ____ ___  ____  ____  ________  _____
 / /   / __ \/ __ `__ \/ __ \/ __ \/ ___/ _ \/ ___/
/ /___/ /_/ / / / / / / /_/ / /_/ (__  )  __/ /
\____/\____/_/ /_/ /_/ .___/\____/____/\___/_/
                    /_/
Composer version 2.0.12 2021-04-01 10:14:59
```
### 5. Laravelのプロジェクト作成とログイン機能追加
#### 今回使用するLaravel6.0をインストールする。
```
cd /vagrant
composer create-project --prefer-dist laravel/laravel プロジェクト名 "6.*"
```
##### Laravelバージョン確認
```
cd /vagrant/Laravelプロジェクト名

上記実行し、Laravelプロジェクトディレクトリへ移動。

php artisan -V

更に上記実行し、下記のような表示で、バージョンが6.0以上であればOK。

Laravel Framework 6.20.25
```
#### インストールしたLaravel6.0に認証機能を実装する。
```
cd /vagrant/Laravelプロジェクト名
composer require laravel/ui 1.*
php artisan ui vue --auth
```
### 6. Nginxインストール
#### 最新版インストールの準備のためファイルの作成と書き込み。
```
vagrant ssh
sudo vi /etc/yum.repos.d/nginx.repo

〜ファイルに下記内容書き込み〜

[nginx]
name=nginx repo
baseurl=https://nginx.org/packages/mainline/centos/\$releasever/\$basearch/
gpgcheck=0
enabled=1
```
#### インストールし、バージョン確認後、起動。
##### インストール
```
sudo yum install -y nginx
```
##### Nginxバージョン確認
```
nginx -v

上記実行し、下記のような表示になればOK。

nginx version: nginx/1.19.10
```
##### 起動
```
sudo systemctl start nginx
```
### 7. DBのインストール/起動/接続
MySQLのバージョン5.7をインストールするため、  
rpmに新たにリポジトリを追加し、インストールを行う。  
インストール後、バージョン確認し、起動。
##### インストール
```
sudo wget https://dev.mysql.com/get/mysql57-community-release-el7-7.noarch.rpm
sudo rpm -Uvh mysql57-community-release-el7-7.noarch.rpm
sudo yum install -y mysql-community-server
```
##### MySQLバージョン確認
```
mysql --version

上記実行し、下記のような表示になればOK。

mysql  Ver 14.14 Distrib 5.7.34, for Linux (x86_64) using  EditLine wrapper
```
##### 起動
```
sudo systemctl start mysqld
```
#### 簡易PW設定用に編集を施す。
```
sudo vi /etc/my.cnf

〜変更箇所〜

datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
validate-password=OFF ← 追記。
```
#### 編集後、設定反映のため再起動。
```
sudo systemctl restart mysqld
```
#### 初期PWを確認後ログインし、PWを再設定後、データベースの作成。
```
sudo cat /var/log/mysqld.log | grep 'temporary password' ← 左記の実行結果 root@localhost: ~この箇所~ をコピー。
mysql -u root -p ← PWにコピーしたものを貼り付け。
set password = "新たなpassword";
create database プロジェクトのDB名;
```
### 8. Laravelプロジェクトディレクトリ下の.envファイル編集/マイグレーション実行
#### Laravelを動かせるようにするため、.envファイルを編集。
```
DB_DATABASE=プロジェクトのDB名
DB_PASSWORD=MySQLのPW
```
#### Laravelプロジェクトディレクトリ下にてマイグレーション実行。
```
php artisan migrate
```
### 9. Laravelを動かすための準備
#### まずはNginxファイルの編集。
```
sudo vi /etc/nginx/conf.d/default.conf

〜変更箇所〜

server {
    listen       80;
    server_name  192.168.33.19; ← 変更。
    root /vagrant/Laravelプロジェクト名/public; ← 追記。
    index  index.html index.htm index.php; ← 追記。

    #charset koi8-r;
    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        #root   /usr/share/nginx/html; ← 先頭に#を付ける。
        #index  index.html index.htm;  ← 先頭に#を付ける。
        try_files $uri $uri/ /index.php$is_args$args; ← 追記。
    }


    location ~ \.php$ { ← #を外す。
    #    root           html; ← rootはそのまま。
        fastcgi_pass   127.0.0.1:9000; ← #を外す。
        fastcgi_index  index.php; ← #を外す。
        fastcgi_param  SCRIPT_FILENAME  /$document_root/$fastcgi_script_name; ← #を外し、$fastcgi_script_name以前を /$document_root/に変更。
        include        fastcgi_params; ← #を外す。
    } ← #を外す。
```
#### 続いてphp-fpmファイルの編集。
```
sudo vi /etc/php-fpm.d/www.conf

〜変更箇所〜

24行目辺
変更前 user = apache
変更後 user = nginx

26行目辺
変更前 group = apache
変更後 group = nginx
```
#### 設定ファイルの編集を行ったので、設定反映のためNginxを再起動。
```
sudo systemctl restart nginx
```
#### php-fpmを起動。
```
sudo systemctl start php-fpm
```
#### Laravelプロジェクトディレクトリへ移動し、Nginxのアクセス権限を変更する。
```
cd /vagrant/Laravelプロジェクト名
sudo chmod -R 777 storage
```
#### 上記まで完了したら http://192.168.33.19 へアクセス。  
#### 問題無くLaravelが動かせれば完了。
## 〜環境構築の所感〜
今回初めてゼロから環境構築を実施しましたが、所々でエラーが発生し、スムーズには進みませんでした。  
しかし、その発生しているエラーは何が原因なのか、最初はさっぱりでしたが、エラー文を翻訳しながら、  
検索しながら、といろいろしているうちに段々分かるようになっていくのが楽しかったです。  
やはり1番大切なのはエラー文をしっかり読んで、どんなエラーが起こっているのか  
理解した上で調べることだと改めて実感しました。  
それから、今までLinuxと聞いたら無料で手に入るOSのイメージしか無く、  
次PCを組む時に使ってみようかな程度で考えていたのですが、  
今回のこのServer Lessonで触れることができ、理解を深めることができたので良かったです。
## 〜参考サイト〜
- [Giztech -Server Lesson-](https://giztech.gizumo-inc.work/lesson/18)
- [CentOS7にPHP7.3をインストールする](https://www.suzu6.net/posts/152-centos7-php-73/)
- [laravelをバージョン指定(laravel6)でインストールする方法](https://qiita.com/megukentarou/items/d62bf85822cd57c75bff)
- [Laravel6 ログイン機能を実装する](https://qiita.com/ucan-lab/items/bd0d6f6449602072cb87)
- [Qiita マークダウン記法 一覧表・チートシート](https://qiita.com/kamorits/items/6f342da395ad57468ae3)