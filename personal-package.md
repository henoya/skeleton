---
created: 2024-01-14T13:19:28 (UTC +09:00)
tags: []
source: https://www.usagi1975.com/23feb172352/
author: usagi1975
---

# 自作の非公開npmモジュールの作成と共通ライブラリ化

## Node.jsの共通関数をどうしたらいいのか？

複数のWebアプリケーションを作成すると、共通化したい関数が増えてくる。一般的なスタンドアロンのアプリケーションの場合、共通機能を共有ライブラリ化して、各アプリケーションからリンクすれば良い。では、Node.jsアプリケーションの場合、どうしたらいいのか？

現状の環境では、共通化されているのは、個々のWebアプリケーション内だけで、別のアプリケーションで必要になった関数とかは、基本的には`ソースのコピー`でやってしまっている。Gitリポジトリの再構成を検討しているときに、自分的にどうすべきかを検討して、「これならやれそう」という方法を備忘録として残しておく。

## アプリケーションの構成

- 共通機能、Webアプリ(クライアント＋サーバー）ごとで、Gitリポジトリ管理する。(自分の場合は、プライベート環境のGitLab)
- Webアプリ側の`package.json`の`dependencies`で、共通機能のgitリポジトリのURLを指定する。
- Webアプリ側で使用する共通機能を`npm install`する。
- 共通機能のロードは、一般モジュールと同じように`require`を使ってモジュールをロードして使用できるようにする。

## GitLabのリポジトリ構成

とりあえず、グループを1つ作成してその中に機能ごとのプロジェクト（リポジトリ）を作成する。グループは別々になっても影響はない。

| グループ  | プロジェクト |                リポジトリURL                |
| --------- | ------------ | ------------------------------------------- |
| test\_grp | test\_lib    | git+ssh://hoge.com:test\_grp/test\_lib.git  |
| test\_grp | test\_app    | git+ssh://hoge.com:test\_grp/test\_app.git  |
| test\_grp | test\_app2   | git+ssh://hoge.com:test\_grp/test\_app2.git |

## テスト用ライブラリの作成

テスト用のライブラリは以下とする。依存モジュールは`path`で、このモジュールのjoin関数を利用した共通関数としてみた。コールすると、引数a,bをパス文字列で連結する関数である。呼び出し側からは関数名`test_join`という名前で実行できるようにしている。

```js
var path = require('path');
var test_join = function( a , b ){
  return path.join(a, b);
}
module.exports = {
  test_join:test_join
}
```

package.jsonの内容は、特筆すべきものはない。強いて言えば、うっかり`publish`しないようにする場合は、`private:true`としておくと良い。

```json
{
  "name": "test_lib",
  "version": "1.0.0",
  "description": "test library",
  "main": "index.js",
  "private":true,
  "repository": {
    "type": "git",
    "url": "git+ssh://hoge.com:test_grp/test_lib.git"
  },
  "author": "hoge",
  "license": "ISC",
  "dependencies": {
    "path": "^0.12.7"
  }
}
```

もちろん、このプロジェクトを実際にgitリポジトリへ登録しておく必要がある。  
`.gitignore`ファイルには、`/node_modules`を登録して、うっかり余計なものをコミットしないようにしておく。

```bash
> cd test_lib
> git init
> echo /node_modules> .gitignore
> git add .
> git commit -m "test commit"
> git push ssh
```

ライブラリを使用するサンプルアプリケーション側のpackage.jsonとソースを示す。ポイントとしては、使用するモジュールが記載される`dependencies`に自作ライブリラリ名とその`gitリポジトリのURL`を記載する。

リポジトリ名の後ろには、タグ名やコミットIDなどを記載することも可能。無記名だとおそらくHEADがチェックアウトされるはず。詳しくは、

[npm package.json](https://docs.npmjs.com/files/package.json)

を参照する。

```json
{
  "name": "test_app",
  "version": "1.0.0",
  "description": "test app",
  "main": "index.js",
  "repository": {
    "type": "git",
    "url": "git+ssh://hoge.com:test_grp/test_app.git"
  },
  "dependencies":{
 "test_lib":"git+ssh://hoge.com:test_grp/test_lib.git"
  },
  "author": "hoge",
  "license": "ISC"
}
```

テスト用のアプリケーションプログラムの例。いつもと同じように`require`で自作モジュール名を指定する。直接パスを参照するわけではない。

```js
var test_lib = require('test_lib');
console.log( test_lib.test_join('aaa', 'bbb') );
```

この状態で、`npm install`を実行し、gitリポジトリからダウンロードされることを確認する。リポジトリのURLなどが間違っていると、このタイミングでエラーになる。また、test\_libが依存しているpathモジュールも自動でダウンロードされていることがわかる。

```bash
> npm install
test_app@1.0.0 /work/test_app
└─┬ test_lib@1.0.0  (git+ssh://hoge.com:test_grp/test_lib.git#12k9a9f4163ce8e23a7bb66bc14143bfe802bca0)
  └─┬ path@0.12.7 
    ├── process@0.11.9 
    └─┬ util@0.10.3 
      └── inherits@2.0.1 
```

実際にアプリケーションを実行して、モジュールが機能していることを確認する。

```bash
> node index.js
aaa/bbb
```

ネットを探索すると、シンボリックリンクを貼ったり、直接ローカルパスを参照する方法など幾つかあったが、gitリポジトリで繋がっている方式のほうが個人的にはしっくりきたので、とりあえずは上記方式で管理してみることにする。

ほかにもクライアントとサーバーのソースコードの一元化する方法なども見つかったので、必要に迫られたら試してみたい。(正規表現や文字列処理などありそう）
