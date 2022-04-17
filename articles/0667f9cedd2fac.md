---
title: "Docker を使った環境の共有にめちゃくちゃ苦戦した話"
emoji: "💭"
type: "tech"# tech: 技術記事 / idea: アイデア
topics: [ "docker", "Go", "mysql" ]
published: false
---

# Docker を使った環境の共有は思った以上になんとかならない
友人と開発を進めるにあたって、同じ環境を用意できるようにするために、docker を使用することにしたのですが、色々な場面で僕の環境では立ち上がるけど、友人の環境では立ち上がらない。実行できない。などの問題が頻発したため、起きた出来に関してまとめます

今回のdocker-compose の使用用途は、app → db へのSQL発行です

今回は友人と開発を進めるにあたって、ぶつかった壁に関して記載していきます
# Golang コンテナに関して
## Golang コンテナの立ち上げができない
最初に作成したdocker-compose.yaml のGolang コンテナの構成は下記
```dockerfile:docker-compose.yaml

```


## DB コンテナが立ち上がらない

## 