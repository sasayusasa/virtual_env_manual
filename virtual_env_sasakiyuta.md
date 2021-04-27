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
作業用ディレクトリを作成し、そのディレクトリ内で使うBOXを宣言。

    mkdir ディレクトリ名
    cd ディレクトリ名
    vagrant init centos/7
### 2. Vagrantfileの編集
Vagrantfile内のポート、IP、同期ファイルに関する記述を編集。  
※下記を全てコメントインし、表示通りに編集。

    config.vm.network "forwarded_port", guest: 80, host: 8080
    config.vm.network "private_network", ip: "192.168.33.19"
    config.vm.synced_folder "./", "/vagrant", type:"virtualbox"
### 3. Vagrantプラグインのインストール/ゲストOSの起動/ログイン
正常に動作するようvagrant-vbguestのプラグインをインストールする。  
(Guest AdditionsのバージョンをVirtualBoxのバージョンに合わせて最新化してくれる。)

    vagrant plugin install vagrant-vbguest
    vagrant plugin list (インストールの確認)
    vagrant up
    vagrant ssh
### 4. 必要なパッケージ/PHP/comperのインストール
パッケージインストール  
下記のコマンドはgit等の開発に必要なパッケージを一括でインストールしてくれる。

    sudo yum -y groupinstall "development tools"
PHPインストール  
yumコマンドでのインストールは古いバージョンのPHPがインストールされてしまうので、  
外部パッケージツールをダウンロードして、そこからPHPをインストールする。  
(laravelを動作させるにはバージョン7以上が必要。)

    sudo yum -y install epel-release wget
    sudo wget http://rpms.famillecollet.com/enterprise/remi-release-7.rpm
    sudo rpm -Uvh remi-release-7.rpm
    sudo yum -y install --enablerepo=remi-php72 php php-pdo php-mysqlnd php-mbstring php-xml php-fpm php-common php-devel php-mysql unzip
    php -v (バージョン確認)
composerインストール  
PHPのパッケージ管理ツールであるcomposerをインストール。

    php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
    php composer-setup.php
    php -r "unlink('composer-setup.php');"
    sudo mv composer.phar /usr/local/bin/composer
    composer -v (バージョン確認)
    exit (1度ゲストOSからログアウト)
### 5. laravel_appのコピー作成
今回使用するlaravel_appを作業ディレクトリにコピーする。

    cd Vagrant用作業ディレクトリ
    cp -r laravel_appディレクトリまでの絶対パス ./
### 6. Nginxインストール
最新版インストールの準備のためファイルの作成と書き込み。

    vagrant ssh
    sudo vi /etc/yum.repos.d/nginx.repo

    〜ファイルに下記内容書き込み〜

    [nginx]
    name=nginx repo
    baseurl=https://nginx.org/packages/mainline/centos/\$releasever/\$basearch/
    gpgcheck=0
    enabled=1
インストールが完了したら起動。

    sudo yum install -y nginx
    nginx -v (バージョン確認)
    sudo systemctl start nginx
### 7. DBのインストール/起動/接続
MySQLのバージョン5.7をインストールするため、  
rpmに新たにリポジトリを追加し、インストールを行う。  
インストール後、起動。

    sudo wget https://dev.mysql.com/get/mysql57-community-release-el7-7.noarch.rpm
    sudo rpm -Uvh mysql57-community-release-el7-7.noarch.rpm
    sudo yum install -y mysql-community-server
    mysql --version (バージョン確認)
    sudo systemctl start mysqld
簡易PW設定用に編集を施す。

    sudo vi /etc/my.cnf

    〜変更箇所〜

    datadir=/var/lib/mysql
    socket=/var/lib/mysql/mysql.sock
    validate-password=OFF ← 追記
編集後、再起動

    sudo systemctl restart mysqld

初期PWを確認し、1度そのPWでログイン。  
その後、新たに自分でPWを再設定し、
データベースの作成。

    sudo cat /var/log/mysqld.log | grep 'temporary password' ← 左記の実行結果 root@localhost: ~この箇所~ をコピー。
    mysql -u root -p ← PWにコピーしたものを貼り付け。
    set password = "新たなpassword";
    create database laravel_app;
### 8. laravel_appディレクトリ下の.envファイル編集/マイグレーション実行
laravel_appを動かせるようにするため、.envファイルを編集。

    DB_PASSWORD=MySQLのPW
laravel_appディレクトリ下にてマイグレーション実行。

    php artisan migrate

### 9. laravel_appを動かすための準備
まずはNginxファイルの編集。

    sudo vi /etc/nginx/conf.d/default.conf

    〜変更箇所〜

    server {
        listen       80;
        server_name  192.168.33.19; ← 変更
        root /vagrant/laravel_app/public; ← 追記
        index  index.html index.htm index.php; ← 追記

        #charset koi8-r;
        #access_log  /var/log/nginx/host.access.log  main;

        location / {
            #root   /usr/share/nginx/html; ← 先頭に#
            #index  index.html index.htm;  ← 先頭に#
            try_files $uri $uri/ /index.php$is_args$args; ← 追記
        }


        location ~ \.php$ { ← #外す
        #    root           html;
            fastcgi_pass   127.0.0.1:9000; ← #外す
            fastcgi_index  index.php; ← #外す
            fastcgi_param  SCRIPT_FILENAME  /$document_root/$fastcgi_script_name; ← #外し、編集
            include        fastcgi_params; ← #外す
        } ← #外す
続いてphp-fpmファイルの編集。

    sudo vi /etc/php-fpm.d/www.conf

    〜変更箇所〜

    user = nginx
    group = nginx
設定ファイルの編集を行ったので、nginxは再起動。  
php-fpmは起動させ、laravel_appへ移動し、  
nginxのアクセス権限を変更する。

    sudo systemctl restart nginx
    sudo systemctl start php-fpm
    cd /vagrant/laravel_app
    sudo chmod -R 777 storage
上記まで完了したら http://192.168.33.19 へアクセス。  
問題無くlaravel_appが動かせれば完了。
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
- [Qiita マークダウン記法 一覧表・チートシート](https://qiita.com/kamorits/items/6f342da395ad57468ae3)