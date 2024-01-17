<!-- https://qiita.com/belgianbeer/items/962dfd394dbe287856c0 -->

# はじめに

海外の「現地時間は今何時」を知りたいことはよくある話で、単純に現在の時刻を調べるのであれば世界時計を使えばわかるしWebサービスも多数あります。ところが「現地時間の午前9時は日本時間の何時」とか知りたい場合は意外に調べやすありません。スマホにある世界時計の機能も、現在時刻を知るのは簡単ですが、今の時間とは違う時間についてはあまり簡単にはできません。

普段シェルのコマンドラインを利用していることが多いので、時差をコマンドラインで表示するツールが欲しくなったということもあって作ってみたtzdiffコマンドを紹介します。

> この記事の公開の段階ではコマンド名は**timediff**だったが、別のコマンドとコマンド名が被ることが判明したため**tzdiff**という名前に変更した。(2018-09-04 更新)

Githubのtzdiffのページ https://github.com/belgianbeer/tzdiff

# tzdiff の使い方

tzdiff を引数無しで実行すると、/usr/share/zoneinfo の 一覧を表示します。

```
$ tzdiff
Africa/         EET             GMT0            MST             Singapore
America/        EST             Greenwich       MST7MDT         Turkey
Antarctica/     EST5EDT         HST             Mexico/         UCT
Arctic/         Egypt           Hongkong        NZ              US/
Asia/           Eire            Iceland         NZ-CHAT         UTC
...
```

この中からAustraliaを指定すると、Austaraliaのタイムゾーンデータベースの一覧が表示されます。

```
$ tzdiff Australia
Australia/ACT           Australia/Hobart        Australia/Queensland
Australia/Adelaide      Australia/LHI           Australia/South
Australia/Brisbane      Australia/Lindeman      Australia/Sydney
Australia/Broken_Hill   Australia/Lord_Howe     Australia/Tasmania
Australia/Canberra      Australia/Melbourne     Australia/Victoria
Australia/Currie        Australia/NSW           Australia/West
Australia/Darwin        Australia/North         Australia/Yancowinna
Australia/Eucla         Australia/Perth
```

ここでAustralia/Sydneyを指定すると、時差が10時間分表示できます。

```
$ tzdiff Australia/Sydney
Sydney
2023-07-19 23:56 AEST   2023-07-19 22:56 JST
2023-07-20 00:56 AEST   2023-07-19 23:56 JST
2023-07-20 01:56 AEST   2023-07-20 00:56 JST
2023-07-20 02:56 AEST   2023-07-20 01:56 JST
2023-07-20 03:56 AEST   2023-07-20 02:56 JST
2023-07-20 04:56 AEST   2023-07-20 03:56 JST
2023-07-20 05:56 AEST   2023-07-20 04:56 JST
2023-07-20 06:56 AEST   2023-07-20 05:56 JST
2023-07-20 07:56 AEST   2023-07-20 06:56 JST
2023-07-20 08:56 AEST   2023-07-20 07:56 JST
```

このように引数に知りたい地域のタイムゾーンを指定するだけです。デフォルトでは今の時間から10時間分ですが、24時間分表示したいのであれば、最後に数字で指定します。

```
$ tzdiff America/New_York 24
New_York
2023-07-23 02:56 EDT   2023-07-23 15:56 JST
2023-07-23 03:56 EDT   2023-07-23 16:56 JST
2023-07-23 04:56 EDT   2023-07-23 17:56 JST
2023-07-23 05:56 EDT   2023-07-23 18:56 JST
2023-07-23 06:56 EDT   2023-07-23 19:56 JST
2023-07-23 07:56 EDT   2023-07-23 20:56 JST
2023-07-23 08:56 EDT   2023-07-23 21:56 JST
2023-07-23 09:56 EDT   2023-07-23 22:56 JST
2023-07-23 10:56 EDT   2023-07-23 23:56 JST
2023-07-23 11:56 EDT   2023-07-24 00:56 JST
2023-07-23 12:56 EDT   2023-07-24 01:56 JST
2023-07-23 13:56 EDT   2023-07-24 02:56 JST
2023-07-23 14:56 EDT   2023-07-24 03:56 JST
2023-07-23 15:56 EDT   2023-07-24 04:56 JST
2023-07-23 16:56 EDT   2023-07-24 05:56 JST
2023-07-23 17:56 EDT   2023-07-24 06:56 JST
2023-07-23 18:56 EDT   2023-07-24 07:56 JST
2023-07-23 19:56 EDT   2023-07-24 08:56 JST
2023-07-23 20:56 EDT   2023-07-24 09:56 JST
2023-07-23 21:56 EDT   2023-07-24 10:56 JST
2023-07-23 22:56 EDT   2023-07-24 11:56 JST
2023-07-23 23:56 EDT   2023-07-24 12:56 JST
2023-07-24 00:56 EDT   2023-07-24 13:56 JST
2023-07-24 01:56 EDT   2023-07-24 14:56 JST
```

毎時 0分で指定したければ引数に0を追加します。

```
$ tzdiff America/New_York 0
New_York
2023-07-23 02:00 EDT   2023-07-23 15:00 JST
2023-07-23 03:00 EDT   2023-07-23 16:00 JST
2023-07-23 04:00 EDT   2023-07-23 17:00 JST
2023-07-23 05:00 EDT   2023-07-23 18:00 JST
2023-07-23 06:00 EDT   2023-07-23 19:00 JST
2023-07-23 07:00 EDT   2023-07-23 20:00 JST
2023-07-23 08:00 EDT   2023-07-23 21:00 JST
2023-07-23 09:00 EDT   2023-07-23 22:00 JST
2023-07-23 10:00 EDT   2023-07-23 23:00 JST
2023-07-23 11:00 EDT   2023-07-24 00:00 JST
```

また`tzdiff America/New_York 24 0`のように毎時 0分で 24時間分という指定もでき、一目瞭然で現地時間の9:00は日本時間の何時？というのがわかります(この程度であれば考えるより表示したものから探すほうが早い)。


都市の指定は、UNIX系OSで /usr/share/zoneinfoにあるTime Zone Databaseをそのまま利用しています。そのため"UTC", "Etc/GMT+9" なども指定できます。

tzdiffではちょっとしたコンプリーションもどきを実装して、使い勝手をよくしています。
次の例のように都市名の指定ではフルにスペルを指定する必要はありません。

```
$ tzdiff Lon  ( ここで最後は Tab ではなく Return つまり引数に "Lon" を指定する。)
Arctic/Longyearbyen     Europe/London
```

都市名が 「Lon」で始まる都市の一覧が表示されるので、「Lond」まで指定すれば都市が確定します。

```
$ tzdiff Lond
London
2023-07-19 15:01 BST   2023-07-19 23:01 JST
2023-07-19 16:01 BST   2023-07-20 00:01 JST
2023-07-19 17:01 BST   2023-07-20 01:01 JST
2023-07-19 18:01 BST   2023-07-20 02:01 JST
2023-07-19 19:01 BST   2023-07-20 03:01 JST
2023-07-19 20:01 BST   2023-07-20 04:01 JST
2023-07-19 21:01 BST   2023-07-20 05:01 JST
2023-07-19 22:01 BST   2023-07-20 06:01 JST
2023-07-19 23:01 BST   2023-07-20 07:01 JST
2023-07-20 00:01 BST   2023-07-20 08:01 JST
```

シェルのコマンドライン編集機能と組み合わせるわけで、このように操作するのはやりやすいと思います。

また、開始時刻を-tオプションで指定できます。次の例は、2023年のアメリカ東海岸での夏時間から標準時間に切り替わる時間帯を表示しています。

```
$ tzdiff -t 2023-11-05T10:00 New_
New_York
2023-11-04 21:00 EDT   2023-11-05 10:00 JST
2023-11-04 22:00 EDT   2023-11-05 11:00 JST
2023-11-04 23:00 EDT   2023-11-05 12:00 JST
2023-11-05 00:00 EDT   2023-11-05 13:00 JST
2023-11-05 01:00 EDT   2023-11-05 14:00 JST
2023-11-05 01:00 EST   2023-11-05 15:00 JST
2023-11-05 02:00 EST   2023-11-05 16:00 JST
2023-11-05 03:00 EST   2023-11-05 17:00 JST
2023-11-05 04:00 EST   2023-11-05 18:00 JST
2023-11-05 05:00 EST   2023-11-05 19:00 JST
```

さらに同時に複数の都市も指定できます。ワールドワイドなテレカンの時間調整にはもってこいでしょう。

```
$ tzdiff  Los New_ Brus 0
Los_Angeles            New_York               Brussels
2023-07-19 07:00 PDT   2023-07-19 10:00 EDT   2023-07-19 16:00 CEST   2023-07-19 23:00 JST
2023-07-19 08:00 PDT   2023-07-19 11:00 EDT   2023-07-19 17:00 CEST   2023-07-20 00:00 JST
2023-07-19 09:00 PDT   2023-07-19 12:00 EDT   2023-07-19 18:00 CEST   2023-07-20 01:00 JST
2023-07-19 10:00 PDT   2023-07-19 13:00 EDT   2023-07-19 19:00 CEST   2023-07-20 02:00 JST
2023-07-19 11:00 PDT   2023-07-19 14:00 EDT   2023-07-19 20:00 CEST   2023-07-20 03:00 JST
2023-07-19 12:00 PDT   2023-07-19 15:00 EDT   2023-07-19 21:00 CEST   2023-07-20 04:00 JST
2023-07-19 13:00 PDT   2023-07-19 16:00 EDT   2023-07-19 22:00 CEST   2023-07-20 05:00 JST
2023-07-19 14:00 PDT   2023-07-19 17:00 EDT   2023-07-19 23:00 CEST   2023-07-20 06:00 JST
2023-07-19 15:00 PDT   2023-07-19 18:00 EDT   2023-07-20 00:00 CEST   2023-07-20 07:00 JST
2023-07-19 16:00 PDT   2023-07-19 19:00 EDT   2023-07-20 01:00 CEST   2023-07-20 08:00 JST
```

一応簡単なヘルプもあります。

```
$ tzdiff -h
Usage: tzdiff [-0lvHN] [-n count] [-f format] [-t start] timezone [timezone ...] [count] [0]
    -0:  round down to hour
    -n:  max hours (default: 10)
    -l:  display full timezone name, only city in default
    -f:  output format (using '+output_fmt' of 'date' command)
    -t:  specify the start time (YYYY-mm-ddTHH:MM[Z], etc...)
    -v:  show version.
    -H:  scripting mode (with timezone name)
    -HH: scripting mode (without timezone name)
    -N:  without localtime
```

# 動作環境

tzdiffそのものはシェルスクリプトであり、FreeBSD / NetBSD / OpenBSD / macOS / Debian / Ubuntu / CentOS での動作を確認しています。FreeBSDではpkg、NetBSD等ではpkgsrc、DebianやUbuntuではaptでインストールできます。またmacOSではMacPortsやHomeBrewでもインストールできます。

UNIX系のOSであれば大部分のもので問題無く動作すると思いますが、dateコマンドのオプションの違いで動かないものがあるかもしれません。その場合はGNUのdateコマンドを用意すれば動くはずです。Windowsの場合は、Windows Subsystem for Linuxで利用できます。

# おわりに

tzdiffの最初の版であるtimediffを作ったのは2016年7月頃なので、すでに7年ほど経過しています。その間細かい修正を行い、使い勝手の向上と機能強化を行ってきました。現在では当初想像できない程の方に使ってもらえているようで、思わぬところからgithubのリポジトリへのプルリクやissueをもらえるようになりました。

- 2023-07-19追記：tzdiffのバージョン1.2に合わせて実行例を更新。
- 2023-07-23追記：さらに文章を調整