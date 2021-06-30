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

出来ればFrontendはGCFでデプロイしたい. ~~オーバーヘッドやばそうだけど~~  
バックエンドはWebsocketもあるのでGCFじゃなくGCEかなぁと検討中.