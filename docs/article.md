# Algoliaを使って全文検索可能なTODOアプリを作る

## はじめに
最近Nuxt.jsとFirebaseの手軽さに惹かれ家開発のメインウエポンになっています。
Webアプリを作っているとサイト内を横断的に検索できる機能が欲しくなりますが
FirebaseのデータベースRealtime Database・Firestoreは簡単なクエリにしか対応していません。
そこで検索機能実現のためにFirebase公式もオススメしている方法としてAlgoliaという全文検索エンジンのSaasがあるのでそちらを使ってみようと思い立ちました。

## 対象読者
- Nuxt.js・Firebaseでアプリを作るためそれらの基礎知識（この記事では必要以上な解説はしません）
- Algolia聞いたことがあるけど触ったことはない
- Algoliaの使い方を実例を見ながら理解したい

## 登場人物

### Nuxt.js
Nuxt.jsはVue.jsによる開発を便利に構築するためのフレームワークです。
ストア（Vuex）やルーティング（Vue Router）など大規模なVue製アプリケーション開発に
必要なものが標準で含まれていたり、
サーバサイドレンダリング（SSR）への対応が簡単だったりと、
Vue.jsでアプリケーション開発をする際の各ユースケースに上手くレールを敷いて
開発しやすくしてくれるフレームワークです。

### Firebase
アプリケーションのバックエンドを手軽に構築できるクラウド。
各リソースに月毎の使用料の上限が設定されており、それを超えなければずっと無料で使える。
ドキュメントやSDKが非常に開発者フレンドリーなためキャッチアップが簡単。
サービスのプロトタイプ作成や小規模なサイト構築に向いていると思う。

今回利用するのはFirebaseのうち以下のサービス。

| サービス | 説明 |
| --- | --- |
| Hosting | サイトの公開に使います |
| Firestore | TODOの詳細を保存するデータストアとして使います |

### Algolia
今回の主役。
全文検索できるよ。
詳細は後で書く。

## memo
### 概要
- RDBでいうデータベースの中にテーブルがあるようなイメージで、Algoriaにはインデックスがある。テーブルとの違いは検索に特化している点（インデックスをインダイスと呼ぶこともある
- インデックスの中の要素がレコード。レコードはスキーマレスJSONオブジェクト（レコードをオブジェクトと呼ぶこともある
- オペレーションとして「インデクシング」と「検索」がある。インデクシングがレコードの追加更新削除、検索はその名の通りクエリに関係ある要素を探す
- 世界中どこでも100msで返すぜ！と謳っている
- 言語を問わず動く
- タイポにも対応できる
- 検索結果の中でクエリに該当する部分のハイライトができる、ユーザがタイポしててもハイライトが働く
- ファセットが使える（何かよくわからん
- 類義語を登録すると考慮してくれる
- 検索結果が見つからない場合は、クエリの条件を緩めて検索結果を探してくれる（クエリ拡張）
- 位置情報検索もできる
- ソート戦略もいろいろあるよ
- キーを指定して重複排除もできる
- 個人最適かもできる

### 検索エンジンの利用方法
- 検索エンジンの利用方法は3ステップ
  -  データをインデックスに投入
  -  ランキングの設定をする
  - ユーザインターフェイスを構築する

### データの管理
- 追加
  - APIから投入
    - バッチ投入も可能、通信回数が減る上インデクシングのパフォーマンスがかなり向上する
  - ダッシュボードから投入
  - サードパーティからの投入
    - Wordpressとかから入れられるらしい

- 更新
  - リプレイスとPartial Updateがある
    - 1レコードを作るために複数のデータソースがある場合、各データソースが関係ある部分だけを変更するための機能
    - 例：TODOリストインデックスのレコードはTODOの内容とユーザ情報が含まれ、それぞれ別で管理されている場合にユーザ情報のみアップデートしたいなど

- 削除
  - ID指定して消すのとクエリして消すのがある

- データ構造が変得るときもサービスをダウンしなくて済むよう、Reindexingの仕方が書いてある（詳細：https://www.algolia.com/doc/tutorials/indexing/synchronization/atomic-reindexing/）

- 各レコードはobjectIDという識別子を持つ

### データの検索
- バックエンドに検索用APIを作るよりフロントから直接検索メソッド呼ぶことを公式はオススメしている、パフォーマンスのため
- バックアップなどのためにインデックスからデータを全部ぶっこ抜くためのbrowse機能がある
- 1ページの取得件数を指定することでページングできる
- あらかじめハイライトしたい項目を教えておくことで、検索結果がハイライトされた状態で送られてくる
    - ハイライトのためのHTMLを含めて返すので検索結果はサニタイズされない。データ登録時に適切にサニタイズしておくなど、特にユーザが入力してコンテンツを表示する場合は注意が必要
    - ハイライトはデフォルトでレコードの全ての項目が対象となる、設定により絞ることができる
- スニペットなるものも取れる
    - スニペットはマッチした文字列の前後を含むもの
    - デフォルトではオフのためスニペットを取得したい項目を明示する必要がある
- フィルタは大きく3種類。AND OR NOTも使える。
    - faset：要はカテゴリ。数値でない項目をフィルタする場合はカテゴリを設定しとけよってこと
        - `filter: attribute:value`の形式で指定
    - numeric：数値のフィルタ。特に設定はいらん
        - `filter: attribute > 0`のような比較式と`filter: attribute 0 to 10`のような範囲指定がある
    - tagged：_tags項目に値を設定しておくことでフィルタに利用できる
        - `filter: tagvalue`の形式で指定
    - 配列項目の場合、要素のどれかがマッチすれば取得される
    - 日付はUnixtimeに変換してnumeric方式でフィルタする
    - booleanの場合は`filter: attribute:true`/`filter: attribute:false`で検索

### 制限
- パフォーマンス向上のためレコードのサイズは制限がある（無料プランなら10KB、有料プランなら20KB、エンタープライズプランなら設定可能）
  - 大きなドキュメントを作る際はいくつかのレコードに分けて管理する必要がある　

- 利用可能な型
  - string
  - int/float
  - boolean
  - nested objects
  - array

- HTMLが含まれているとクエリの検索対象からは除外される（サニタイズされる）

### 今後読んで理解したい単語
facet filter
snipetted
hightlighted


## 実装

### Nuxt.jsアプリケーションを用意
```
# Nuxt.jsの開発環境を用意して開発サーバを起動するまで
npm i -g create-nuxt-app
npx create-nuxt-app algolia-sample-todo-app
cd algolia-sample-todo-app
npm run dev

# create-nuxt-app
? Project name algolia-sample-todo-app
? Project description This is an TODO management application sample using Nuxt.js and Algolia
? Use a custom server framework none
? Use a custom UI framework bootstrap
? Choose rendering mode Single Page App
? Use axios module no
? Use eslint no
? Use prettier no
? Author name Yutaka Omido
? Choose a package manager npm
```

### 今回の開発に不要なコードを削除してHello, World!
- components/Logo.vueを削除
- layouts/default.vueの<style>タグ内をごっそり削除
- pages/index.vueを以下のように変更

```
<template>
<div>
    <h1>Hello, Nuxt.js</h1>
</div>
</template>

<script>
export default {
}
</script>
```

### Algoliaのインデックスを作る
1. アカウント登録
2. インデックスを作る
※ addObjectした時にインデックス勝手に作られるのでいらないかも

### インデックスにレコードを追加してみよう
いい感じのフォームを作る

```
npm install algoliasearch --save
```

```javascript
import * as algoliasearch from 'algoliasearch'
import config from '~/algolia.config.js'

const client = algoliasearch(config.appId, config.apiKey)
const index = client.initIndex('todo')

await index.addObject(record) // 1つだけ追加
await index.addObjects(records) // 複数追加
```

レコードにobjectIdを自分で付与してもOK
自分で付与した場合、存在するIDを指定すると既存のオブジェクトが上書きされる

### データを取得してみよう
検索UIの構築が楽になるライブラリをAlgoliaは公開している
- InstantSearch.js：リアルタイムに検索結果を反映するUI 
- autocomplete.js：サジェストをドロップダウンで表示
- DocSearch：技術文書などの検索機能用
- Algolia Places：住所のオートコンプリート

### インデックスのレコードを更新してみよう
```
```

### インデックスのレコードを部分更新してみよう


### インデックスのレコードを削除してみよう

