# 📋 システム構築手順書：スマホLinux（UserLAnd）で作るWebサーバー環境

Android/iOS上のLinuxエミュレータ環境（UserLAnd Ubuntu）において、Webサーバー（Nginx）を構築し、動作検証を行うための手順および構成の記録です。

## 1. 目的（何のためのサーバーか）
* **目的:** スマホ（UserLAnd）環境におけるWebサーバー構築およびLinux（Ubuntu）の操作学習
* **用途:** 自作HTML/CSSの表示テスト、およびローカル環境でのWebページ開発・検証用

## 2. 環境（OS・ホスト情報）
* **OSの種類・バージョン:** Ubuntu 26.04 LTS
* **ホスト名:** localhost
* **IPアドレス:** 127.0.0.1

## 3. 構成（ミドルウェア・バージョン）
* **Webサーバー:** Nginx (バージョン: 1.28.3)
* **利用ポート:** `8080/tcp`（スマホの権限制限回避のため、標準の80番から変更）
* **DB（データベース）:** なし（未インストール）
* **ミドルウェア:** なし（未インストール）

## 4. 構築手順とコマンド記録

### ① パッケージリストの更新とNginxのインストール
```bash
# パッケージリストを最新にする
sudo apt update && sudo apt upgrade -y

# Nginxのインストール
sudo apt install nginx -y

```
### ② 待受ポートの変更（80 → 8080）
UserLAnd環境では特権ポート（1024番以下）が使えないため、設定を変更する。
 * **編集ファイル:** /etc/nginx/sites-available/default
 * **変更内容（差分）:**
```diff
server {
-       listen 80 default_server;
-       listen [::]:80 default_server;
+       listen 8080 default_server;
+       listen [::]:8080 default_server;

        root /var/www/html;
        index index.html index.htm;
}

```
### ③ Nginxの起動とステータス確認
```bash
# 設定ファイルの構文チェック
sudo nginx -t

# Nginxサービスの起動
sudo service nginx start

# 起動状態の確認（Active: active (running) であること）
sudo service nginx status

```
### ④ 動作確認
 * **確認コマンド:**
   ```bash
   curl [http://127.0.0.1:8080](http://127.0.0.1:8080)
   
   ```
 * **ブラウザ確認:**
   スマホのブラウザで http://localhost:8080 にアクセスし、Nginxの初期画面が表示されることを確認。
## 5. トラブルシューティング・気付き
 * **Systemdの制限:**
   UserLAnd環境によっては systemctl コマンドがエラーになる場合がある。その場合は代わりに sudo service nginx start を使用する。
 * **ドキュメントルート:**
   自作のHTML/CSSは /var/www/html/ 配下に配置してテストする。

