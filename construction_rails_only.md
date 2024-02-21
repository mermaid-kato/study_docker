## 1. ファイルの作成

```Shell
% mkdir rails_only_app
% touch Dockerfile docker-compose.yml Gemfile Gemfile.lock entrypoint.sh
```

## 2. Dockerfileを編集

```
FROM ruby:3.1
ARG RUBYGEMS_VERSION=3.3.20
RUN mkdir /rails_practice
WORKDIR /rails_practice
COPY Gemfile /rails_practice/Gemfile
COPY Gemfile.lock /rails_practice/Gemfile.lock
RUN bundle install
COPY . /rails_practice

COPY entrypoint.sh /usr/bin/
RUN chmod +x /usr/bin/entrypoint.sh
ENTRYPOINT ["entrypoint.sh"]
EXPOSE 3000

CMD ["rails", "server", "-b", "0.0.0.0"]

apt-get install -y vim
```

## 3. docker-compose.ymlを編集

```yml
version: '3'
services:
  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: password
    ports:
      - '3306:3306'
    command: --default-authentication-plugin=mysql_native_password
    volumes:
      - mysql-data:/var/lib/mysql
  web:
    build: .
    command: bash -c "rm -f tmp/pids/server.pid && bundle exec rails s -p 3000 -b '0.0.0.0'"
    volumes:
      - .:/rails_practice
    ports:
      - "3000:3000"
    environment: RAILS_ENV: development 
    depends_on:
      - db
    environment:
      - EDITOR=vim
    stdin_open: true
    tty: true
volumes:
  mysql-data:
    driver: local
```

## 4. entrypoint.shを編集

```
#!/bin/bash
set -e

rm -f /docker_rails/tmp/pids/server.pid

exec "$@"
```

## 5. Gemfileを編集

```ruby
source 'https://rubygems.org'
gem "rails", ">= 7.1.2"
```

## 6. Applicationを新規作成

```Shell
% docker-compose run web rails new . --force --no-deps --database=mysql --skip-test --webpacker
% docker-compose build
```

## 7. database.ymlを編集

6でdatabase.ymlが作成されているので、そのファイルを修正

```yml
# default: &defaultの部分を以下に修正
default: &default
  adapter: mysql2
  encoding: utf8mb4
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  username: <%= ENV.fetch("MYSQL_USERNAME", "root") %>
  password: <%= ENV.fetch("MYSQL_PASSWORD", "password") %>
  host: <%= ENV.fetch("MYSQL_HOST", "db") %>
```

## 8. DB作成

```Shell
% docker-compose run web rails credentials:edit
% docker-compose run web bundle exec rake db:create RAILS_ENV=development
```

## 9. サーバー起動

```Shell
% docker-compose up
```

## 10. 確認
- localhost:3000で確認できます

## 11. その他
docker-compose run web rails new に失敗する場合以下のコマンドを実行して全て削除する（他のコンテナが消えるので注意）

```Shell
% docker-compose down --rmi all --volumes --remove-orphans
```
