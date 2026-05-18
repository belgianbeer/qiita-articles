<!-- https://qiita.com/belgianbeer/items/cdab7919ec3555ffd339 -->

# SambaとZFSでMacのTime Machineバックアップを実現する

## はじめに

MacユーザーであればTime Machineでのバックアップが必須であることは言うまでもない。Time Machineのバックアップ先としてはUSB接続のストレージやNAS等もあるが、Sambaサーバーを運用しているのであればそれを使うのも一手である。特にFreeBSDとZFSでSambaサーバー運用しているのであればZFSの恩恵も受けられる。当記事は実際にこれを実現するためにSambaの設定を試して確認した結果である。

> ご覧の通り当記事はFreeBSD Advent Calendar 2020に投稿したものであるが、2026年2月、FreeBSD 15の導入時にSambaを4.23にバージョンアップして改めて設定を見直した。さらにmacOS SeqauoiaとmacOS Tahoeとで動作確認を行い[^tahoe]、全面的に書き換えた。当初の記事には動かなかった設定例も記述してあったが、それらは削除した。

[^tahoe]:2026年3月現在macOS TahoeのTime Machineにはバックアップファイル作成時にUnicodeのNFC/NFD問題があり、初期バックアップのみ拳固を英語にする等の回避策をとる必要がある。

ここで目的とする設定は次の通りである。

- Time Machineバックアップが正しく動作する
- ファイル共有でMacのリソースフォークをZFSの拡張属性に保存する(“._*” が作成されない)

## SambaでMacのファイル共有とTime Machineの設定

SambaではVFSモジュールの追加で特定のファイルシステムやプロトコルに対応できるようになっている。AFP用のサービスのモジュールはvfs_fruitで、Macとファイル共有を行うためにはこのモジュールの導入が必要となる。またstreams_xattrは、streamの内容をファイルシステムのPOSIIX拡張属性に保存するモジュールである。

動作した設定例は以下の通りで、Macには関係のない一般的な設定は省略してある。またこの設定はFreeBSD 15.0-RELEASEのSamba 4.23、macOS TahoeとSequoiaの組み合わせでの動作を確認している。

### global セクション

vfs_fruitを含め必要なモジュールとMacでのファイル共有に必要な設定を追加する。

```ini
[global]
        vfs objects = catia fruit streams_xattr
        fruit:resource = stream
        fruit:metadata = stream
```

macOSではファイル名にWindowsで通常禁止されている文字(?, <, >, *, | など)を使用できてしまうので、トラブルを避けるためにファイル名を適切にマッピングしてくれるcatiaというVFSモジュールを追加してある。

### 個別の共有フォルダ

特別な設定は行う必要は行わなくてもMacから問題無くファイルアクセスができる。

```ini
[share]
        path = /data/share
        # Mac用の特別な設定は追加する必要は無い
```

### Time Machine用のバックアップフォルダ

Time Machine用の共有フォルダであることを設定するために`fruit:time machine = yes`を追加するだけでよい。

```ini
[TimeMachine]
        path = /data/TimeMachine
        fruit:time machine = yes
```

## Time Machineを使うためのZFSの構成

上記の設定で、/data/share と /backup/TimeMachine は ZFSでは別々のファイルシステムとして設定している。つまり`zfs list`では次のように見える

```console
$ zfs list -o name -r zroot/data
NAME
zroot/data
zroot/data/share
zroot/data/TimeMachine
$
```

このように独立したファイルシステムにしておけば、ZFSのプロパティを設定することでTimeMachine用ディスク容量の制限やその値の変更を簡単に行える。例えば1TBに制限するのであれば次のようにquotaを設定する。

```console
$ zfs set quota=1T zroot/data/TimeMachine
$
```

もちろん容量制限はSamba側でも行えるが、ZFSプロパティであればSambaサーバーが設定の読み直す必要もなく即座に設定変更できる。

## おわりに

実は今回記載した設定は、本記事を最初に投稿した時点では動作せず「ダメだった例」として記載していたものである。当時これで動作するはずなのに変だなぁと思いながらテストを繰り返した記憶がある。
