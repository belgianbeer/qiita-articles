<!-- https://qiita.com/belgianbeer/items/a5ffcee86d2c660747f6 -->

# 不必要に/usr/srcを持ってませんか？ freebsd-update upgrade が遅いですよ

## はじめに

FreeBSDで新しいリリースが出て既存のFreeBSD環境をアップグレードする場合、`freebsd-update ugprade -r リリース`を実行します。この時`/usr/src`ディレクトリにソースが展開してあると、アップグレードにかかる時間が大きく増えます。これについて簡単に説明してみたいと思います。

なお本記事を書く前に同様の話があるかと検索してみたところ、数日前に投稿された「[FreeBSD14.0 アップグレード祭り](https://qiita.com/strnh/items/1940eec8cb42f2b0327d)」の記事でも中で触れられています。

## どうして/usr/srcがあると遅くなるの？

答えは単純です。/usr/srcがあれば、アップグレード時に更新しなければならないファイルが増えるので時間がかかるのです。

では実際/usr/srcにはどのぐらいのファイルがあるのでしょうか。そもそも全体でシステム関連ファイルの数はいくつあるのでしょう。ということで、簡単に数えてみましょう。

## FreeBSD 14.0-RELEASEでファイル数を数えてみる

次の一覧は FreeBSD 14.0-RELEASEで用意されているインストール用のtarballです。

```
$ ls
base-dbg.txz    kernel-dbg.txz  lib32-dbg.txz   ports.txz       tests.txz
base.txz        kernel.txz      lib32.txz       src.txz
$
```

portsとtestを除いて[^test]、各tarballのファイルを数えてみます。なおこの数え方ではディレクトリを除いているだけなのでシンボリックリンク等が含まれてしまいますが、全体から見れば誤差なので無視します。

[^test]:portsはfreebsd-updateでは関係無いですが、testは含めるべきかもしれません。

```
$ mv ports.txz tests.txz ..
$ ls
base-dbg.txz    kernel-dbg.txz  lib32-dbg.txz   src.txz
base.txz        kernel.txz      lib32.txz

$ for f in * ; do printf '%-16s %6d\n' $f $(tar tf $f | grep -v '/$' | wc -l) ; done
base-dbg.txz       1058
base.txz          27979
kernel-dbg.txz      858
kernel.txz          861
lib32-dbg.txz       243
lib32.txz           710
src.txz           96438
$
```

結果を見るとわかるように、srcのファイル数は10万個近いのに対して、他を合計してもファイル数は3万2千個程度で、3倍程度の違いになります。freebsd-update upgradeやfetchでは変更の無いファイルはダウンロードしませんが、OSのメジャーバージョンの時には大多数のファイルが更新されます。つまり/usr/srcがある場合は、無い場合に比べて4倍程度のファイル数のダウンロードが必要になるわけです。

## freebsd-updateの挙動とSSLのオーバーヘッド

通常freebsd-updateではシステムを更新する場合、次の2つの手順を踏むことになります。

1. 更新に必要なファイルをダウンロードする (freebsd-update fetch や freebsd-update upgrade -r XXXX)
2. ダウンロードしたファイルをインストールする (freebsd-update install)

問題となるのは最初の更新に必要なファイルをダウンロードするところで、freebsd-updateでは更新に必要なファイルを**1個づつダウンロード**します。現状freebsd-updateのサーバーはすべて海外であるため、RTTの関係でファイルのダウンロードに時間に占めるSSLのオーバーヘッドの割合が国内のサーバーに比べて極端に大きくなります。つまり更新ファイルを海外のサーバーから1個づつダウンロードする状況において/usr/srcが存在すると、占有時間が極端に増加することになります[^rtt]。

[^rtt]:実際には各ファイルのチェックサムによる確認なども行われるますが、トータルに占める割合は転送時間の方が大きくなります。

## おわりに

毎回カスタムカーネルを使っている状況なら/usr/srcは必須かもしれませんが、そうでなければ/usr/srcは無い方がリリースの更新時の処理時間を短縮できまるので、迷わず削除をお勧めします。

```
$ doas rm -rf /usr/src/* 

```

アップグレード時の処理時間を短縮する観点から、パッと思いついたアイデアを並べてみます。

- カスタムカーネルが必要な場合でもカーネルコンパイルを行うのは1台だけに限定して、そのサーバーだけ/usr/srcを保持し、他のサーバーで/usr/srcは持たない。
- 単に参照用に/usr/srcを持つのであれば、freebsd-updateでupgradeの作業中のみ/usr/srcを別名に変更し、OSの更新完了後に/usr/srcに名前を戻してあらためて`freebsd-update fetch ; freebsd-updaet install`を実行する
- ソースファイルは/usr/srcには展開せず、他の場所でgitを使って管理する

筆者の場合複数のFreeBSDサーバーを運用していますが、/usr/srcがあるのは1台だけに限定しています。また、昨年投稿した「[ストレージを消費しているものを探す](https://qiita.com/belgianbeer/items/44d2e9c2f9ee43f6537f)」にも書いたようにDEBUG関連のファイルを置かないのも、ストレージ使用量だけでなくファイル数の面でも多少のメリットがあります。
