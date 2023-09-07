---
title: "次世代gRPC ライブラリConnect はいったい何が「イイ」のか？"
emoji: "🍣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
---

# はじめに
[Connect](https://connectrpc.com/docs/introduction) は最近注目されているgRPC のライブラリです。
web, Node.js, Go などの環境でgRPC を使うことができるようになります。
gRPC-web などでも同様のことができていましたが、Proxy サーバを立てる必要がありました。
Connect は、Proxy サーバを立てる必要がなく、gRPC の通信を行うことができます。

今回は、Connect を使ったWebアプリケーションを運用してわかった、メリットを紹介します

## メリット
主なメリットは下記の３点です。
- Node.js, Go, Kotlin, Swift などの複数の環境でgRPC を使うことができる 
- HTTP/1.1, HTTP/2, WebSocket などの通信を行うことができる
- gRPC のAPI定義から、サーバ、クライアントの両方に型定義を持ったコードを生成できる


.proto ファイルを使用したスキーマの自動生成を行うだけで、サーバ、クライアントの両方に型定義を持ったコードを生成できることはわかりやすいメリットの一つです。
1. proto ファイルを修正する
2. コードを自動生成する
3. サーバ、クライアントで処理を実装する
この３手順だけでAPIを叩く準備ができることが非常に開発効率を上げてくれます。

ただし、このメリットはaspida, GraphQL などのライブラリでも同様のことが言えますが、Connect が一つ抜きん出ていることはRest API とは異なり、path を考慮せず、メソッドを呼び出すことで通信を行うことができる点です。
例えば、aspida を使用した時、リクエストを行うためのコードは下記のようになります
```ts
import type { AspidaClient, BasicHeaders } from 'aspida'
import type { Methods as Methods0 } from './books'
import type { Methods as Methods1 } from './books/_id@string'

const api = <T>({ baseURL, fetch }: AspidaClient<T>) => {
  const prefix = (baseURL === undefined ? 'http://localhost:3000' : baseURL).replace(/\/$/, '')
  const PATH0 = '/books'
  const GET = 'GET'
  const POST = 'POST'

  return {
    books: {
      _id: (val1: string) => {
        const prefix1 = `${PATH0}/${val1}`

        return {
          /**
           * @returns Book
           */
          get: (option?: { config?: T | undefined } | undefined) =>
            fetch<Methods1['get']['resBody'], BasicHeaders, Methods1['get']['status']>(prefix, prefix1, GET, option).json(),
          /**
           * @returns Book
           */
          $get: (option?: { config?: T | undefined } | undefined) =>
            fetch<Methods1['get']['resBody'], BasicHeaders, Methods1['get']['status']>(prefix, prefix1, GET, option).json().then(r => r.body),
          $path: () => `${prefix}${prefix1}`
        }
      },
      /**
       * @returns Book Created
       */
      post: (option: { body: Methods0['post']['reqBody'], config?: T | undefined }) =>
        fetch<Methods0['post']['resBody'], BasicHeaders, Methods0['post']['status']>(prefix, PATH0, POST, option).json(),
      /**
       * @returns Book Created
       */
      $post: (option: { body: Methods0['post']['reqBody'], config?: T | undefined }) =>
        fetch<Methods0['post']['resBody'], BasicHeaders, Methods0['post']['status']>(prefix, PATH0, POST, option).json().then(r => r.body),
      $path: () => `${prefix}${PATH0}`
    }
  }
}

export type ApiInstance = ReturnType<typeof api>
export default api


```

リクエストを実行するには、下記のような実装を行う必要があります
```ts
import axiosClient from '@aspida/axios'
import axios from 'axios';
import api from "./api/$api"

// APIクライアントの設定
const axiosConfig = { baseURL: "http://localhost:3000" };
const client = api(aspida(axios, axiosConfig));

// id1のBookを取得する
client.books._id("1").$get();
// => { id: 1, status: 'published', name: 'Sample Book1', created_at: '2023-03-08T06:05:38' }

// Bookを作成する
client.books.$post({ body: { name: "Sample", status: "published" });
// => { id: 1, status: 'published', name: 'Sample', created_at: '2023-03-08T06:05:38' }
```
aspida で生成したファイルは、実際にリクエストされているAPIが何なのかを直感的に知ることが難しいと思っています。
REST API の方式をとるため、実際のリクエストからどこでAPIが叩かれているのかを知るのが難しいです。

一方で、Connect の場合、.proto ファイルから生成されるのは
```ts
export const ElizaService = {
    typeName: "eliza.v1.ElizaService",
    methods: {
        /**
         * @generated from rpc eliza.v1.ElizaService.Say
         */
        say: {
            name: "Say",
            I: SayRequest,
            O: SayResponse,
            kind: MethodKind.Unary,
        },
        /**
         * @generated from rpc eliza.v1.ElizaService.Hello
         */
        hello: {
            name: "Hello",
            I: HelloRequest,
            O: HelloResponse,
            kind: MethodKind.Unary,
        },
    }
} as const;

```

実際に使用するには

```ts
"use client";
import styles from "./page.module.css";
import { FormEvent, useState } from "react";
import { PartialMessage } from "@bufbuild/protobuf";

import { createPromiseClient } from "@bufbuild/connect";
import { createGrpcWebTransport } from "@bufbuild/connect-web";
import { ElizaService } from "../../gen/eliza/v1/eliza_connect";
import { SayRequest, SayResponse } from "../../gen/eliza/v1/eliza_pb";

const baseUrl = process.env.NEXT_PUBLIC_GRPC_HOST ?? 'http://localhost:8080';
// gRPCクライアントの初期化
const transport = createGrpcWebTransport({
  baseUrl
});
const client = createPromiseClient(ElizaService, transport);

export default function Home() {
  console.log(baseUrl);
  const [sentence, setSentence] = useState("");
  const [text, setText] = useState("");

  const handleSubmit = async (e: FormEvent) => {
    console.log("greetingMessage");
    e.preventDefault();
    // リクエストメッセージのオブジェクトはPartialMessageを使うと取れます
    const response: PartialMessage<SayRequest> = { sentence };
    // gRPCメソッドを呼び出す
    const greetingMessage: SayResponse = await client.say(response, {headers: { "Access-Control-Allow-Origin": baseUrl }});
    console.log("greetingMessage: ", greetingMessage);
    setText(greetingMessage.sentence);
  };
  return ( // 省略)
}


```

となります。
型定義はすでにされており、どのようなメソッドを呼び出しているのかがわかりやすいです。
どのようなスキーマになっているかは.proto ファイルを見れば一目瞭然で、サーバ、クライアント間で齟齬があることもありません。


さらに、Connect はweb2Server の通信だけでなく、Server2Server の通信もgRPC を使って行うことができるため、クライアント、マイクロサービスサーバなどが増えた場合でも対応が簡単に行えます。
connectサーバ → REST サーバ
connectサーバ → gRPC サーバ
connectサーバ → connect サーバ
のような組み合わせでそれぞれ通信を行うことができます。



## まとめ
Connect は、gRPC を使ったAPIを作成する際に、非常に便利なライブラリです。
gRPC は、web, Node.js, Go などの環境で使用することができますが、それぞれの環境で使用するためには、それぞれのライブラリを使用する必要があります。
Connect を使用することで、gRPC のAPIを作成する際に、サーバ、クライアントの両方に型定義を持ったコードを生成することができます。
また、.proto ファイルから生成されるinterface を実装するだけで、サーバ側の実装が完了するため、ルーティングを考える必要がありません。
APIが増えるにつれ、path の名前のつけ方が難しくなることがありますが、Connect を使うことで、そのような悩みを解決することができます。
昔は適切な名前だったけど、現状から見た時には不適切だということがわかり、名称を変更したい場合でも、.proto ファイルを修正し、コードを自動生成するだけで修正が終わります。




