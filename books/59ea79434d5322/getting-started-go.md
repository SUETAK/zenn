---
title: "Getting started [Connect for Go]"
---

# Getting started

Connectは、ブラウザとgRPCに対応したHTTP APIを構築するためのスリムなライブラリです。Protocol
Bufferスキーマでサービスを定義すると、Connectがタイプセーフのサーバーとクライアントのコードを生成します。サーバーのビジネスロジックを埋めれば完了です。手書きのマーシャリング、ルーティング、クライアントコードは必要ありません！

この15分間のウォークスルーでは、Goで小さなConnectサービスを作成するのに役立ちます。手書きで書くもの、Connectが生成するもの、新しいAPIを呼び出す方法などが紹介されています。

## 前提条件

Goの直近2つのメジャーリリースのうち1つ、最低でもGo 1.18が必要です。インストール方法については、Goの「Getting Started」ガイドを参照してください。
また、cURLを使用します。cURLはHomebrewやほとんどのLinuxパッケージマネージャから入手可能です。

## Install tools

まず、新しいGo moduleを作成し、いくつかのコード生成ツールをインストールする必要があります：

```
mkdir connect-go-example
cd connect-go-example
go mod init example
go install github.com/bufbuild/buf/cmd/buf@latest
go install github.com/fullstorydev/grpcurl/cmd/grpcurl@latest
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install github.com/bufbuild/connect-go/cmd/protoc-gen-connect-go@latest
```

`buf`, `protoc-gen-go`, `protoc-gen-connect-go`を`PATH`に設定する必要があります。
もし、`wich buf grpcurl protoc-gen-go protoc-gen-connect-go` が成功しない場合は、Goのインストールディレクトリをパスに追加してください：

```
[ -n "$(go env GOBIN)" ] && export PATH="$(go env GOBIN):${PATH}"
[ -n "$(go env GOPATH)" ] && export PATH="$(go env GOPATH)/bin:${PATH}"
```

## service を定義する

これで自分のserviceにプロトコルバッファのスキーマを描く準備ができました。
ターミナルを開き、下記のコマンドを実行してください

```
mkdir -p greet/v1
touch greet/v1/greet.proto
```

エディターで`greet/v1/greet.proto`を開き、以下の内容を追加してください：

```yaml
syntax = "proto3";

  package greet.v1;

  option go_package = "example/gen/greet/v1;greetv1";

  message GreetRequest {
  string name = 1;
}

  message GreetResponse {
  string greeting = 1;
}

  service GreetService {
  rpc Greet(GreetRequest) returns (GreetResponse) {}
}
```

このファイルは、`greet.v1` Protobufパッケージ、`GreetService`というサービス、`Greet`というメソッドとそのリクエストおよびレスポンス構造を宣言します。
これらのパッケージ、サービス、およびメソッド名は、HTTP APIのURLですぐに再表示されます。

## コード生成

