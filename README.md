# 複数のコンテナからアプリケーションを作成



## 環境変数の設定

  - variableName=value: コンテナ起動時、docker,docker-composeファイルの環境変数を参照する

  - variableName: コンテナ起動時、パソコンの環境変数を参照する

## docker-compose.yml

```
version: '3'
services:
  postgres:
    image: 'postgres:latest'
    environment:
      - POSTGRES_PASSWORD=postgres_password
  redis:
    image: 'redis:latest'
  nginx:
    restart: always
    build:
      dockerfile: Dockerfile.dev
      context: ./nginx
    ports:
      - '3050:80'
  api:
    build:
      dockerfile: Dockerfile.dev
      context: ./server
    volumes:
      - /app/node_modules
      - ./server:/app
    environment:
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - PGUSER=postgres
      - PGHOST=postgres
      - PGDATABASE=postgres
      - PGPASSWORD=postgres_password
      - PGPORT=5432
  client:
    build:
      dockerfile: Dockerfile.dev
      context: ./client
    volumes:
      - /app/node_modules
      - ./client:/app
    environment:
      - WDS_SOCKET_PORT=0
  worker:
    build:
      dockerfile: Dockerfile.dev
      context: ./worker
    volumes:
      - /app/node_modules
      - ./worker:/app
    environment:
      - REDIS_HOST=redis
      - REDIS_PORT=6379
```
- `services.サービス名.environment` : 環境変数の設定

## nginx
- reactサーバーとexpressサーバーにリクエストを仕分けるため、クライアントと２つのサーバーの間にnginxを設置する。ルーティング専用のnginxにするため設定ファイルを作成
- nginxのイメージも作成

### nginx,dockerfile

```
FROM nginx
COPY ./default.conf /etc/nginx/conf.d/default.conf
```
- `Copy ローカルのdefault.confのパス コンテナのdefault.confのパス`: ローカルのdefault.confファイルで既存のdefault.confファイルをオーバーライドする
- `Copy`はファイルもコピーできる。指定したパスにファイルが存在する場合、オーバーライドされる。

### default.conf: nginxの設定 
```
upstream client {
  server client:3000;
}

upstream api {
  server api:5000;
}

server {
  listen 80;

  location / {
    proxy_pass http://client;
  }

  location /api {
    rewrite /api/(.*) /$1 break;
    proxy_pass http://api;
  }

  location /ws {
      proxy_pass http://client;
      proxy_http_version 1.1;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "Upgrade";
  }
}
```
- `upstream サーバー名(docker.composeで指定){server サーバー名: ポート番号}`:ルーティング先のサーバーにポート番号を設置
- `server {listen ポート番号}` : nginxにポート番号を設定
- location パス {proxy_pass url} : nginxに指定したパスで始まるリクエストが来たとき、`http://サーバー名`のサーバーにリクエストを仕分ける
- rewrite : nginxに来たリクエストのパスをサーバーに仕分けるとき、特定のルールに従いパスを形式を変更できる
- `location /ws {...}` : `WebSocket connection to 'ws://localhost:3000/ws' failed`のエラーを解消