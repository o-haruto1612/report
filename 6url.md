## URL埋め込み機能

### 実装方針
#### 1. リンク埋め込み機能について
リンク埋め込みとはそもそもどういう機能なのでしょうか。  

URLはそのまま表示すると以下のようになります。  
https://www.google.co.jp/

このくらいの短さのリンクなら良いのですが、もっと長いリンクだったらどうでしょうか。  

https://www.google.com/search?q=mastodon&rlz=1C5CHFA_enJP991JP1007&oq=mastodo&gs_lcrp=EgZjaHJvbWUqBwgAEAAYgAQyBwgAEAAYgAQyBggBEEUYOTIHCAIQABiABDIHCAMQABiABDIHCAQQABiABDIGCAUQRRg9MgYIBhBFGD0yBggHEEUYPagCALACAA&sourceid=chrome&ie=UTF-8

こんな長いリンクなんて出てきた際には文章全体がわかりにくくなってしまいます。  
そんな時Slackでは以下のようにリンクを文字に埋め込む表示方法ができます。

[このリンクにアクセス](https://www.google.com/search?q=mastodon&rlz=1C5CHFA_enJP991JP1007&oq=mastodo&gs_lcrp=EgZjaHJvbWUqBwgAEAAYgAQyBwgAEAAYgAQyBggBEEUYOTIHCAIQABiABDIHCAMQABiABDIHCAQQABiABDIGCAUQRRg9MgYIBhBFGD0yBggHEEUYPagCALACAA&sourceid=chrome&ie=UTF-8)  

【Slackの実際の画面の様子】  
<!--画像挿入-->



それではMastodonの場合はどうでしょうか。  
長いURLをトゥート(投稿のこと)すると以下のようになります。  
<!--画像挿入-->

ただしSlackのようにリンクを文字に埋め込むことはできません。この章ではこの機能をMastodonに実装していきます。

#### 2. 実装のために変更が必要な箇所

リンクを埋め込む機能の実装にはプログラムのどこを変更する必要があるでしょうか。  
以下が最初に浮かんだ変更必要箇所です。

1. __投稿するフォームのUI__  
リンクを埋め込むには、埋め込むリンクとリンクを埋め込むテキストの取得が必要です。そのためには投稿フォームのUIの変更が必要ですね。フロントエンドをいじることになりそうです。  

1. __投稿を表示するUI__  
リンクとテキストを受け取っただけでは、表示できていないので、受け取ったリンクとテキストを実際の画面に反映する必要があります。こちらもUIの変更にあたるので、フロントエンドですね。  

1. __UIで受け取ったデータの処理__  
投稿フォームのUIで受け取ったリンクとテキストのデータを、投稿を表示するUIに渡す必要があります。この部分はデータを管理することになるので、バックエンドに入ってくる可能性があります。  


***

### 投稿するフォームのUIの変更
#### 1. 取っ掛かり
ここでのテーマ
やう必要のあること
この部分→この関数
この部分→この関数
#### 2.コードの変更

1. auto_suggest_textarea.jsx
1. compose_form.jsx


***

### 投稿を表示するUIの変更
#### 1. 取っ掛かり
ここでのテーマ
やう必要のあること
この部分→この関数
この部分→この関数
#### 2.コードの変更
1. text_formatter.jsx
1. formatting_helper.jsx
***

### UIで受け取ったデータの処理の変更
#### 1. 取っ掛かり
ここでのテーマ
やう必要のあること
この部分→この関数
この部分→この関数
#### 2.コードの変更
1. compose_form.jsx
1. text_formatter.rb
1. formatting_helper.rb
***

###



## 使えそうな記法

<details><summary>サンプルコード</summary>

(上に空行が必要)

```rb
puts 'Hello, World'
```
</details>

![Qiita](https://qiita-image-store.s3.amazonaws.com/0/45617/015bd058-7ea0-e6a5-b9cb-36a4fb38e59c.png "Qiita")

<img width="50" src="https://qiita-image-store.s3.amazonaws.com/0/45617/015bd058-7ea0-e6a5-b9cb-36a4fb38e59c.png">

<!-- コメントアウトしたい内容 -->