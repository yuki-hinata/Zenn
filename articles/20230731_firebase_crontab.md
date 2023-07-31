---
title: "firebaseのデータを永続化してみた"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Firebase","Tech"]
published: true
publication_name: "manalink_dev"
---
目次
1. 概要と背景
2. 方針
3. 実装
4. まとめ




## 概要と背景
弊社の環境では、ユーザーがログインするためのメールアドレスとパスワードなどをFirebaseのAuthenticationで、チャット部屋の情報などをFirestoreで管理しています。
ただ、これらのデータはdockerを再起動したりすると消えてしまいます。特にチャット部屋に関するテストを行うときには、毎回問い合わせしてチャット部屋を作成する必要があり、非常に手間がかかっていました。
そこで、一度作成したデータを永続化する処理を定期的に実行しておき、再起動してもデータが消えないようにしたいと思いました。今回は自分が実現した方法に関して説明していきたいと思います。

## 方針
定期的に実行するため、crontabを使うか、launchdを使うかの2択のどちらかになるかと思います。結論から言うと、crontabを使った方法で進めることにしました。
launchdを用いた方法をやめた理由は、launchdは各エンジニアのmacに対して設定を行う必要があるため、エンジニアが増えたときに設定を行う手間が増えると考えたためです。そのため、crontabを使った方法を採用しました。
またdockerを使っているため、docker containerの中でcrontabを実行する方針に決めました。

## 実装
```md:docker/local/Dockerfile
FROM node:14.18.1-bullseye

RUN apt-get update -y

RUN apt-get install -y curl openjdk-11-jre-headless cron vim

RUN npm install -g firebase-tools@11.27.0 && mkdir functions

COPY ./package.json ./
COPY ./yarn.lock ./
COPY ./functions/package.json ./functions/.
COPY ./functions/yarn.lock ./functions/.
COPY ./docker/local/crontab /etc/cron.d/crontab

RUN yarn && cd functions && yarn
```

```md:docker/local/crontab
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

*/5 * * * * root cd /opt/workspace/ && yes | npx firebase emulators:export ./data/export_my_data -P default
#
```

```json:package.json
"scripts": {
    "emulators:with_import": "service cron restart && firebase emulators:start -P default --import=./data/export_my_data --export-on-exit",
  },
```

Dockerfileに関しては、特に変わったことをやっていないので、説明を省略します。
package.jsonのscriptsに関しては、cronを起動してからfirebaseのemulatorsを起動しています。また、`--export-on-exit`をつけることでdocker containerを終了したときに、強制的にデータをexportするようにしています。

肝心のcrontabファイル内部でやっていることに関しては、5分ごとにfirebaseのemulators:exportを実行するようにしています。
ここで、exportするディレクトリはdocker containerの中にあるディレクトリを指定しています。これにより、docker containerを再起動してもデータが消えないようになります。

## まとめ
今回は、firebaseのデータを永続化する方法について説明しました。保存先に関してはあくまでも一例なため、参考にしていただければと思います。

