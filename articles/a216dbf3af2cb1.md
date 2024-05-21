---
title: "サービスベースアーキテクチャを実際に実装して得られた知見のまとめ"
emoji: "✨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

# サービスベースアーキテクチャとは
# 実際に実装してみる

## サービスベースアーキテクチャで実装するサービスを決める
今回は簡単に、お金の管理アプリにしましょう

## サービスベースアーキテクチャのサービスにどんなサービスが必要か決める

お金の管理アプリなので、ユーザー登録系のサービス
お金の管理サービス
管理状態の共有サービス


## 使用する技術とか諸々

クラウドサービス
- google cloud
  - cloud run 
  - Firestore

API フレームワーク
- NestJS

フロントエンドフレームワーク
- React

言語
- TypeScript

リポジトリ構成
- モノレポ
  - ルートディレクトリから3つのサービス用のディレクトリを生やす

リポジトリ内での依存管理
NestJSがDI をいい感じにしてくれるのでなるべくDI して依存関係逆転を行いつつ疎結合にしていく

注意。NestJS をモノレポで実装する場合、1つのルートディレクトリ配下に複数のサービスを作る時は`nest new {project-name} --skip-git` と入力すること


# 環境構築
各種サービスが1つのFirestore に接続するので、ルートディレクトリで
`firebase login`
`firebase init`
を実行


# gcloud をinstall するためにpythonとjava をインストール
mac
```
brew install openjdk@11
brew link --force --overwrite openjdk@11
```

```
echo 'export PATH="/usr/local/opt/openjdk@11/bin:$PATH"' >> ~/.zshrc
export CPPFLAGS="-I/usr/local/opt/openjdk@11/include"
```

gcloud を実行。このコマンドで、firestore のエミュレーターをローカルで起動してくれます。
インターネット接続なしでローカルでの動作を提供してくれます。

ただし、メモリにデータを保存するので、データは基本的に揮発性です
```shell
gcloud emulators firestore start
```
 
リモートのデータをエミュレーターに渡すには、適したデータをローカルに落としてくる必要がある
firestore のデータはのエクスポート先は、同じリージョン内にあるcloudstrage にしか渡せない。
default の場合、num5 ならus-central1 がエクスポートできる先になる
```
gcloud auth login
```

エクスポート
```shell
gcloud firestore export gs://suetak-firestore
```

ローカルにコピー
```shell
gsutil cp -r gs://{バケット名称}/2024-05-21T16:53:40_13998 ~{現在のディレクトリまでのpath}/{好きなディレクトリ}
```


