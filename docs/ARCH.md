# Lasagna retry

[![hackmd-github-sync-badge](https://hackmd.io/gHQu4I8GT8mF4u9cgBQsug/badge)](https://hackmd.io/gHQu4I8GT8mF4u9cgBQsug)


### Introduction

WebsocketとWebRTCで何か, 作りたくないですか？

### Overview

簡易的なchat roomと通話機能を実装します.

## Arch

具体的な実装を決めていきます.

### endpoints (frontend)

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

## Feature and others

機能や運用予定等について色々書いておく.

### auth

~~あまり厳重にやるつもりもないので(少なくとも初期段階は),~~ `account_name`&`password`を保存, Cookieで認証情報を保持させる…の予定.
Cookieに乗せるのがまずいようだったらやめます, ~~適当に思いついただけなので深く考えていません~~

### deploy

出来ればFrontendはGCFでデプロイしたいが, インスタンス起動時のオーバヘッドなどを鑑みると, GCEで少数のreqを反応良く捌いた方が良い気はする.  
GAEも検討中.

バックエンドはWebsocketもあるのでGCFじゃなくGCEが確定している.  

このPjに限らず, nginx他で1つのipを共有する形にしているため, その形式に合わせる.  
具体的には`docker-compose`である.

~~後述のVCの仕組みが確定していないのでなんとも言えないが, 鯖側の通信量がそれなりになりそうなので, GCPだと通信量で運営が破産する恐れがある. -> なんとかしないといけない~~  
確定はしたんだけど. 結局vcはfeatureになりそう  
docker~~ﾄﾞｶｰﾝ~~した時にWebRTCの挙動がどうなるのか不透明なのが一つ,  
後は通信量の話.

### application

機能は2つ: テキストベースのchatと, ボイスベースのchat.  

textchatはWebsocket上でやりとりをする, 読み込み時には最新nメッセージを取得し, 残りはページングのような処理にしていく.  
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
tagの付与されたMessageを取得する際には明示的に要求を示す必要があり, 示されていない又は拒否するように示された場合は内容が断片的になる.  
この"完全な状態の情報を要求する"という行為を「**[tag名]を剥がす**」と呼び, 総称して「**tag剥がし**」と呼びます.

tag剥がしを常に行うことは基本的に**推奨されておらず**, 必要に応じてtag剥がしを行うべきです.  
要は*deprecated*な操作となります. **剥がされていないことが基本**です.  
基本的にはユーザーに確認を取る, 操作を分離する等の対策を取ることを要求します.

#### 「Media#hiddenと被ってない？」

~~ちょっと思った.~~

Messageが持つtag(hiddenの拡張のようなもの)とMediaの持つhidden についてですが,  
*Message has Media*だけではなくMediaも独立して存在できるため, **被ってません**.

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

### "Can I use VC as 1 to 1 calls?"

使え…ないとちょっと不自然だよね… と思ったので*そのうち*実装します. ~~面倒だし~~

### password平文問題について

`passwordを保存 → 文字列比較`  
というのが危険っぽい…のかな？と思いたち, archlinuxのpasswordそれに習って(慣習的？)文字列比較からchecksumへの置換.  

取り敢えずなんかつよそうなのでSHA-3の512bitsで`keccak-512sum`(in archlinux)の使用. rsだと`sha3`を使用. まぁ現行のsha-3 512ですね…  

多分問題はあるんだろうが, 平文保存はアレなので一応対処ってことで…

### mediaのupload機能について

Discordみたいに容量に制約を付けて実現するのも良いんだけど, 取り敢えずはurlからのリンクでのみ対応ってのが望ましい と思っている  
実際ここのfeatureにはuploadの項は無い そういうこと

保存と転送のデータ容量が肥大化するのはまじで困るので正直微妙ではあるけど, 実装だけしてみて無効化しておくってのも運用法の一つかもしれない  
というかdeploy時にconfig読んでその設定道理に起動するような機構を作るべきでは？となっているけど, 今のところは未定です.  
というかそもそも保存の方法が定まっていないのでアレだったり dbでいいんじゃないかな(適当)

*Lasagna-extensioning-plan*と並行して*configuration-plan*と*media-uploader-plan*みたいな拡張案を作成すべきかもしれない  
~~というか名前ダサすぎる 無理に英語使わなくて良いんですよ~~

### 「textchatがAPI-basedなら, voicechatも勝手にclient作れるってこと？」

A. **現段階では無理です.** やめてください. 自己責任でお願いします.

なんでかって, 結局WebRTCを使うわけだから適当にjs使ってclientは仕立てられるだろうから制限は出来ないけど, どうなるかは正直あれだし…  
あと, **client側でAPIを更新する責務**が発生してしまうとcustom-clientを看過するわけにはいかない…となってしまうのですよね.  
結局鯖とConnectionを作るのであれば, Dataの方で色々送信するなりしつつ鯖側で全てどうにかしてしまうのが一番良さそうではあるので, その辺の修正が求められる. ~~気が向けばします.~~

まぁ鯖で全てを処理するようにしてしまった場合, 某DiscordのGateway或いはWebhookのように**client実装者側が責任を持って実装しないと利用できない**みたいなアレになる可能性はあるので, ちょっと…うん…  
そういうの, 例えばratelimitみたいな定数の調整が面倒だし, やりたくないんだけどね…うん…  
まぁ定数はconfigで弄れたとしてもそういう機構が面倒だよねっていう 怠惰である.

### 「監査ログ的なものは作らないの？」

今回作るapplicationは某Discord…というよりかは某LINEに近いものです.  
…LINEに監査ログ, ありますかね？

でも大して実装運用コストはかからないと思うので可能性はあります.  
監査ログっていうかただのRoomのLogですけど.

### feature / release scheduleまとめ

色々ごちゃごちゃになってきたのでまとめ.

#### feature

- User
  - basic-features
    - icon
    - status message
    - self-introduction
- UserAuth
  - basic-features
    - [...]
- Room
  - basic-features
    - icon
    - description
  - text chat
  - voice chat
- Text Chat
  - basic-features
    - text content
  - media contents
  - tag / channel
  - branch / line
- Voice Chat
  - basic-features
    - multiple tracks

#### release schedule

※Backend, not Frontend.

- v0.1
  - User
    - basic-features
  - UserAuth
    - basic-features
  - Room
    - basic-features
- v0.2
  - Text Chat
    - basic-features
- v0.3
  - Text Chat
    - media contents
    - tag / channel
    - branch / line
- v0,4
  - Room
    - text chat
- v0.5
  - Voice Chat
    - basic-features
- v0.6
  - Room
    - voice chat
- \>=v0.7
  - bug-fixes and `rc`
- v1.0
  - 🎉 ***release.***

#### no scheduled feature

- media uploader
- user to user voicechat *(direct call)*

## db / collection

db / collectionの運用法を考えていなかった. ~~愚か~~  
dbは取り敢えず単一のものを使用する予定. もし悪手だったり他理由で変更はあり得る.  
collectionの運用は列挙しながら考えていく.

- **c:** *collection*
  - *data*

.

- **c:** infos
  - **c:** users
    - User
  - **c:** rooms
    - Room
  - **c:** textchats
    - TextChatInfo
  - **c:** voicechats
    - VoiceChatInfo
- **c:** private_infos
  - **c:** auths
    - UserAuthData
  - **c:** tokens
    - Token
- **c:** textchats_messages
  - **c:** [Room#textchat_id]
    - Message
- **c:** voicechats_actions
  - **c:** [Room#voicechat_id]
    - VoiceChatAction

.

## Schema

### type:basic_types

| name | entity | description |
|:---- |:------ |:----------- |
| Uuid | string |             |
| Url  | string |             |
| Date | string |             |

### \_template

| field | type | optional | default | description |
|:----- |:---- |:-------- |:------- |:----------- |

| name | position | description |
|:---- |:-------- |:----------- |


### type:Viewable

汎用化 (尚早)

```typescript=
type Viewable = {
  "id": Uuid,
  "name": string,
  "view_id": string,
  "created": Date,
  "updated"?: Date,
}
```

| field   | type   | optional | default | description                |
|:------- |:------ |:-------- |:------- |:-------------------------- |
| id      | Uuid   |          |         | unique id (for system)     |
| name    | string |          |         | (unique?) \[entity\] name  |
| view_id | string |          |         | (unique?) id (for users)   |
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
}
```

| field        | type   | optional | default | description            |
|:------------ |:------ |:-------- |:------- |:---------------------- |
| icon         | Url    |          | yes     | to image for icon      |
| status       | string | yes      |         | "status message"       |
| introduction | string | yes      |         | "self introduction"    |

### Room

```typescript=
type Room = Viewable & {
  "icon"?: Url,
  "description"?: string,
  "permissions": Array<RoomPermission>,
  "textchat_id": Uuid,
  "voicechat_id": Uuid,
}
```

| field        | type                | optional | default | description       |
|:------------ |:------------------- |:-------- |:------- |:----------------- |
| icon         | Url                 |          | yes     | to image for icon |
| description  | string              | yes      |         | "status message"  |
| permissions  | Array\<Permission\> |          | yes     | given permissions |
| textchat_id  | Uuid                |          |         | textchat id       |
| voicechat_id | Uuid                |          |         | voicechat id      |

on `permissions#expression`

| name   | position | description     |
|:------ |:-------- |:--------------- |
| modify | 0        | modifiable room |
| delete | 1        | deletable ...   |

#### 「Room#view_idとRoom#id って分けて何するの？」

Room#idは内部的に使用しますがRoom#view_idはUser#view_idと同様uniqueな名前に近い立ち位置での意味が存在します.  
なのでidは表面的にはあまり使わないと思って良いです.

### Message

```typescript=
type Message = Omit<Viewable, "name" | "view_id"> & {
  "author": Uuid,
  "textchat": TextChatInfo,
  "content"?: string,
  "medias"?: Array<Media>,
  "tags": Array<string>,
  "branches": Array<string>,
}
```

| field    | type            | optional | default | description       |
|:-------- |:--------------- |:-------- |:------- |:----------------- |
| author   | Uuid            |          |         | written by who    |
| textchat | TextChatInfo    |          |         | written to where  |
| content  | string          | yes      |         | message content   |
| medias   | Array\<Media\>  | yes      | yes     | additional medias |
| tags     | Array\<string\> |          | yes     | message tags      |
| branches | Array\<string\> |          | yes     | message branches  |


### Media

```typescript=
type Media = Omit<Viewable, "view_id"> & {
  "url": Url,
  "type": MediaType,
  "hidden"?: boolean,
}
```

| field  | type      | optional | default | description      |
|:------ |:--------- |:-------- |:------- |:---------------- |
| url    | Url       |          |         | to Media Data    |
| type   | MediaType |          | yes     | kind of Data     |
| hidden | boolean   |          | yes     | like a "spoiler" |

### MediaType

```typescript=
type MediaType = "image" | "video" | "sound" | "unknown";
```

### TextChatInfo

```typescript=
type TextChatInfo = Omit<Viewable, "name" | "view_id"> & {
  "channels": Array<Channel>,
  "permissions": Array<Permission>,
  "lines": Array<Line>,
}
```

| field       | type                | optional | default | description       |
|:----------- |:------------------- |:-------- |:------- |:----------------- |
| channels    | Array\<Channel\>    |          |         | channels          |
| permissions | Array\<Permission\> |          | yes     | given permissions |
| lines       | Array\<Line\>       |          |         | lines             |

on `permissions#expression`

| name          | position | description           |
|:------------- |:-------- |:--------------------- |
| view          | 0        | viewable message      |
| post          | 1        | postable ...          |
| modify_mine   | 2        | modifiable my ...     |
| modify_others | 3        | modifiable others ... |
| delete_mine   | 4        | deletable my ...      |
| delete_others | 5        | deletable others ...  |


### Channel

```typescript=
type Channel = Omit<Viewable, "view_id" | "updated"> & {
  "description": string,
}
```
| field       | type   | optional | default | description         |
|:----------- |:------ |:-------- |:------- |:------------------- |
| description | string | yes      |         | channel description |

### Line

```typescript=
type Line = Omit<Viewable, "view_id" | "updated"> & {
  "description": string,
}
```
| field       | type   | optional | default | description      |
|:----------- |:------ |:-------- |:------- |:---------------- |
| description | string | yes      |         | line description |

### VoiceChatInfo

API側からも情報を取得できるようにしたかったけど悶々として策定が進まなかったので別案にて.

```typescript=
type VoiceChatInfo = Omit<Viewable, "name", "view_id"> & undefined & {
  "permissions": Array<Permission>,
}
```

未策定です.

| field | type | optional | default | description |
|:----- |:---- |:-------- |:------- |:----------- |

on `permissions#expression`

| name   | position | description     |
|:------ |:-------- |:--------------- |
| view   | 0        | viewable action |
| action | 1        | postable ...    |


### VoiceChatAction

Logです.

```typescript=
type VoiceChatAction = Omit<Viewable, "name", "view_id", "updated"> & {
  "user": User,
  "type": VoiceChatActionType,
}
```

| field | type                | optional | default | description      |
|:----- |:------------------- |:-------- |:------- |:---------------- |
| user  | User                |          |         | who did actioned |
| type  | VoiceChatActionType |          |         | kind of action   |


### VoiceChatActionType

```typescript=
type VoiceChatActionType = "join" | "quit"
    | "mute" | "unmute"
    | "deafen" | "undeafen"
```


### Permission

```typescript=
type Permission = Omit<Viewable, "name" | "view_id"> {
  "expression": number, // `u32`
  "target": Uuid,
}
```

| field      | type   | optional | default | description           |
|:---------- |:------ |:-------- |:------- |:--------------------- |
| expression | number |          |         | associated permission |
| target     | Uuid   |          |         | link to User#id       |

## Schema (of private data)

### UserAuthData

```typescript=
type UserAuthData = {
  "id": Uuid,
  "name": string,
  "hash": string,
}
```

| field  | type           | optional | default | description   |
|:------ |:-------------- |:-------- |:------- |:------------- |
| id     | Uuid           |          |         | link to User  |
| name   | string         |          |         | unique name   |
| hash   | string         |          |         | password hash |

### Token

```typescript=
type Token = {
  "id": Uuid,
  "target": Uuid,
  "hash": string,
  "expiration": Date,
  "authority": number, // u32
}
```

| field      | type   | optional | default | description          |
|:---------- |:------ |:-------- |:------- |:-------------------- |
| id         | Uuid   |          |         | unique token id      |
| target     | Uuid   |          |         | bound user id        |
| hash       | string |          |         | token hash           |
| expiration | Date   |          |         | valid until          |
| authority  | number |          |         | associated authority |

on `authority`

| name                | position | description |
|:------------------- |:-------- |:----------- |
| identify            | 0        |             |
| user_update         | 1        |             |
| user_auth_update    | 2        |             |
| user_delete_request | 3        |             |
| user_delete_confirm | 4        |             |

## Usecases

### User

#### create

*no authority required*

##### input

```typescript=
{
  "name": User.name,
  "view_id": User.view_id,
  "auth_name": UserAuthData.name,
  "auth_password": string,
}
```

##### output

```typescript=
User
```

#### read

*no authority required*

##### input

```typescript=
{
  "view_id": User.view_id,
}
```

##### output

```typescript=
User
```

#### update

- identify
- user_update
- user_auth_update *?*
  - from `auth_.*`

##### input

```typescript=
{
  "id": User.id,
  "name"?: User.name,
  "view_id"?: User.view_id,
  "icon"?: User.icon,
  "status"?: User.status,
  "introduction"?: User.introduction,
  "auth_name"?: UserAuthData.name,
  "auth_password"?: string,
}
```

##### output

```typescript=
User
```

#### delete (1)

- identify
- user_delete_request

##### input

```typescript=
{
  "id": User.id,
}
```

##### output

```typescript=
{
  "confirm_token": string,
}
```

#### delete (2)

- identify

.

- on `confirmed`
  - identify
  - user_delete_confirm

##### input

```typescript=
{
  "id": User.id,
  "confirmed": string,
}
```

##### output

```typescript=
{
  "you": User,
  "message": string,
}
```

### Room

#### create

- identify

##### input

```typescript=
{
  "name": Room.name
}
```

##### output

```typescript=
Room
```

#### read

*no authority required*

##### input

```typescript=
{
  "view_id": Room.view_id,
}
```

##### output

```typescript=
Room
```

#### update

- identify

.

- permission
  - modify

##### input

```typescript=
{
  "id": Room.id,
  "name"?: Room.name,
  "icon"?: Room.icon,
  "description"?: Room.description,
}
```

##### output

```typescript=
Room
```

#### delete

- identify

.

- permission
  - delete

##### input

```typescript=
{
  "id": Room.id,
}
```

##### output

```typescript=
{}
```

### TextChat

#### create

*cannot access*

##### input

```typescript=
{}
```

##### output

```typescript=
TextChatInfo
```

#### read (1)

*no authority required*

##### input

```typescript=
{
  "id": TextChatInfo.id,
}
```

##### output

```typescript=
TextChatInfo
```

#### read (2)

- identify

.

- permission
  - view

##### input

```typescript=
{
  "id": Message.id,
}
```

##### output

```typescript=
Message
```

#### read (3)

- identify

.

- permission
  - view

##### input

```typescript=
{
  "id": TextChatInfo.id,
  "items": number,
  "sort"?: MessageSort,
  "order"?: Order,
  "page": number,
  "channel"?: Array<Channel.name>,
  "line"?: Array<Line.name>,
  "filters"?: Array<MessageFilter>,
}
```

Filterについて:  
AND, ORでのfilteringは対応するが, 双方を含んだ複雑なfilteringは将来的に実装されるかもしれない. 現時点では対応されない.  
また`filters`は順序が考慮**される**.

```typescript=
type MessageSort = "created" | "last_updated"
```

```typescript=
type Order = "up" | "down"
```

```typescript=
type MessageFilter = {
  "kind": MessageFilterKind,
  "not": boolean,
  "conditions": MessageFilterCondition,
}
```

```typescript=
type MessageFilterKind = "created" | "updated"
    | "last_updated" | "content"
    | "author" | "has_media"
```

`has_media`について:  
複数のMediaTypeを指定した場合, 照合するのは**それを含んでいるか**についてであり, 数については考慮**されない**.

```typescript=
type MessageFilterCondition =
    Date | string | User.view_id | Array<MediaType>
```

##### output

```typescript=
{
  "total": number,
  "total_pages": number,
  "current_page": number,
  "result": Array<Message>,
}
```

#### update (1)

- identify

.

- permission
  - post

##### input

```typescript=
{
  "content"?: Message.content.
  "medias"?: Message.medias,
  "tags"?: Message.tags,
  "branches"?: Message.branches,
}
```

##### output

```typescript=
Message
```

#### update (2)

- identify

.

- permission
  - modify_mine *or* modify_others
    - if Message is posted from you, need **mine**, otherwise it's **others**.

##### input

```typescript=
{
  "id": Message.id,
  "content"?: Message.content.
  "medias"?: Message.medias,
  "tags"?: Message.tags,
  "branches"?: Message.branches,
}
```

##### output

```typescript=
Message
```

#### delete (1)

*cannot access*

##### input

```typescript=
{
  "id": TextChatInfo.id,
}
```

##### output

```typescript=
{}
```

#### delete (2)

- identify

.

- permission
  - delete_mine *or* delete_others
    - same as the Textchat::"update (2)"

##### input

```typescript=
{
  "id": Message.id,
}
```

##### output

```typescript=
{}
```

### VoiceChat

*cannot access*

#### create

##### input

```typescript=
{}
```

##### output

```typescript=
VoiceChatInfo
```

#### read (1)

- identify

.

- permission
  - view

##### input

```typescript=
{
  "id": VoiceChatInfo.id,
}
```

##### output

```typescript=
VoiceChatInfo
```
#### read (2)

- identify

.

- permission
  - view

##### input

```typescript=
{
  "id": VoiceChatAction.id,
}
```

##### output

```typescript=
VoiceChatAction
```
#### read (3)

- identify

.

- permission
  - view

##### input

```typescript=
{
  "id": VoiceChatAction.id,
  "items": number,
  "sort"?: VoiceChatActionSort,
  "order"?: Order,
  "page": number,
  "filters"?: Array<VoiceChatActionFilter>,
}
```

```typescript=
type VoiceChatActionSort = "created"
```

```typescript=
type VoiceChatActionFilter = {
  "kind": VoiceChatActionFilterKind,
  "not": boolean,
  "conditions": VoiceChatActionFilterCondition,
}
```

```typescript=
type VoiceChatActionFilterKind = "created" | "user" | "kind"
```

```typescript=
type VoiceChatActionFilterCondition =
    Date | User.view_id | VoiceChatActionType
```

##### output

```typescript=
{
  "total": number,
  "total_pages": number,
  "current_page": number,
  "result": Array<VoiceChatAction>,
}
```

#### update

- identify

.

- permission
  - action

##### input

```typescript=
{
  "kind": VoiceChatActionKind,
}
```

##### output

```typescript=
VoiceChatAction
```

#### delete

*cannot access*

##### input

```typescript=
{
  "id": VoiceChatInfo.id,
}
```

##### output

```typescript=
{}
```