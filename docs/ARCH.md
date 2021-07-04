# Lasagna retry

[![hackmd-github-sync-badge](https://hackmd.io/gHQu4I8GT8mF4u9cgBQsug/badge)](https://hackmd.io/gHQu4I8GT8mF4u9cgBQsug)


### Introduction

WebsocketとWebRTCで何か, 作りたくないですか？

### Overview

簡易的なchat roomと通話機能を実装します.

## Arch

具体的な実装を決めていきます.

### endpoints

Discordのもの(`discord.com`)をリスペクトした(なんならそのまんま)ような感じにします.

#### /: start page

アプリの概要とか使い方とか. ~~もっぱらDiscord~~

#### /account: account manager

login他をここででさせる.  
アプリに関係なくアカウントについていじいじできるようにもしたいわね

#### /app: main application

本命のアプリ.  
authに失敗すると`/account`に~~redirectされるか, 誘導される.~~ 誘導にします.

こちらyewアプリとなっております. (他はreact)

### auth

~~あまり厳重にやるつもりもないので(少なくとも初期段階は),~~ `account_name`&`password`を保存, Cookieで認証情報を保持させる…の予定.
Cookieに乗せるのがまずいようだったらやめます, ~~適当に思いついただけなので深く考えていません~~

### deploy

出来ればFrontendはGCFでデプロイしたいが, インスタンス起動時のオーバヘッドなどを鑑みると, GCEで少数のreqを反応良く捌いた方が良い気はする.  
GAEも検討中.

バックエンドはWebsocketもあるのでGCFじゃなくGCEが確定している.  

このPjに限らず, nginx他で1つのipを共有する形にしているため, その形式に合わせる.  
具体的には`docker-compose`である.

後述のVCの仕組みが確定していないのでなんとも言えないが, 鯖側の通信量がそれなりになりそうなので, GCPだと通信量で運営が破産する恐れがある. -> なんとかしないといけない

### application

機能は2つ: テキストベースのchatと, ボイスベースのchat.  

TCはWebsocket上でやりとりをする, 読み込み時には最新nメッセージを取得し, 残りはページングのような処理にしていく.  
(具体的には, Timestampの比較でページの区切りを行う)

VCは未定ですが:  
鯖と接続をし, Room側でStreamの管理を行う.  
n\:(n-1)というよりは, n\:1\:(n-1) って感じ.

## Schema

### \_types

| name | entity | description |
|:---- |:------ |:----------- |
| Uuid | string |             |
| Url  | string |             |
| Date | string |             |

### \_template

| field | type | optional | default | description |
|:----- |:---- |:-------- |:------- |:----------- |
|       |      |          |         |             |


### Viewable

汎用化 (尚早)

```typescript=
type Viewable = {
  "id": Uuid,
  "name": string,
  "view_id": string,
  "created": Date,
  "updated"?: Date,
};
```

| field   | type   | optional | default | description                |
|:------- |:------ |:-------- |:------- |:-------------------------- |
| id      | Uuid   |          |         | unique id (for system)     |
| name    | string |          |         | \[entity\] name            |
| view_id | string |          |         | unique id (for users)      |
| created | Date   |          |         | initial Date               |
| updated | Date   | yes      |         | last updates Date (edited) |


### User

ユーザーの情報.  
パスワードとかそういう系じゃないよ.

```typescript=
type User = Viewable & {
  "icon"?: Url,
  "status"?: string,
  "introduction"?: string,
};
```



| field        | type   | optional | default | description            |
|:------------ |:------ |:-------- |:------- |:---------------------- |
| icon         | Url    | yes      | yes     | to image for icon      |
| status       | string | yes      |         | "status message"       |
| introduction | string | yes      |         | "self introduction"    |

### Room

```typescript=
type Room = Viewable & {
  "icon"?: Url,
  "description"?: string,
  "messages": Array<Message>,
};
```

| field       | type            | optional | default | description         |
|:----------- |:--------------- |:-------- |:------- |:------------------- |
| icon        | Url             | yes      | yes     | to image for icon   |
| description | string          | yes      |         | "status message"    |
| messages    | Array\<string\> |          |         | "self introduction" |

### Message

```typescript=
type Message = Omit<Viewable, "name" | "view_id"> & {
  "author": User,
  "room": Room,
  "content"?: string,
  "medias": Array<Media>,
};
```

| field   | type           | optional | default | description       |
|:------- |:-------------- |:-------- |:------- |:----------------- |
| author  | User           |          |         | written by who    |
| room    | Room           |          |         | written to where  |
| content | string         | yes      |         | message content   |
| medias  | Array\<Media\> |          |         | additional medias |


### Media

```typescript=
type Media = Omit<Viewable, "view_id"> & {
  "url": Url,
  "type": MediaType,
  "hidden"?: boolean,
};
```

| field  | type      | optional | default | description      |
|:------ |:--------- |:-------- |:------- |:---------------- |
| url    | Url       |          |         | to Media Data    |
| type   | MediaType |          |         | kind of Data     |
| hidden | boolean   | yes      | yes     | like a "spoiler" |

### MediaType

```typescript=
type MediaType = "image" | "video" | "sound" | "unknown";
```
