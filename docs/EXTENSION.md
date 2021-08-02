# Lasagna-extensioning-plan

[![hackmd-github-sync-badge](https://hackmd.io/9zv5e8oqQnq2nivou5CbVA/badge)](https://hackmd.io/9zv5e8oqQnq2nivou5CbVA)

### Introduction

或る日の私: 「自作SNS作ってdeployすれば何喋ってもええやろ！作るか！」

## Plan

既存のProjectである`Lasagna`を機能追加し, 擬似的にSNSのような機能を実現する.

追加する機能は以下の通りである.

- Private Space
  - お一人さまchatで, 非公開用.
- Alone Chat
  - お一人さまchatで, 公開用.
- Subscribing Space
  - subscribeしたAlone ChatをTL形式で表示する.

~~Spaceは空間と宇宙のmulti-meaning~~
それに従い, 以下の基底機能を追加する.

- Extensible Chat
  - 既存のChat機能を拡張可能にし, 様々な使用用途を実現する.

また, 既存の機能の名称を変更する.

- Chat => Basic Chat
  - Chatが何を指しているのか良く分からなくなる為

Extensible Chatを基底に上記3機能を追加する.

### Extensible Chat

Extensible Chatは名の通りBasic Chatを拡張可能としたものである.  
~~実はUserも拡張が出来るようにしようかなと思ったんだけど, ちょっと面倒なので現段階では廃案です~~  
Chatを操作可能なAPI Interfaceを追加し(// TODO: 別に追加しなくても操作可能じゃない？), それを利用する形で実装される.  
また, 機能群はmodule化し, 任意に追加削除ができるようにする.

要は, Basic Chatを雛形になんでもできるようにしよう, というだけである.

### Private Space

Discordなら「お一人さまServer」  
LINEなら「お一人さまGroup」  
"ファイル転送用", "メモ", "テスト用" …と, 割とheavy-userにはウケる(であろう)あの機能です.  
たしかSlackとかにはあった気がします.

Alone Chatを考案している時に「これ非公開用ver.もあったほうが良くない？ てか無いほうがおかしくない？」と思い立ち作られました. ~~副産物です.~~

追加される機能は以下の通りです.

- undefined

### Alone Chat

実質的な自己の*TL*です. TwitterとかMastodonとかの.  

追加される機能は以下の通りです.

- undefined

### Subscribing Space

Alone Chatを購読するために使用します. 所謂*TL*です.  
複数channelに分けて購読でき, 且つ公開が出来るのでTwitterの*List*みたいな扱いとも言えます. ~~というかもっぱらそうです.~~  

追加される機能は以下の通りです.

- undefined
