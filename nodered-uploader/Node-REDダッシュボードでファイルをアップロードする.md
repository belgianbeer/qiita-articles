# Node-REDダッシュボードでファイルをアップロードする

## はじめに

Node-REDでファイルをアップロードしたいと思ったことはありませんか。筆者も業務でCSVデータをリレーショナルデータベース(以下DB)に登録する必要があって、Node-REDで簡単に実現できないかなと探したところ[node-red-contrib-ui-upload](https://flows.nodered.org/node/node-red-contrib-ui-upload)と関連ライブラリの[node-red-contrib-chunks-to-lines](https://flows.nodered.org/node/node-red-contrib-chunks-to-lines)が見つかりました。実際に使って試してみたところ、よく考えられていてCSVデータをDBに登録する場合等に最適のノードと思われたので、ここで紹介してみたいと思います。

## 早速node-red-contrib-ui-uploadを使ってみる

ファイルのアップロードを行うライブラリが[node-red-contrib-ui-upload](https://flows.nodered.org/node/node-red-contrib-ui-upload)で、**uploadノード**が含まれています。uploadノードはNode-REDダッシュボードのパーツとして作られているため、利用する場合には事前に[node-red-dashboard](https://flows.nodered.org/node/node-red-dashboard)を用意する必要があります。また、ダッシュボードを用意してuploadノードを設定する場合、その**グループの幅を12に指定する**必要があるようです。デフォルトの6のままでは幅が狭く正しく表示できないので注意してください。

最初に作ったフローはこんな感じでした。

![node-red-csv-upload-01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/130638/8c13ce23-7607-ed42-95dd-1713e0986749.png)

見ての通りの単純なもので、uploadノードで受けたCSVデータをcsvノードで1行毎にjsonデータに変換し、それをDBに登録します。

この状態でWebブラウザでダッシュボードにアクセスすると次のように表示されます。

![node-red-csv-upload-02.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/130638/450b47a5-e145-b48b-a8d5-2503926076bd.png)

参照ボタンを押すとファイル選択のダイアログが開くので、ファイルを選ぶと次のように表示が変わります。右三角のボタンを押せばファイルがアップロードできます。

![node-red-csv-upload-03.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/130638/e66d6e1d-33cf-47fe-65b2-72520c27cc00.png)

アップロード途中でのキャンセルボタン(右端の四角ボタン)とアップロード中をプログレスバーで表示する機能もあります[^progress]。

[^progress]:キャンセルボタンやプログレスバーで見る余裕があるほどのアップロードに時間のかかるデータを扱っていないのでこれらの挙動は試せてません。

本フローを試してみたところ、1行程度のCSVデータであれば問題無くDBに登録できたのですが、数百行の規模になると全部を正しく登録できずデータに欠損が生じてしまいました。

## フロー制御を加えてデータの流れすぎを防ぐ

なぜデータが多くなると正しく登録できなくなったのでしょうか。これはNode-REDがnode.jsで実装されているため、node.jsの非同期の挙動がそのままNode-REDに影響していることが原因です。

一般的にDBへのレコードのインサートはDBの処理の中でも時間のかかるものとなります。**node.jsは非同期に動作するため、DB Insertノードの処理でデータの追加に時間がかかると、その完了を待たずにDB Insertノードは次のデータを受け付けてしまいます**。CSVデータの各行をjsonに変換したものが次々と大量に送られてくるようなシチュエーションでは、DBへの登録処理が間に合わなくなってデータ登録が欠損します。

もしDB Insertノードでの処理が終わってから次のデータが送りこまれるように改良できれば、このような事態を回避できます。そのために[node-red-contrib-ui-upload](https://flows.nodered.org/node/node-red-contrib-ui-upload)にはフロー制御の機能が用意してあり、[node-red-contrib-chunks-to-lines](https://flows.nodered.org/node/node-red-contrib-chunks-to-lines)の**chunks-to-linesノード**を組み合わせて利用する例がドキュメントに記載されています[^hoge]。

[^hoge]:最初にドキュメントを読んだ時点でchunks-to-linesノードと組み合わせるところも確認しているのですが、その時点では意味を理解できていませんでした😅

chunks-to-linesノードを組み込んで修正したのが次のフローです。

![node-red-csv-upload-04.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/130638/3ca35d0d-1d23-0d9d-287d-b77935b6482c.png)

最初のフローと比べると、uploadノードとcsvノードの間にchunks-to-linesノードが加わり、DB Insertノードの出力からnext lineというfunctionノードを経由してchunks-to-linesノードにループが出来ています。

chunks-to-linesノードは、uploadノードから送られてくるバイナリデータをフロー制御しながらテキストとして指定行数を切り出して次のノードへ送る機能があります。これによってCSVなどのテキストデータを指定行数(ここでは1行)づつ次のcsvノードに送るようになります。

chunks-to-linesノードが行を切り出して最初の行を出力するのは理解できますが、2行目以降は何をきっかけに出力するのでしょうか。それを行っているのがnext lineのfunctionノードのループです。このfunctionノードの中身はこれだけです。

```javascript
return { tick: true };
```

つまりchunks-to-linesノードは`{ tick: true }`のメッセージを受けると、新たな行を次のノードへ出力する仕様となっているのです。

DB Insertノードで処理が終わるとそのタイミングで出力されるメッセージをきっかけに、next lineのfunctionノードから`{ tick: true }`のメッセージがchunks-to-linesノードに送られて、それによって次CSVデータが送り出されるループによって、確実に1レコードづつDBへの登録ができるようになります。

chunks-to-linesノードは最終行の出力時に`msg.complete`を出力するので、ファイル読み込みの終わりを判断したい場合はmsg.completeがあるかどうかで確認できます。

## chunks-to-linesノードの挙動を理解する

chunks-to-linesノードの挙動を理解するには、next lineのfunctionノードを使ったループの代わりにinjectノードを使って次のようなフローを作ると、手作業で1個づつデータを登録するテストができてわかりやすいでしょう(実際に筆者はこのようにしてテストを行いました)。

![node-red-csv-upload-05.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/130638/ad687f38-02c7-c22d-f823-667e2fc95071.png)

![node-red-csv-upload-06.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/130638/93ee2464-ce07-e97d-24f1-a6fed6a479ca.png)

## chunks-to-linesノードの設定

chunks-to-linesノードには次の設定項目が用意されています。

![node-red-csv-upload-07.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/130638/7e8c672c-be6d-dcda-506e-42e92afa2fbb.png)

![node-red-csv-upload-08.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/130638/16ee3b8c-28ad-20c3-be93-7ce6f40d9863.png)

- Output n lines at a time : 名前の通りchunks-to-linesノードが一度に送り出す行数です。必要に応じて複数行に設定する場合もあるかもしれませんが、多くの用途では1行の設定で利用するのではないかと思います。
- Output format : Text, CSV, JSON Arrayから選びます。
  - Text : 元の行をそのまま出力します。
  - CSV : ヘッダ有りのCSVデータを扱う場合に、出力毎にヘッダを繰り返し付加して出力します。
  - JSON Arry : 1行を1要素としたJSON配列に変換して出力します。
- Text decoding : アップロードデータをどういう文字エンコーディングを前提に扱うかを指定します。UTF-8, Windows-1254(ASCII, ISO-8801), UTF-16, UTF-16LEから選択します。日本語向けのシフトJIS等はありません。

なお、uploadノードにはダッシュボードに必要な項目とノード固有の設定項目がありますが、ノード固有の項目は通常デフォルトで問題無いと思われるのでここでの説明は省略します。

## まとめ

uploadノードによってNode-REDダッシュボードにファイルアップロードの機能を追加でき、さらにchunks-to-linesノードを組み合わせることで、CSVデータを1行づつDBに登録するような処理でも問題無く対応できます。
