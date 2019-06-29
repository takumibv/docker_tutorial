# Docker 入門 #2

- 参考: [Docker入門 #3 【WordPress環境構築】 - Qiita](https://qiita.com/wMETAw/items/710bbbb328b78757463a)

## Dockerでwordpress環境構築

Docker-composeを使って、php+mysqlのwordpress環境を構築

### やること
1. dataonlyコンテナ生成: データの永続化のためにコンテナ
2. webserverコンテナ,dbserverコンテナ生成
3. Docker Composerによる複数コンテナ起動
4. データ確認
5. MySQL接続

### 1. dataonlyコンテナ生成

- `~/Work/docker/wordpress_tutorial`に作成
- dataonlyコンテナを生成するためのイメージを作成
- `/Dockerfile`を作成し、以下の内容にする。
```sh
# ベースとなるイメージをDockerHubから取得
FROM busybox

# 作成者
MAINTAINER YamadaTaro yamada@testmail.com

# ボリュームの指定
VOLUME /var/lib/mysql
```
- busybox: linuxのフォルダ構成と標準コマンドの最小構成バイナリ
- VOLUME: 指定されたパスは、コンテナが削除されてもデータは永続化する

### Dockerビルド

- ビルドしてイメージを作成する。(コンテナを)
```sh
# docker build -t [イメージ名:タグ名] [Dockerfileのあるディレクトリ]
$ docker build -t dataonly .

# イメージ一覧
$ docker images
```

### dataonlyコンテナの起動
- ビルドしたイメージから、コンテナを起動する。
- 下の例だと、`dataonly`イメージを指定して、`dataonly`コンテナを起動している.

```bash
# $ docker run [オプション] コンテナ名 イメージ名
# 1回目
$ docker run -it --name dataonly dataonly

# 2回目以降
$ docker start -it dataonly
```

- `docker run` オプション ( https://www.nyamucoro.com/entry/2018/01/11/224932 )
  - `-i`: 標準入力を開き続ける。
  - `it`: 疑似ttyを割りあてる。

### 2. webserverコンテナ,dbserverコンテナ生成
docker composeを使用して複数のコンテナを一元管理する。
- `docker-compose.yml`に設定する.

```yml
# webサーバーの設定
webserver:
  # wordpressのイメージを取得。ローカルにないので、DockerHubから4.9.8-php7.0-apacheを指定して取得
  image: wordpress:4.9.8-php7.0-apache

  # 80番ポートに転送 ホストポート：コンテナポート
  ports:
   - "80:80"

  # 別コンテナのエイリアスを設定 （リンク）
  links:
   - "dbserver:mysql"

# dbサーバーの設定
dbserver:
  # mysqlのイメージを取得
  image: mysql:5.6

  # データのマウント先を指定 （先ほど作成したdataonlyコンテナ）
  volumes_from:
   - dataonly

  # 環境変数
  environment:
   MYSQL_ROOT_PASSWORD: password
```

### 3. Docker Composeによる複数コンテナの起動

- 起動.プロセス確認
```bash
# docker-compose.ymlで設定した複数のコンテナをビルドして起動
# -d でコンテナをデーモン化
$ docker-compose up -d

$ docker-compose ps

# 停止/再起動
$ docker-composer stop
$ docker-composer start
```

- `http://0.0.0.0:80`にアクセスして確認

### 4. データの確認

- dataonlyにデータがマウントされているか確認
```bash
# -a: dataonlyコンテナにアタッチ
$ docker start -ia dataonly

# 「①dataonlyコンテナを生成」 で指定したパス
$ ls /var/lib/mysql/
-> ファイルができている。
```

- dataonlyが稼働していなくても大丈夫
- volumeに指定することで、不揮発性を持つようになる。

### MySQL接続

```bash
# 先ほど立ち上げたコンテナ名を確認
$ docker-compose ps

# ipアドレスを確認する。
$ docker inspect wordpress_tutorial_dbserver_1 | grep IPAddress

# バーチャルホスト上のコンテナに移動
# docker-compose run [オプション] [サービス] [コマンド]
$ docker-compose run dbserver mysql -h 172.17.0.2 -u root -p

# docker-compose.ymlで指定したパスワード
Enter password:password
```
- 
![68747470733a2f2f71696974612d696d6167652d73746f72652e73332e616d617a6f6e6177732e636f6d2f302f37303135322f30323038613164612d323437362d353962362d393765332d6338656333396663633866392e706e67.png](:storage/0d08e193-098e-4939-9054-4e08e9f82e4e/c99abf47.png)

## 用語
- イメージ: コンテナを起動するための、設定などを含んだもの。
- Composer: 複数のコンテナを一元管理するための仕組み
