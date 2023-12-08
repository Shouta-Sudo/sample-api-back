# Construction Laravel with Docker


## Directory structure
```
root/
├── docker/
│   ├── app/
│   │   ├── Dockerfile
│   │   ├── php.ini   ............PHP設定ファイル
│   │   └── 000-default.conf   ...Apache設定ファイル
│   ├── db/   ....................Mysqlデータベース
│   │   ├── my.conf   ............Mysql設定ファイル
│   │   ├── initdb/   ............Mysql始動時に起動したいSQL格納
│   │   └── storage/   ...........Mysqlデータ保存用ディレクトリ
│   └── mailpit/   ...............メールサーバー（FakeSMTP）
│       └── Dockerfile   .........SSL、認証の設定（ローカル環境では不要）※
├── src/   .......................laravelソースディレクトリ
├── docker-compose.yml
└── README. md
```

- ※備考：
    mailpitをDockerfileを用いてビルドするとき、Dockercomposeは以下の通りとなる；
    ``` docker-compose.yml
    mailpit:
        container_name: laravel_FSMTP
        build:
            context: .
            dockerfile: ./docker/mailpit/Dockerfile
        tty: true
        ports:
            - 8025:8025
            - 1025:1025
        environment:
            - MP_DATA_FILE: /home/mailpit/mails
            - MP_SMTP_SSL_CERT: /keys/cert.pem
            - MP_SMTP_SSL_KEY: /keys/privkey.pem
            - MP_SMTP_AUTH_FILE: /keys/.htpasswd
        volumes:
            - ./docker/mailpit:/home/mailpit/mails
            - ./keys:/keys
    ``` 


## Version manager
- Docker : 24.0.7
- Docker-compose v2.23.3
- PHP : 8.1 - apache
- OpenSSL : 3.0.8
- Mysql : 8.0
- phpMyAdmin : latest
- mailpit : 1.9
- laravel : 9x
- nodejs : 20x
- npm : 9.5.1
- yarn : latest
- xdebug : latest
- 


## Installation procedure

0. 常時SSL化するときの手順書（・・・現在検証中・・・）
    1. OpenSSL 秘密鍵の生成
    ```
    # ディレクトリ移動 #
    cd (docker-composeのあるディレクトリ)/docker/ssl

    # 秘密鍵生成（server.key） #
    openssl genrsa -out server.key 2048

    # CSRファイル生成（server.csr） #
    openssl req -new -sha256 -key server.key -out server.csr
    >> Common Name (e.g. server FQDN or YOUR name) []:localhost
    >> 上記以外はenterキーを押してください

    # 証明書作成（server.txt） #
    echo "subjectAltName = DNS:localhost" > server.txt
    # （server.crt） #
    openssl x509 -req -sha256 -days 3650 -signkey server.key -in server.csr -out server.crt -extfile server.txt

    server.txtのみ削除してください
    ```
    
    2. 


1. dockerイメージの生成
```
docker-compose build
```

2. dockerコンテナ起動
```
docker-compose up
```

3. dockerコンテナ内に入る
```
docker exec -it laravel_app bash
```

4. laravelを作成する
```
composer create-project "laravel/laravel=~9.0" --prefer-dist .
```

5. 作成したlaravelディレクトリにRWX権限を付与
```
chmod -R 777 storage
```

6. php artisan初期処理
```
php artisan key:generate

# シンボリックリンク #
php artinsa storage:link

# cache生成 #
php artisan optimize
php artisan view:cache

# ※cache削除するときは； #
php artisan optimize:clear
```

## laravel setup
1. .env　編集
    DB_CONNECTION=mysql  
    DB_HOST=laravel_db
    DB_PORT=3306
    DB_DATABASE=laravel_db
    DB_USERNAME=laravel_user
    DB_PASSWORD=laravel_pass
    
    MAIL_HOST=mailpit
    MAIL_PORT=1025
    MAIL_ENCRYPTION=null

2. 

