# Construction Laravel with Docker


## Directory structure

```script
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
    ```bash
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
```bash
docker-compose build
# エラーが発生する場合はうしろに、 "--no-cache" を加えて実行する。
```

2. dockerコンテナ起動
```bash
docker-compose up
```

3. dockerコンテナ内に入る
```bash
docker exec -it laravel_app bash
```

4. laravelを作成する
```bash
composer create-project "laravel/laravel=~9.0" --prefer-dist .
# リモートリポジトリからプルする場合；
git clone git@bitbucket.org:apical-point-project/ctoc.git
```

5. 作成したlaravelディレクトリにRWX権限を付与
```bash
chmod -R 777 storage
```

6. php artisan初期処理
```bash
php artisan key:generate

# シンボリックリンク #
php artisan storage:link

# cache生成 #
php artisan optimize
# php artisan view:cache

# ※cache削除するときは； #
php artisan optimize:clear
```



## laravel setup (以下srcディレクトリをルートとする)

1. .env (.env.exmapleも)　編集
```.env
APP_NAME=アプリケーション名

DB_CONNECTION=mysql
DB_HOST=laravel_db
DB_PORT=3306
DB_DATABASE=laravel_db
DB_USERNAME=test_user
DB_PASSWORD=pwd
```

2. app.php編集
```php:/config/app.php
    'timezone' => 'Asia/Tokyo',
    'locale' => 'ja',
    'faker_locale' => 'jp_JP',
```


3. ライブラリ・パッケージインストール
    - vite   .............Vite
        1. npm install
        
        2. configを少々修正
        ```js:/vite.config.js
        // 略
        server: {
            hmr: {
                host: 'localhost',
            },
        },
        ```
        
        3. app.jsをちょこっと修正
        ```js:/resources/js/app.js
        // 略
        import '../css/app.css';
        ```
        
    
    - fortiry   ..........認証バックエンド
        1. fortifyインストール
        ```bash
        composer require laravel/fortify
        php artisan vendor:publish --provider="Laravel\Fortify\FortifyServiceProvider"
        ```
        
        2. app.phpのプロバイダに以下追加
        ```php:/config/app.php
        'providers' => [
            // 略
            /*
             * Fortify Service Providers...
             */
            App\Providers\FortifyServiceProvider::class,
        ],
        ```
        
        3. fortify.phpで各認証機能のON/OFFを切替（jetstreamと併用しない場合は、上３つのみが推奨）
        ```php:/config/fortify.php
        'features' => [
            Features::registration(),               // ユーザー登録
            Features::resetPasswords(),             // パスワードリセット
            Features::emailVerification(),          // メールアドレス確認
            // Features::updateProfileInformation(),   // 登録情報の更新
            // Features::updatePasswords(),            // パスワード更新
            // Features::twoFactorAuthentication([     // 二要素認証
                // 'confirm' => true,
                // 'confirmPassword' => true,
                // 'window' => 0,
            // ]),
        ],
        ```
        
        4. ビュー用のルーティングを無効化＆URLにプレフィックスを追加
        ```php:/config/fortify.php
        'prefix' => 'api',
        
        'views' => false,
        ```
        
        5. 各認証機能のカスタマイズ
        app/Actions/Fortifyにある各ファイルを修正可能
        CreateNewUser.php
        → ユーザー登録用
        PasswordValidationRules.php
        → パスワードのバリデーションルールが書かれたファイルで、各クラスの中でトレイトで呼ばれています。
        ResetUserPassword.php
        → パスワードリセット用
        UpdateUserPassword.php
        → パスワード更新用
        UpdateUserProfileInformation.php
        → 登録情報の更新用
        
        6. ログインビューを返す方法を指示
        ```php:/app/Providers/FortifyServiceProvide.php
        
        use Laravel\Fortify\Contracts\LogoutResponse;
        
        public function boot(): void
        {
            // 登録view
            Fortify::registerView(function () {
                return view('auth.register');
            });
            
            // ログインview
            Fortify::loginView(function () {
                return view('auth.login');
            });
            
            // パスワード忘れたview
            Fortify::requestPasswordResetLinkView(function () {
                return view('auth.forgot-password');
            });
            
            // パスワードリセットview
            Fortify::resetPasswordView(function (Request $request) {
                return view('auth.reset-password', ['request' => $request]);
            });
            
            // メール認証view
            Fortify::verifyEmailView(function () {
                return view('auth.verify-email');
            });
            
            // パスワード認証view
            Fortify::confirmPasswordView(function () {
                return view('auth.confirm-password');
            });
        }
        ```
        
        
    
    - 認証系バリデーションメッセージ日本語化
        0. 前提
        先述のapp.php編集において、言語周辺を"ja"に変更していること。
        
        1. インストール
        ```bash
        php -r "copy('https://readouble.com/laravel/8.x/ja/install-ja-lang-files.php', 'install-ja-lang.php');"
        php -f install-ja-lang.php
        php -r "unlink('install-ja-lang.php');"
        ```
        
        2. メッセージ修正
        ```php:/resources/lang/ja/validation
        // 略
        'custom' => [
            'terms' => [
                'required' => '登録には規約への同意が必須となります。',
            ],
        ],
        
        // 略
        
        'attributes' => [
            'name' => 'ユーザー名',
            'email' => 'メールアドレス',
            'password' => 'パスワード',
            'terms' => '規約',
        ],
        ```
        
        3. langディレクトリ直下にja.jsonを追加、最低以下の７つを追加(バリデーションメッセージ)
        ```json:/resources/lang/ja.json
        {
            "The :attribute must be at least :length characters and contain at least one number.": ":attribute は :length 文字以上で、1つ以上の数字が必要です",
            "The :attribute must be at least :length characters and contain at least one special character.": ":attribute は :length 文字以上で、1つ以上の記号が必要です",
            "The :attribute must be at least :length characters and contain at least one uppercase character and one number.": ":attribute は :length 文字以上で、1つ以上の大文字と数字が必要です",
            "The :attribute must be at least :length characters and contain at least one uppercase character and one special character.": ":attribute は :length 文字以上で、1つ以上の大文字と記号が必要です",
            "The :attribute must be at least :length characters and contain at least one uppercase character, one number, and one special character.": ":attribute は :length 文字以上で、1つ以上の数字と記号が必要です",
            "The :attribute must be at least :length characters and contain at least one uppercase character.": ":attribute は :length 文字以上で、1つ以上の大文字が必要です",
            "The :attribute must be at least :length characters.": ":attribute は :length 文字以上でなければなりません",
        }
        ```
        4. 各bladeの記述を英語⇒日本語の多言語対応にする場合；
        ```php:/resources/views/auth/login.php
        /*
         * 二重波括弧＋アンダーバー2本＋括弧＋シングルクォーテーションで記述。
         * ja.jsonに対応している言葉を記述している必要があります。
        */
        {{ __('Login') }}
        {{ __('Enter your e-mail.') }}
        ```
        
    
    - tailwind   .........CSSコンポーネント
        1. インストール
        ```bash
        npm install tailwindcss
        npx tailwindcss init
        ```
        
        2. コンフィグの設定
        ```js:/tailwind.config.js
        content: [
            './storage/framework/views/*.php',
            './resources/**/*.blade.php',
            './resources/**/*.js',
            './resources/**/*.vue',
            "./vendor/laravel/framework/src/Illuminate/Pagination/resources/views/*.blade.php",
        ],
        ```
        
        3. スタイルシートの設定
        ```css:/resources/css/app.css
        @tailwind base;
        @tailwind components;
        @tailwind utilities;
        ```
        
        4. htmlのhead内にスタイル適用
        ```php:/resources/views/layout/app.blade.php
            <meta charset="UTF-8" />
            <meta name="viewport" content="width=device-width, initial-scale=1.0" />
            <link href="/css/app.css" rel="stylesheet">
        ```
        
        5. ラン
        ```bash
        npm run dev
        ```
    
    - daisyUI   ..........CSSライブラリ
        1. インストール
        ```bash
        npm i -D daisyui@latest
        ```
        
        2. tailwindコンフィグに追加
        ```js:/tailwind.config.js
        plugins: [require("daisyui")],
        ```
    
    - Vue   ..............フロントエンド
        1. viteでvue.jsを使用するためのプラグインをインストール
        ```bash
        npm install @vitejs/plugin-vue --save-dev
        ```
        
        2. configに設定を追加
        ```js:vite.config.js
        // 略
        import vue from "@vitejs/plugin-vue";
        
        // 略
            plugins: [
                vue(),
                
        // 略
        ```
        
        3. app.cssまたはbootstrap.cssに記述することで反映される
        
    
    - mobile middleware
        1. Middlewareを作成
        ```bash
        php artisan make:Middleware GetIsMobile
        ```
        
        2. 作成したMiddlewareを編集
        ```php:/app/Http/Middleware/GetIsMobile.php
        
        ```
        
        3. カーネルに追加
        ```php:/app/Http/Kernel.php
        protected $middleware = [
            \App\Http\Middleware\CheckForMaintenanceMode::class,
            // 略
            \App\Http\Middleware\GetIsMobile::class,
        ```
        
        4. viewで使用
        ```php:any.blade.php
        @if($isMobile)
            //モバイルだけで行う処理
        @endif
        
        @if(!$isMobile)
            //モバイル以外だけで行う処理
        endif
        ```



## Laravel blade components directory structure

```script
! ".blade.php"は省略

resources/views/
├── components/
│   │   -----呼び出し先Blade（以下、子コンポーネント／孫コンポーネントと表記）-----
│   ├── layouts/
│   │   ├── global   ..........非ログイン時のページ土台
│   │   └── private   .........ログイン時のページ土台
│   ├── auth/
│   │   ├── session-status   ..
│   │   └── 
│   ├── modules/   ............＜動的ではないモジュールを格納、タイプによるif分岐をすることでファイル数を抑える＞
│   │   ├── button   ..........ボタン系（特に不要？）
│   │   └── modal   ...........モーダルウィンドウを表示させる（成功・失敗・警告など）
│   ├── app-logo   ............アプリケーションロゴ（svg形式）
│   ├── header   ..............ヘッダー
│   ├── footer   ..............フッター
│   └── navigation   ..........ナビゲーションバー（プルダウン形式の場合は不要かもしれない）
├── auth/
│   ├── register   ............初期登録
│   ├── login   ...............ログイン
│   ├── forgot-password   .....パスワードリセットメール送信ページ
│   ├── reset-password   ......パスワードリセットページ
│   └── verify-email   ........メール認証送信完了ページ（兼再送依頼）
│
│   -----呼び出し元Blade（以下、親コンポーネントと表記）-----
├── profile/
├── */   ......................各種ページを格納する場所
├── main   ....................メイン画面（ログアウト後のリダイレクト先）
└── home   ....................ログイン時のホーム画面
```

- 書き方例：

```php
1.
親=main
    <x-layouts.user>content</x-layouts.user>
子=components/layouts/user
    {{ $slot }}

2.
親=main
    <x-layouts.user title="hoge">content</x-layouts.user>
    or
    <x-layouts.user>
        <x-slot name="title">
            hoge
        </x-slot>
        content
    </x-layouts.user>
子=components/layouts/user
    {{ $title }}
    ->タイトルが表示される

3.
親=main
    <x-layouts.user>
        <x-slot name="header">(ここでの内容は、子にて{{ $header }}で反映される)</x-slot>
        content
    </x-layouts.user>
    ※必ず</x-slot>を書くこと！
子=components/layouts/user
    @if (isset($header))
        <x-header />
    @endif
孫=components/header
    ->ファイル内容が子に呼び出される

4.
TODO 認証関係で、表示を切り替える方法

```



## Laravel site structure



