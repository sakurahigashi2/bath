# Bahtについて
BahtはWordPressテーマ。Docker上でWordPressを動かす構成になっている。  
この構成で運用しているサイト <https://baht.tokyo/>
## 初回構築手順
構築手順は以下の通り。
## サーバーの設定
### Dockerのインストール
インストール方法は[公式サイト](https://docs.docker.com/install/linux/docker-ce/debian/)を参照。
### Docker-composeのインストール
以下のコマンドでインストール可能。
```
$ sudo curl -L https://github.com/docker/compose/releases/download/1.21.2/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
$ sudo chmod +x /usr/local/bin/docker-compose
```
詳しくは[公式サイト](https://docs.docker.com/compose/install/)を参照。
## アプリの設定
### 環境設定ファイルの用意
config配下に `mysql.env` `wordpress.env` `letsencrypt.env` を配置。  
wordpress配下に `.htaccess` を配置。
##### mysql.env
```
MYSQL_ROOT_PASSWORD=ルートユーザーのパスワード
```
#### wordpress.env
※ wp-config.phpに設定した環境変数が反映。
```
WORDPRESS_DB_NAME=データベース名
WORDPRESS_DB_USER=ルートユーザー推奨(ルート以外だと接続できない問題有)
WORDPRESS_DB_PASSWORD=ルートユーザーのパスワード
WORDPRESS_DB_HOST=mysql(docker-composeのDBに指定している識別名)
```
#### letsencrypt.env
```
DOMAIN=Let's Encrypt(サーバー証明書)に登録するドメイン名
MAIL=何かあった際にLet's Encryptから連絡が届くアドレス
```
#### .htaccess
```
# BEGIN SSL転送設定を行う場合のみ
<IfModule mod_rewrite.c>
RewriteEngine On
RewriteCond %{HTTPS} off
RewriteRule ^(.*)$ https://%{HTTP_HOST}%{REQUEST_URI} [R,L]
</IfModule>
# END SSL転送設定を行う場合のみ

# BEGIN WordPress
<IfModule mod_rewrite.c>
RewriteEngine On
RewriteBase /
RewriteRule ^index.php$ - [L]
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule . /index.php [L]
</IfModule>
# END WordPress
```
## 起動
### コンテナ起動
dockerディレクトリまで移動してdocker-composeコマンドを実行。
```
$ docker-compose up -d
```
### SSLへ登録(Let's Encryptを利用)
SSLに登録する場合のみ以下のコマンドを実行。  
※ [Let's Encrypt](https://letsencrypt.jp)について。
```
$ docker exec baht-app letsencrypt.sh
```
### SSLの自動更新
90日間で証明書の有効期限が切れるので、cronでSSL更新確認コマンドを定期実行することを推奨。  
```
$ echo "0 3 1,15 * *   docker exec baht-app letsencrypt.sh" >> /var/spool/cron/crontabs/root
```
### ブラウザからアクセス
<http://localhost>
## バッチの設定
### 環境設定ファイルの用意
config配下に `db.env` を配置。
#### db.env
```
DB_HOST=mysql(docker-composeのDBに指定している識別名)
DB_PORT=docker-composeのDBに指定しているポート番号
DB_USER=WordPressで使用しているDBユーザー名
DB_PASSWORD=上記ユーザーのパスワード
DB_DATABASE=WordPressで使用しているdatabase
```
### バッチの実行
#### バッチコンテナ内から
```
$ docker exec -it baht-batch bash
$ ruby /usr/local/batch/currency_exchange/runner.rb  "CurrencyExchangeBatch.execute"
```
#### コンテナ外から
```
$ docker exec baht-batch ruby /usr/local/batch/currency_exchange/runner.rb  "CurrencyExchangeBatch.execute"
```
## DBバックアップ
`db/backup` 直下に `wordpress_<実行日>.sql` で出力される。  
※ `wordpress` はデータベース名
#### コンテナ内から
```
$ mysql_bk.sh wordpress
```
#### コンテナ外から
```
$ docker exec baht-mysql mysql_bk.sh wordpress
```
## DB復旧
※ `wordpress` はデータベース名
#### コンテナ内から
```
$ mysql -u<ユーザ名> -p<パスワード> wordpress < <バックアップファイル名>
```
## 備考
### エラー関連
2回目以降の起動時にDBコンテナが起動せず、エラーになってしまう場合には `db/data/tc.log` を削除してから起動する。
### DB関連
DBのデータを確認する場合、DBコンテナに接続してからmysqlを起動する。
```
$ docker exec -it baht-mysql bash
$ mysql -u root -p
```
### フロントエンド関連
フロントエンドは `gulpfile` を使用してコンパイルしたソースを `wordpress/wp-content/themes/baht/assets` 配下に設置している。  
※コンパイル前のソースは `wordpress/wp-content/themes/baht/frontend` に配置している。
#### コンパイル手順
##### 1. `yarn` をグローバルにインストール
```
$  brew install yarn
```
または `npm` をインストール  
バージョン管理ツール込みでのインストールは[こちら](https://qiita.com/taketakekaho/items/dd08cf01b4fe86b2e218)を参照
##### 2.パッケージ群をインストール
```
$ yarn install
```
`npm` の場合は
```
$ npm install
```
##### 使い方
postCSS,JSをコンパイル
```
$ gulp allSetAssets
```
ファイルの変更監視
```
$ gulp watch
```
