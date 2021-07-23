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

~~後述のVCの仕組みが確定していないのでなんとも言えないが, 鯖側の通信量がそれなりになりそうなので, GCPだと通信量で運営が破産する恐れがある. -> なんとかしないといけない

### application

機能は2つ: テキストベースのchatと, ボイスベースのchat.  

TCはWebsocket上でやりとりをする, 読み込み時には最新nメッセージを取得し, 残りはページングのような処理にしていく.  
(具体的には, Timestampの比較でページの区切りを行う)

~~VCは未定ですが:  
鯖と接続をし, Room側でStreamの管理を行う.  
n\:(n-1)というよりは, n\:1\:(n-1) って感じ.~~
VCでは鯖との1:1のconnectionを張る.  
Connection内のTrackが`clone()`可能なのを利用し鯖を中心に音声を配信する構図とする.  
また, `mute`や`deafen`の感知はTrackの状態を利用することも検討するが, いずれにせよVC内の状態はバックエンド側で管理したいので, 若干微妙ではある.

### Message#tags

tagとは？: Messageに付与する意味的な区別情報  
tagを付けることによって全体で分別が出来る.

特に(N)SFWやSpoilerを意識したもので, これをすることで一般にそのまま流布してほしくないような情報を隠蔽したり全く見えなくしたりすることが出来る.  
Twitterの`#`とは異なる. MastodonのCWに類似する.

これにより, APIから提供される情報が**断片的な場合**が含まれることとなる.  
例えばSpoilerの付与されたMessageを取得する際には明示的に示す必要があり, 示されていない又は拒否するように示された場合は内容が断片的になる.  
この"完全な状態の情報を要求する"という行為を「**[tag名]を剥がす**」と呼び, 総称して「**tag剥がし**」と呼びます.

tag剥がしを常に行うことは基本的に**推奨されておらず**, 必要に応じてtag剥がしを行うべきです.  
要は*deprecated*な操作となります. 基本は **剥がされていないこと** です.  
基本的にはユーザーに確認を取る, 操作を分離する等の対策を取る必要がある.

### Message#branches

~~branchって命名めっちゃ迷ったけどありきたりで若干後悔~~  
これこそTwitterの`#`に該当します.

1Messageに対して無制限個のbranchを指定することが出来ます.  
又, branchはMessageのSchema内に存在しており, twiの`#`みたいにcontent中に埋め込んで付与することは**ありません**. ~~処理が面倒臭すぎる~~  
複数のbranchを指定した場合でも各branchに反映される該当Messageは同一のものです.  
~~もっぱらbranchってのがgitを意識してますし語弊があっては困る~~

branchは内部的にはcollectionになっており, 取り出すときにはqueryで全探索…みたいなことではなく, post時にbranchがMessageの識別情報を格納しています.  
~~データ量増えるけど, queryごりごりするくらいだったらこっちのほうが私は好き~~

### Room#channels

~~Lasagna=extensioning-planからの発案だなんて言えない~~  
~~↑実は考案はあったけど忘れてたんだ, ごめん~~  

channelとはなんぞや？: Messageをtagで分離させて管理したもの.

\-\-\-

一つのTLが存在するとします.  
channelはは適用されていません.  
標準ではtagの付与されたMessageは表示されていません.  
tagは`spoiler`が存在しており, それに対応するchannelが存在しています.

TLに`spoiler`のchannelを適用すると, `spoiler`のtagの付与されたMessage群が以前のTLに統合されて表示されます.  
これで初めてMessageを観測でき, tag剥がしが可能になります.  
tagを剥がすと, 内容を取得することが出来ます.

\-\-\-

結局の所, Roomに対して複数のMessage-collectionを持たせると管理が~~面倒~~煩雑だよなぁとなってしまってこういう結果になったとも言えるし, (元来, )機能をsimpleに保とうねってお気持ちだし, そもそも`Array<Array<Message>>`に近い構造になるのが嫌なのでこうなってます, はい.

### Room#lines

channelのbranch版です.

channelとの違いはbranchが特にcontentを秘匿するような物で無い為, 標準で全てのlineが観測できる点です. それ以外は大差ありません.

### Message関係のまとめ

Messageは以下のようなメタ的な情報を含んでいます:

- tag
  - NSFWやspoiler等の他とは**分離**する必要のあるものに付与
- branch
  - 某Twitterの`#`のように関連性を示したい場合に付与

tag付きMessageはchannelを適用することで観測が可能になり, tag剥がしを持って内容を取得できます.  
一方branch付きMessageは何もせずとも表示されますが, 観測するlineを制限することで該当branchの付与されたMessageのみを観測することが出来ます.

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

## Schema (of private data)

### UserAuthData

```typescript=
type UserAuthData = {
  "id": Uuid,
  "name": string,
  "password": string,
  "tokens": Array<Token>,
}
```

| field    | type           | optional | default | description                            |
|:-------- |:-------------- |:-------- |:------- |:-------------------------------------- |
| id       | Uuid           |          |         | link to User                           |
| name     | string         |          |         | unique name                            |
| password | string         |          |         | password (MUST rethink about security) |
| tokens   | Array\<Token\> |          |         | issued tokens                          |

### Token

```typescript=
type Token = {
  "id": Uuid,
  "token": string,
  "expiration": Date,
  "authority": TokenAuthority,
};
```

| field      | type           | optional | default | description              |
|:---------- |:-------------- |:-------- |:------- |:------------------------ |
| id         | Uuid           |          |         | unique token id          |
| token      | string         |          |         | token (MUST rethink ...) |
| expiration | Date           |          |         | valid until              |
| authority  | TokenAuthority |          |         | associated authority     |

### TokenAuthority

```typescript=
type TokenAuthority = number;
```

// TODO: Usecaseが固まったらそれごとに割り当てれば良いんじゃね？

| name     | position | description |
|:-------- |:-------- |:----------- |
| identify | 0        |             |

