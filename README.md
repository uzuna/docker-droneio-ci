# Drone io

[Official Repository](https://github.com/drone/drone/issues/2018)

## Install

ここではdockerを使う。設定は[公式の手順](http://docs.drone.io/installation/)ほぼそのままの[docker-compose.yml](./docker-compose.yml)。
8000は別appで使っていたため8180に変更している。

drone-serverはgithubとやり取りをするためのもので、Buildは自体は全てagentで走るため、agentを起動しないとテストがpendingのままになる(後からAgentが認識でき次第実行される)。
drone-agentは見ての通りdocker.sockを渡しておりホストのdockerを利用して動くのは注意すべき点。

repository-serviceと連携することを前提としているので
ここでは[GitHubのOAuthApplication](https://github.com/settings/developers)に登録して｀ClientID｀と`ClientSercret`を取得する。

入力値は以下のようにする
Homepage: https://<my-drone-address>
Callback: https://<my-drone-address>/authorize

#### localhostに立てている場合

[ngrok](https://ngrok.com/)を使ってngrokのアドレスから中継してもらうことで外部とやり取りする。
登録すれば`ngrok authtoken ****`で設定ファイルがhome dir配下に創られるので以下のように書き換える。

設定は以下の通り
```yaml
authtoken: ****
region: ap # region: AsiaPacificを指定
tunnels:
  drone:
    addr: 8180
    proto: http
  registry:
    addr: 5000
    proto: http
```

`ngrok start --all`で起動してアドレスを取得


### Droneioを起動

GitHubのTokenとDroneのアドレスを設定したら`docker-compose up`で起動
GitHubアカウントでログインしたらrepository一覧が表示される。

buildする対象はスイッチを入れたらデフォルトではpushをトリガーに設定される。

buildに関わる設定はほぼ全て対象リポジトリ内の`.drone.yml`に書くため、droneio上で設定することはあまりない。
ここでの例は[https://github.com/uzuna/auto-build-droneio](https://github.com/uzuna/auto-build-droneio)を参照のこと。

#### secret

droneioではbuild実行のpipelineの中で特定のregistryへのpushができる
registryにアクセスするにはusernameとpasswordが必要になるが、ビルドファイルに直接書き込むのはセキュリティ上よろしくない。
そのためいくつかの方法が在るが、今回は`drone secret`を利用する。

drone上でrepository毎に任意の変数を指定できる。
下記の例では`my-drone-address`にpushするときに`username`と`password`を`drone secrets`で置き換えすように指定している

```yml
pipeline:
  docker:
    image: plugins/docker
    repo: my-drone-address/auto-build-node
    secrets: [ docker_username, docker_password ]
    tags: latest
    dockerfile: Dockerfile
    registry: my-drone-address
```

secretsの設定はwebUI上からの他に`drone-cli`を使って行うことが出来る。
tokenは`https://<my-drone-address>/account/token`から取得できる。
コマンドラインで以下のようにTokenを環境変数に入れてdroneコマンドを叩けば設定できる。
連携するrepository毎に指定という点がやや気になる点か。
環境変数のほうが良いかもしれないが動作確認できていない。

```sh
export DRONE_SERVER=https://<my-drone-address>
export DRONE_TOKEN=<jwt>

drone secret add \
  -repository uzuna/auto-build-droneio \
  -name docker_password \
  -value <username>
  
drone secret add \
  -repository uzuna/auto-build-droneio \
  -name <password>
```


### bitbucket serverとの連携

- bitbucketの場合は認証ユーザー固定のためCI結果を見れる=操作できるとなって危険が危ない
- post web hookのPluginが必須

oauth1はRSAキーが必要でapp側が秘密鍵を持つ。
```
openssl genrsa -out key.pem 2048
openssl rsa -in key.pem -pubout > key.pub
```
Incomingにはdroneioで決め打ちをしたCosumerKeyとpublic keyを入れる

applicationは`https://<my.drone.server>`
callbackは`https://<my.drone.server>/authorize`

Outcomingはテキトーに設定
Token/Authはすべて`/authorize`に対して行わせる
