# Docker Sample 2

Docker Composeで複数のコンテナ（nginxのプロキシサーバ、node.jsアプリケーション）を管理するサンプル

## How To

### リポジトリのクローン

```
$ git clone https://github.com/KeitaMoromizato/docker-sample-2
$ cd docker-sample-2
```

### node.jsイメージのビルド

```
$ sudo docker-compose build
```

### 2つのコンテナを起動

```
$ sudo docker-compose run
```

`localhost:8080`にアクセスすると、Webサイトが表示される。

## 解説
### Docker Compose
複数のコンテナを協調して動作させるときは、`docker-compose.yml`を使用する。ここでは、node.jsのプリケーションサーバーと、nginxのプロキシサーバーの2つのコンテナを使用。

```yml
version: '2'
services:
  app:
    build: .
    container_name: 'node'
    ports: 
      - '3000:3000'
  nginx:
    image: nginx
    container_name: 'nginx'
    ports: 
      - '8080:8080'
    volumes:
      - ./nginx/conf:/etc/nginx/conf.d:ro
      - ./nginx/www:/var/www:ro
```

`services`に起動したいコンテナを起動していく。各コンテナの主な設定は以下。

#### build

自作の`Dockerfile`を元にコンテナを起動する場合は、`build`ディレクティブを使用し、相対パスで`Dockerfile`の位置を指定する。

#### image

配布されているイメージを元にコンテナを起動する場合は、`image`ディレクティブでイメージ名を指定する。

#### ports

`ホストポート:コンテナポート`でポートのマッピングを指定する。

#### volumes

コンテナにマウントするボリューム（≒ ディレクトリ）を指定する。主な用途としては

- コンテナを停止してもデータを永続化したい
- ホスト上にあるファイルをコンテナ内で使用したい
- あるコンテナから出力されるファイルを他のコンテナから使用したい

など。ここではホスト上のファイルをコンテナにマウントしており、`ホスト上のパス:コンテナ上のパス`と指定する。`:ro`を付けると、コンテナからはReadOnlyとなるので、基本的には付ける。

### コンテナの解説

nginxの設定は、`./nginx/conf/default.conf`に記述している。このファイルは`volumes`に指定されている通り、コンテナ上の`/etc/nginx/conf.d`にマウントされ、nginxの設定に反映される。

`default.conf`内では、`/var/www`にマウントしたファイルを配信している。

```conf
  location / {
    root /var/www;
  }
```

そして、配布した`index.html`内では`/api`にリクエストを投げている。

```js
const $list = $('#list');
$.ajax({
  url: '/api',
  success: result => {
    $list.append(result.map(item => `<li>${item.text}</li>`));
  }
});
```

`/api`へのアクセスは、nodeコンテナにフォワードされて処理される。

```
  location /api {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_pass http://app:3000/;
  }
```
