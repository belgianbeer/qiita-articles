# USBメモリで ZFS ！

## ZFSの習得にはUSBメモリが最適

時々ZFSを勉強するにはどうすればいいの？と聞かれることがあるのですが、筆者のおすすめはUSBメモリで試すことです。

ZFSによるRAID構成を実験する場合複数のストレージデバイスを用意する必要がありますが、SSDやHDDでRAID構成を試せる台数をそろえるのは大変です。USBメモリであればUSBハブと組み合わせることで複数のデバイスを試せて価格とスペースの点で有利で、さらに稼働中にいきなり抜く等の乱暴な試験も実現でき、ZFSでのRAID構成を習得するにはある意味うってつけと言えます。特にLED付きのUSBメモリであればアクセスの様子も目視で確認できるので見ていても楽しいです。🙂

ここではUSBメモリを組み合わせて、ZFSによる様々なRAIDを構成毎の特徴を含めて設定してみます。

なおZFSにUSBメモリを使うという元ネタは、過去にサン・マイクロシステムズの「やっぱり Sun がスキ！」というSolarisの機能などを紹介するブログで、本記事と同じく「USBメモリで ZFS ！」のタイトルで投稿されたものです[^yappari]。

[^yappari]:2007年5月に投稿されたものでサイトはすでに閉鎖されていますが、当時の記事は[インターネットアーカイブで確認](https://web.archive.org/web/20071013152120/http://blogs.sun.com/yappri/entry/usb_zfs)できます。この記事を見て数年後に実際に自分で試したときは、なぜかUSB HDDが複数あったため、USBメモリでは無く[USB HDDで実験](https://www.facebook.com/photo?fbid=685848231475753&set=a.148549968538918)しました。

## USBメモリ 4個 と USBハブの準備

実験に使うUSBメモリですが、筆者の場合少し前に100円ショップでUSBのメモリカードリーダを入手していました。使ってみるとアクセスLED付きで便利だったので、今回3個追加購入して手元にあったSDカードやmicroSDカードと組み合わせてUSBメモリにします。4個あればRAID 6やRAID 10(イチゼロ)も試せるので、実験にはもってこいとなります。

USBハブは手持ちのものを利用しました。この形だとメモリカードリーダが相互に干渉せずに配置できます(笑)。筆者の場合はmicroSDカード、SDカード共2GBのものを使っています。

実際にPCに接続すると、次のアニメーションのようにデバイスを1個づつ認識するのがわかります。

![DSC07194.GIF](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/130638/cac6d3a9-41e0-403a-8f60-cc7732a14f8b.gif)

## USBメモリの初期化

新品のUSBメモリのパーティションは容量にもよりますが多くはMBR(DOS)形式で全体をFAT32あるいはexFATでフォーマットされた状態となっています。パーティションをそのまま使うことも出来ますが、柔軟な管理のためにGPTでパーティションを作り直します。

### FreeBSDでのUSBメモリの初期化

FreeBSDでのUSBメモリの初期化をda0に対して行う場合は次のようになります。

```console
$ gpart destroy -F da0                  # (1) 既存のパーティションテーブルのクリア
$ gpart create -s gpt da0               # (2) gpt形式のパーティションの初期化
$ gpart add -t freebsd-zfs -l usb0 da0  # (3) ZFS用パーティションの作成 ラベル名は「usb0」
$ zpool labelclear -f da0p1             # (4) ZFSのプールラベルの消去 (初めて使うUSBメモリの場合は不要)
$ gpart show da0                        # (5) 作ったパーティションの確認
=>     40  3987376  da0  GPT  (1.9G)
       40  3987376    1  freebsd-zfs  (1.9G)

$
```

ここではda0のmicroSDカードを初期化していますが、da1のSDカードに対して同様に初期化したところ公称は同容量ですが実容量に少し差がありました。

```console
$ gpart show da1                        # da1のSDカードのパーティションの確認
=>     40  3862448  da1  GPT  (1.8G)
       40  3862448    1  freebsd-zfs  (1.8G)

$
```

このまま使えないことも無いのですが、RAID Z(後述)を構成する場合は同容量の方が良いので(容量違うと注意メッセージが表示される)、容量の小さいSDカードに合わせてmicroSDカードのパーティションを少し減らし、`gpart add`のときに`-s`オプションを使ってサイズを制限します。4個のUSBメモリを初期化するため、実際には次のシェルスクリプトを使いました。

```console
$ cat usbmem-init-freebsd     # シェルスクリプトの内容の表示
#!/bin/sh
for i in 0 1 2 3
do
    gpart destroy -F da$i
    sleep 1
    gpart create -s gpt da$i
    sleep 1
    gpart add -t freebsd-zfs -s 3862448 -l usb$i da$i
    sleep 1
    zpool labelclear -f da${i}p1
done
$ ./usbmem-init-freebsd     # シェルスクリプトの実行
da0 destroyed
da0 created
da0p1 added
failed to clear label for /dev/da0p1
da1 destroyed
da1 created
da1p1 added
failed to clear label for /dev/da1p1
da2 destroyed
da2 created
da2p1 added
failed to clear label for /dev/da2p1
da3 destroyed
da3 created
da3p1 added
failed to clear label for /dev/da3p1
$ ls /dev/gpt/usb?     # 用意したパーティションが存在するのを確認
/dev/gpt/usb0   /dev/gpt/usb1   /dev/gpt/usb2   /dev/gpt/usb3
$
```

シェルスクリプトの実行中に`failed to clear label for /dev/XXXX`のエラーメッセージが表示されていますが、`zpool labelclear`を実行する必要が無かったためメッセージで問題はありません。

各コマンドの間に`sleep 1`を入れているのは、このような操作ではコマンドの実行はすぐに終了するのにUSBメモリへの書き込みに時間がかかることがあって、シェルスクリプトでたて続けに行うとトラブルになることがあるの回避しています。

## LinuxでのUSBメモリの初期化

```console
$ cat usbmem-init-linux
#!/bin/sh
j=0
for i in b c d e
do
    wipefs -a /dev/sd${i}
    sleep 1
    echo 'label: gpt' | sfdisk /dev/sd${i}
    sleep 1
    echo "type=6A898CC3-1DD2-11B2-99A6-080020736631, name=usb${j}, size=3858432" | sfdisk /dev/sd${i}
    sleep 1
    zpool labelclear -f /dev/sd${i}1
    j=$((j + 1))
done
$ ./usbmem-init-linux
★★★ # 実行結果を入れる
$ ls /dev/disk/by-partlabel/usb?
$
```
