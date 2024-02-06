## docker構築手順
### 1. Docker Desktopのインストール
以下のURLからインストールを行う
- https://matsuand.github.io/docs.docker.jp.onthefly/get-docker/

### 2. プロジェクトのディレクトリを作成
```Shell
# ポートフォリオのフォルダを作成
% mkdir portfolio
% cd portfolio

# Railsを入れるフォルダ
% mkdir backend

# Reactを入れるフォルダ
% mkdir frontend

# Docker、Rails、Reactの設定ファイル
% touch docker-compose.yml
% touch backend/Gemfile
% touch backend/Gemfile.lock
% touch backend/entrypoint.sh
% touch backend/Dockerfile
% touch frontend/Dockerfile
```

### 3. Rails側（backend）の設定
Railsアプリケーションを作成するため、以下のファイル群に記述を追加

`backend/entrypoint.sh`
```shell
#!/bin/bash
set -e

# Remove a potentially pre-existing server.pid for Rails.
rm -f /myapp/tmp/pids/server.pid

# Then exec the container's main process (what's set as CMD in the Dockerfile).
exec "$@"
```

`backend/Dockerfile`
```docker
FROM ruby:3.1.0
RUN apt-get update -qq && apt-get install -y build-essential libpq-dev nodejs

WORKDIR /myapp
COPY Gemfile /myapp/Gemfile
COPY Gemfile.lock /myapp/Gemfile.lock
RUN bundle install
COPY . /myapp

# Add a script to be executed every time the container starts.
COPY entrypoint.sh /usr/bin/
RUN chmod +x /usr/bin/entrypoint.sh
ENTRYPOINT ["entrypoint.sh"]
EXPOSE 3000

# Start the main process.
CMD ["rails", "server", "-b", "0.0.0.0"]
```

`backend/Gemfile`
```ruby
# 最新バージョン
source 'https://rubygems.org'
gem "rails", "~> 7.1.3"
```

`docker-compose.yml`
```yml
version: '3.7'

services:
  db:
    image: mysql:8.0
    platform: linux/x86_64
    command: --default-authentication-plugin=mysql_native_password
    ports:
      - "4306:3306"
    volumes:
      - db:/var/lib/mysql
    environment:
      MYSQL_ALLOW_EMPTY_PASSWORD: 'yes'
    security_opt:
      - seccomp:unconfined
  backend:
    build:
      context: ./backend/
      dockerfile: Dockerfile
    stdin_open: true
    tty: true
    volumes:
      - ./backend:/myapp
      - bundle:/usr/local/bundle
    command: bash -c "rm -rf tmp/pids/server.pid && bundle exec rails s -p 3000 -b '0.0.0.0'"
    depends_on:
      - db
    ports:
      - "3001:3000"
    environment:
      TZ: Asia/Tokyo
volumes:
  db:
    driver: local
  bundle:
    driver: local
```

docker-compose runコマンドで新しいコンテナを作成（設定ファイルで定義された通りに、サービスとして新しいコンテナを開始）

```shell
% docker-compose run --no-deps backend rails new . --force -d mysql --api --skip-test
```

`backend/config/database.yml`

```diff
default: &default
  adapter: mysql2
  encoding: utf8mb4
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  username: root
  password:
  socket: /tmp/mysql.sock
+ host: db
```

### 4. React側（frontend）の設定

```diff
version: '3.7'

services:
  db:
    image: mysql:8.0
    platform: linux/x86_64
    command: --default-authentication-plugin=mysql_native_password
    ports:
      - "4306:3306"
    volumes:
      - db:/var/lib/mysql
    environment:
      MYSQL_ALLOW_EMPTY_PASSWORD: 'yes'
    security_opt:
      - seccomp:unconfined
  backend:
    build:
      context: ./backend/
      dockerfile: Dockerfile
    stdin_open: true
    tty: true
    volumes:
      - ./backend:/myapp
      - bundle:/usr/local/bundle
    command: bash -c "rm -rf tmp/pids/server.pid && bundle exec rails s -p 3000 -b '0.0.0.0'"
    depends_on:
      - db
    ports:
      - "3001:3000"
    environment:
      TZ: Asia/Tokyo
+  frontend:
+    build:
+      context: ./frontend/
+      dockerfile: Dockerfile
+    volumes:
+      - ./frontend:/usr/src/app
+    command: sh -c "cd app && npm start"
+    ports:
+      - "3000:3000"
volumes:
  db:
    driver: local
  bundle:
    driver: local
```

```shell
$ docker-compose run --rm frontend sh -c "npx create-react-app app"
```

### 5. 起動
以下のコマンドでサーバー起動

rails → http://localhost:3000
react → http://localhost:3001

```shell
$ docker-compose up
```

compose upを実行した状態でdatabaseを作成

```shell
$ docker-compose exec backend rails db:create
```

## 参考資料
- https://qiita.com/k-penguin-sato/items/e3cc04f787cf3254cfae
