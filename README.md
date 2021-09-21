# システム開発演習A用 質問受け付けシステム

## 概要

- 質問等のための待ち登録システムです
- PHPさえ動けば動作します(5.6以上で作確認済み(8.0で開発))
- データベースはSQLiteを使用しています

## 大まかな利用方法
学生サイドと教員・アシスタントサイドにページが分かれています

### 学生サイド

![](https://i.imgur.com/uDfiVvy.png)


**班名**と**質問内容**をプルダウンメニューから選び、`待ち行列に並ぶ`ボタンをクリックすれば、上部の「現在の待ち状況」に追加されます。

画面更新は10秒ごとに自動で行われますが、`更新`ボタンをクリックしても更新できます。

学生が行える作業は以上です。

### 教員・アシスタントサイド

![](https://i.imgur.com/f42SYEd.png)

「現在の待ち状況」に待ち登録をした情報が並びます。

それぞれの状況には「対応待ち」「対応中」「対応完了」が設定されています(対応完了したものは列から消えるため、表示はされません)。

対応を開始した場合には`対応開始`ボタンをクリックします。状況が「対応中」に変化します。

対応が完了した場合は`対応完了`ボタンをクリックします。内部処理で「対応完了」となり、一覧表示から消えます。

画面更新は10秒ごとに自動で行われますが、`更新`ボタンをクリックしても更新できます。

教員・アシスタントが行える作業は以上です。

## 設置方法等

1. index.php, admin.php, functinos.php, sysdeva_support.dbをWebサーバーのディレクトリに配置します。(相対パスで動作するため、どのディレクトリに設置しても問題ありません)
2. 学生にindex.php側のURLを提示します。
3. 教員及びアシスタントはadmin.phpにアクセスをします。

## Basic認証に関して(デフォルト:無効)

index.php、admin.php共に認証無しで動作します。

認証を求める場合には、index.php, admin.phpそれぞれ冒頭の「`$auth = false`」を「`$auth = true`」に書き換えてください。(この場合ユーザー名とパスワードの設定を忘れないでください。平文で保存しますのであくまでも簡易的な防御です。)

## 初期化方法

DBを初期化したい場合は、`sysdeva_support_orig.db`を`sysdeva_support.db`にリネームしてください。(コピーしてリネームを推奨)

## 処理等について

基本的に複雑な処理にはしていないため、コード内のコメントで伝わる、とは思いますが、DB周りと特徴的な処理について記述します

### 定数系の書き換えについて

index.phpの冒頭(BASIC認証周りの後)に、定数系の変数宣言があります。

`$group_max`には班の最大数、`$question_max`には質問の選択肢の最大数を入れてください。

### DB設計について

```sql=
CREATE TABLE "queue" (
	"id"	INTEGER PRIMARY KEY AUTOINCREMENT,
	"groupnum"	INTEGER NOT NULL,
	"question"	INTEGER NOT NULL,
	"status"	INTEGER NOT NULL
);
```

上記のCREATE文で生成されたSQLiteのDBを利用しています。

利用されていくと、DBの中はこのようにレコードが追加されていきます

![](https://i.imgur.com/Jsmnvam.png)

#### id

idと順番待ちの順位は一致していません。

- 対応完了時はDELETEを行っていない
- idはAUTOINCREMENT

というのが理由です。

(DELETEを使用しても良いのですが、若干雑に作成していることもあり、何らかの事情でDELETEが被ってエラーを起こすよりは、UPDATEで書き換えのほうが事故が少ないのでは、と考えた結果このような実装になりました。DBの容量も大したことにはならないはずです。)


#### groupnum

班番号そのままです。

#### question

質問の内容です。INTの数字で管理をしています。

システム開発演習Aでは

- 1: 授業に関する質問
- 2: 技術に関する質問
- 3: 店長に質問

としています。
この部分を他の選択肢に変えたい場合は、index.phpとadmin.phpを書き換える必要があります。(逆に言えばDBに関しては書き換えずに対応ができる汎用型になっています)

#### status

質問の状態です。INTの数字で管理をしています。

INSERT/UPDATEで書き換えを行います。

index.phpでINSERTされるときは0でINSERTされます

- 0: 対応待ち
- 1: 対応中
- 2: 対応完了(としての扱い)

としています。表記を変えたい場合はindex.phpとadmin.phpの当該表記を書き換える必要があります。

admin.phpで`対応開始`または`対応完了`ボタンが押されたときに、1や2にUPDATEを行っています。

表の表示では、`WHERE NOT status = 2`とすることで対応完了のものを除外して表示しています。

### POSTされてきた値の扱いについて

#### index.php

POSTは班番号と質問内容の2つのみです。

これらはプルダウンメニューから選んで送信するため、内容が決め打ちになっています。そのため、PHPのDBに書き込む処理に関してはそれらの値をそのまま使用します。

簡易さを優先して実装を行ったため、DBに値を入れる際にエスケープ等はしていませんが、POSTされてきた値が

- 数字のみかどうか
- 班番号が存在する範囲か
- 質問番号が存在する範囲か

は確認しています。実装的にこれらがクリアできれば安全であると考えています。

リロードに関して: 完全に同一の登録が対応待ち/対応中の状態で存在していた場合はINSERTを行わないようにしているため、ブラウザのリロードによるPOSTの再送信があっても問題ないようにはしてあります。

#### admin.php

POSTする内容はid, statusです。

idはDB内のidの値です。表に出ているレコードのUPDATEを行うにはidをベースに書き換えるのが手っ取り早いため、このようにしています。

このidの値はWebページ上には表示されていませんが、各ボタンのform内にはhidden属性として記載されています。

同じ用にhidden属性として、そのレコードの現在の状態を数値としてボタンのformに記述しています。

これらの2つの値を元に、POSTされてきたstatusが0だったら当該idのレコードのstatusを1にUPDATE、1だったら2にUPDATEという処理をしています。

POSTされてきた値の確認については

- 数字のみかどうかが
- statusが0~2の間か

については確認していますが、idが存在するかは確認していません。