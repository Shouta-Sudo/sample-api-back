FROM php:8.1-apache

# iniファイルの読込
ADD ./docker/app/php.ini /usr/local/etc/php/
ADD ./docker/app/apache2/000-default.conf /etc/apache2/sites-enabled/

# Composerインストール
RUN cd /usr/bin && curl -s http://getcomposer.org/installer | php && ln -s /usr/bin/composer.phar /usr/bin/composer

# Middlewareインストール
# 上から順：
# gnupg(=GNU Privacy Guard)、暗号化・署名の為のパッケージ（dockerイメージ構築時に必要）
# ca-certificates（これも認証関係）
# curl \
# Git \
# Vimエディタ(デバッグ・設定など) \
# zip関連ユーティリティ \
# zip解凍ユーティリティ \
# libzip開発ヘッダ \
# gd拡張モジュール；FreeType開発ヘッダ \
# gd拡張モジュール；libpng開発ヘッダ \
# gd拡張モジュール；libjpg開発ヘッダ \
# Oniguruma開発ヘッダ \
# SSL関連ヘッダ \
# MySQL開発ヘッダ \
RUN apt-get update \
    && apt-get install -y \
    gnupg \
    ca-certificates \
    curl \
    git \
    vim \
    zip \
    unzip \
    libzip-dev \
    libfreetype6-dev \
    libpng-dev \
    libjpeg-dev \
    libonig-dev \
    libssl-dev \
    default-libmysqlclient-dev


# ApacheのSSLモジュールの有効化
# RUN a2enmod ssl
# COPY ./docker/app/apache2/default-ssl.conf /etc/apache2/sites-available/default-ssl.conf
# COPY ./docker/ssl/server.crt /etc/ssl/certs/server.crt
# COPY ./docker/ssl/server.key /etc/ssl/private/server.key

# RUN a2ensite default-ssl.conf

# # Apacheを再起動
# RUN service apache2 restart

# pdo_mysql拡張モジュールのインストール（mysqlとの接続が可能に）
RUN docker-php-ext-install pdo_mysql mysqli

# gd拡張モジュール（FreeType、libjpegのヘッダファイルの場所を指定、libpngはシステム標準場所にあるので通常は不要）
# -jオプションとnprocコマンドで、使用可能なCPU数を指定してgd拡張モジュールをインストール
RUN docker-php-ext-configure gd \
    --with-freetype=/usr/include/ \
    --with-jpeg=/usr/include \
    && docker-php-ext-install -j$(nproc) gd


# 認証関係のディレクトリ作成
RUN mkdir -p /etc/apt/keyrings
# npmパッケージのインストール
RUN curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key | gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg \
    && echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_20.x nodistro main" | tee /etc/apt/sources.list.d/nodesource.list\
    && apt-get install -y nodejs \
    && apt-get update
# npm関係のインストール
RUN apt-get install -y npm \
    && apt-get update
RUN npm install npm@9.5.1 -g cross-env
# webpackとnodesassのインストール
RUN npm install webpack --save
RUN npm install node-sass --nodedir=/usr/bin/node

# yarnをインストール(GPG認証を使用)
RUN curl -fsSL https://dl.yarnpkg.com/debian/pubkey.gpg | gpg --dearmor -o /usr/share/keyrings/yarn-keyring.gpg \
    && echo "deb [signed-by=/usr/share/keyrings/yarn-keyring.gpg] https://dl.yarnpkg.com/debian stable main" > /etc/apt/sources.list.d/yarn.list \
    && apt-get update \
    && apt-get install -y yarn

# Xdebugインストール
# RUN pecl install xdebug \
#     && docker-php-ext-enable xdebug

# laravelに必要なmodRewriteの有効化
RUN mv /etc/apache2/mods-available/rewrite.load /etc/apache2/mods-enabled
RUN /bin/sh -c a2enmod rewrite

