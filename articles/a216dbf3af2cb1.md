---
title: "サービスベースアーキテクチャを実際に実装して得られた知見のまとめ"
emoji: "✨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

# サービスベースアーキテクチャとは
マイクロサービスまでは行かないが、機能ごとのサービスを立てて、DBを共有するアーキテクチャの事
マイクロサービスほど境界を分割しないため、既存のDBからの分割が難しい場合などに収束する

# 実際に実装してみる

## サービスベースアーキテクチャで実装するサービスを決める
今回は簡単に、お金の管理アプリにしましょう

## サービスベースアーキテクチャのサービスにどんなサービスが必要か決める

お金の管理アプリなので、ユーザー登録系のサービス
お金の管理サービス


## 使用する技術とか諸々

クラウドサービス
- google cloud
  - CloudRun
  - Firestore

API フレームワーク
- NestJS

言語
- TypeScript

リポジトリ構成
- モノレポ
  - ルートディレクトリから3つのサービス用のディレクトリを生やす

# 環境構築

リポジトリ内での依存管理
NestJSがDI をいい感じにしてくれるのでなるべくDI して依存関係逆転を行いつつ疎結合にしていく

注意。NestJS をモノレポで実装する場合、1つのルートディレクトリ配下に複数のサービスを作る時は`nest new {project-name} --skip-git` と入力すること

# Firestore とFirebase とgcp
## Firestore とは
Firestore はgoogle cloud とFirebase の両方から接続できるNoSQLデータベース。
Key-Value のペアが保存される「ドキュメント」があり、それが「コレクション」と言う大きなくくりでまとめられます。
RDBで言うテーブルが「コレクション」で、「ドキュメント」がKey。Key に紐づくValue を保存していくようなイメージでしょうか。

ただし、Value は1つだけではなく複数のフィールドも持つことができます。
例えば、Userの1つを示すドキュメントがあり、そのKey が`userId`だとした場合、`name, email` の2つを持つことができます。
この場合はValueは「2つのフィールドを持つ1つのValue」として作成されます。
```
Key: userId, 
Value: 
  - name: "testName"
  - email: "test.com"
```

コレクションはドキュメントをまとめたもの。と言うことになるので、`users` と言うコレクションに対してuserId をKeyに持つドキュメントが1:1...N で紐づくことになります。
Firestoreはスキーマレスであることもあり、RDBのようにDDLなどでこのテーブルはA,B,C が含まれている！と言うのを決めて上げることはしません。
ドキュメントごとに異なるフィールドを持たせることもできます。

詳しくは[Firestore の概要](https://cloud.google.com/Firestore/docs/overview?hl=ja)を参考にしてください

## SDK について
Firestore は上記で少し触れたように、Firebase 用のSDKとgoogle cloud 用のSDKが存在します。
今回のようにAPIサーバから接続したいなら、google cloud用のSDKを使用してください。
接続方法が全く異なりますが、Node.js からの接続を行う場合、Firebase 用とgoogle cloud 用のどちらもインストールできてしまうので注意してください。

Firebase 用は[@firebase/firestore](https://www.npmjs.com/package/@firebase/firestore)

google-cloud 要は[@google-cloud/firestore](https://www.npmjs.com/package/@google-cloud/firestore) 

となっています。

## docker-compose による環境構築
ローカル環境では、直接クラウド環境のFirestore に繋ぐ必要はないため、エミュレーターを使用します。
今回は2つのサービスと、Firestore エミュレーターの合計3つのコンテナを起動させます。

```yaml
version: '3'
services:
  auth-service:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    volumes:
      - ./auth-service:/usr/src/app
    command: npm start
    environment:
      - NODE_ENV=development
    depends_on:
      - firestore-emulator

    networks:
      - emulator-network

  money-service:
    build:
      context: .
      dockerfile: Dockerfile.money
    ports:
      - "3002:3002"
    volumes:
      - ./money-service:/usr/src/app
    command: npm start
    environment:
      - NODE_ENV=development
    depends_on:
      - firestore-emulator

    networks:
      - emulator-network

  firestore-emulator:
    image: gcr.io/google.com/cloudsdktool/cloud-sdk:316.0.0-emulators
    command: gcloud beta emulators firestore start --host-port=0.0.0.0:8080
    ports:
      - "8080:8080"
    networks:
      - emulator-network

networks:
  emulator-network:
    driver: bridge

```



# 実装

## Firestore に接続する
```shell
npm install @google-cloud/Firestore
```

app.module で初期化する



## Firestore にデータを保存する
userRepository を作成して、ユーザーを生成するメソッドを作成。
userService がuserRepository に依存するように実装する
## 相互に通信するようにする

money-service からuser-service のAPIを実行する
```shell
npm i --save @nestjs/axios axios
```


# 本番環境にデプロイする
本番環境で相互に通信しつつ、データが保存されることを確認する

リポジトリにcloudrun を紐づける
ログエクスプローラーからログを辿れる
resource type > cloud build にフィルターを設定すると、何が原因でダメかが書いてある

iam に権限がない場合は有効にする

# 本番環境で動かす
cloudrun にデプロイし、正しくビルドできればコンテナにアクセスできる状態にはなっている

## 本番環境のAPIを実行する

