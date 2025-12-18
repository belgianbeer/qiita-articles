<!-- https://qiita.com/belgianbeer/items/156251d0675d456c2207 -->
# USBメモリで ZFS！

## ZFSプールの実験にはUSBメモリが最適

時々ZFSを勉強するにはどうすればいいの？と聞かれることがあるのですが、おすすめはUSBメモリで試すことです。

ZFSによるRAIDを構成する場合複数のストレージデバイスを用意する必要がありますが、実験のためにSSDやHDDでRAIDを構成できる数をそろえるのは大変です。USBメモリであればUSBハブと組み合わせることで複数のデバイスを試せてコストとスペースの点で有利です。さらにUSB接続であれば稼働中にいきなり抜く等の乱暴な試験も簡単に実現でき、修復等のトラブル対処を含めた管理方法を習得するにはある意味うってつけと言えます。最近あまり見かけなくなりましたが、LED付きのUSBメモリであればアクセスの様子も目で見て確認できて楽しいです🙂。

ところでZFSにUSBメモリを使うという元々のアイデアは、過去に「やっぱり Sun がスキ！」というサン・マイクロシステムズ社のSolarisの機能などを紹介するブログで、本記事と同じ「USBメモリで ZFS ！」のタイトルで投稿されたものです。ここでは当時の記事を参考に、FreeBSDとLinuxでの操作を紹介します。

> 「やっぱり Sun がスキ！」には2007年5月に投稿された記事で、サイトはすでに閉鎖されていますが当時の記事は[インターネットアーカイブ](https://web.archive.org/web/20071013152120/http://blogs.sun.com/yappri/entry/usb_zfs)で見られます。オリジナルの投稿では写真も掲載されていたのですが、残念なことにインターネットアーカイブでは残っていないようです[^usbhdd]。

[^usbhdd]:実際に自分で試したのは記事を見てから数年後の話です。この時は十分な数のUSB HDDがあったので、USBメモリでは無く[USB HDDで実験](https://www.facebook.com/photo?fbid=685848231475753&set=a.148549968538918)しました。

## USBメモリ 4個 と USBハブの準備

実験に使うUSBメモリですが、少し前に100円ショップでUSBのメモリカードリーダを入手していました。使ってみるとアクセスLED付きで便利だったので、今回3個追加購入して手元にあったSDカードやmicroSDカード(いずれも2GB)と組み合わせてUSBメモリにしました。4個あればRAID 6やRAID 10(イチゼロ)も試せるので、実験にはもってこいとなります[^usbid]。

[^usbid]:100円のショップのUSBデバイスではよくある話なのですが、USBデバイスの識別IDが同じあることが多く(同じでも実害は殆ど無い)、ここで使ったメモリカードリーダーも識別IDは全て同じでした。

USBハブは手持ちのものを利用しました。この形だとうまい具合にメモリカードリーダが相互に干渉せず配置できます。

実際にFreeBSDやLinuxの機器に接続すると、次のアニメーションのようにデバイスを1個づつ認識するのがわかります。

![DSC07194.GIF](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/130638/cac6d3a9-41e0-403a-8f60-cc7732a14f8b.gif)

## USBメモリの初期化

新品のUSBメモリのパーティションは容量にもよりますが多くはMBR(DOS)形式で全体をFAT32あるいはexFATでフォーマットされています。MBR形式のパーティションをそのまま使うこともできますが、障害時のトラブルシュートを簡単にするために(後述)GPTでパーティションを作り直し、各パーティションにラベル(名前)をつけてusb0～usb3で利用します。

### FreeBSDでのUSBメモリの初期化

FreeBSDの場合、通常USBメモリは /dev/da0 ～ /dev/da3 として認識されるはずです。もしすでにda0やda1のデバイスが存在するなら、番号はその後ろにずれます。ここではda0からda3までで認識されたとして説明します。

FreeBSDでのUSBメモリの初期化をda0に対して行う場合はrootユーザーで次のコマンドを実行します。

```console
# gpart destroy -F da0                  # (1) 既存のパーティションテーブルのクリア
# gpart create -s gpt da0               # (2) gpt形式のパーティションの初期化
# gpart add -t freebsd-zfs -l usb0 da0  # (3) ZFS用パーティションの作成 ラベル名は「usb0」
# zpool labelclear -f da0p1             # (4) ZFSのプールラベルの消去 (初めて使うUSBメモリの場合は不要)
# gpart show da0                        # (5) 作ったパーティションの確認
=>     40  3987376  da0  GPT  (1.9G)
       40  3987376    1  freebsd-zfs  (1.9G)

#
```

ここではda0のmicroSDカードを初期化しています。続いてda1のSDカードを同様に初期化したところ同じ2GBなのに実容量に差があり、今回使ったものではmicroSDカードのほうが容量が大きいようです。

```console
# gpart show da1                        # da1のSDカードのパーティションの確認
=>     40  3862448  da1  GPT  (1.8G)
       40  3862448    1  freebsd-zfs  (1.8G)

#
```

容量が違っていてもZFSでRAIDを構成できますが、構成時に同容量でないとミラーやRAIDZでは次のように注意のメッセージが表示されることがあります。

```console
# zpool create upool mirror ...
invalid vdev specification
use '-f' to override the following errors:
raidz contains devices of different sizes
#
```

メッセージの通り`-f`オプションで強制すれば回避でき、容量が余計な分は使われないだけです。ここでは容量の小さいSDカードに合わせるため、microSDカードのパーティションサイズを調整して`gpart add`に`-s`オプションで1884MBに制限します。同じUSBメモリを4個そろえられればこのようなサイズ調整は不要です。なおRAID 0で構成する場合は容量が違っていてもエラーにならず全容量が利用できます。

4個のUSBメモリを初期化する必要があるので、手作業でのミスを防ぐために次のシェルスクリプトを作って実行しました。

```console
# cat usbmem-init-freebsd     # シェルスクリプトの内容の表示
#! /bin/sh

for i in 0 # 1 2 3
do
        gpart destroy -F da$i
        sleep 1
        gpart create -s gpt da$i
        sleep 1
        gpart add -t freebsd-zfs -a 1M -s 1884M -l usb$i da$i
        # sleep 1
        # zpool labelclear -f da${i}p1
done
# ./usbmem-init-freebsd     # シェルスクリプトの実行
da0 destroyed
da0 created
------ (中略) ------
da3 created
da3p1 added
# ls /dev/gpt/usb?     # 用意したパーティションの存在を確認する
/dev/gpt/usb0   /dev/gpt/usb1   /dev/gpt/usb2   /dev/gpt/usb3
#
```

ここではda0～da3に対して、ラベル名usb0～usb3を割り当てています。後述するLinuxの場合と物理的に同じ場所にパーティションが作られるように、`-a`オプションを指定して1Mバイトでアラインを調整しています。また各コマンドの間で`sleep 1`を実行しているのは、実際のUSBメモリへの書き込みには時間がかかるためシェルスクリプトで連続して実行すると、前の処理が終わる前に次のコマンドを実行しようとしてトラブルになるのを回避しています。

### LinuxでZFSを利用する

LinuxではFreeBSDと違って一部のディストリビューションを除いて標準ではZFSが含まれていません。LinuxでZFSを利用する場合は、[ZFS on Linux](https://zfsonlinux.org/)の[Getting Started](https://openzfs.github.io/openzfs-docs/Getting%20Started/index.html)にある手順にしたがって、必要なパッケージを追加します。

### LinuxでのUSBメモリの初期化

Linuxの場合、USBメモリは /dev/sdb ～ /dev/sde として認識されるはずなので(すでにsdbやsdcのデバイスが存在するならアルファベット順にずれる)、それを前提に説明します。

LinuxでsdbにあるUSBメモリの初期化を行う場合はrootユーザーで次のコマンドを実行します。

```console
# wipefs -a /dev/sdb                        # (1) 既存のパーティションテーブルのクリア
# echo 'label: gpt' | sfdisk --quiet /dev/sdb    # (2) gpt形式のパーティションの初期化
# echo "type=6A898CC3-1DD2-11B2-99A6-080020736631, name=usb0, size=1884M" | sfdisk --quiet /dev/sdb  # (3) ZFS用パーティションの作成 ラベル名は「usb0」
# zpool labelclear -f /dev/sdb1             # (4) ZFSのプールラベルの消去 (初めて使うUSBメモリの場合は不要)
# sfdisk --list --quiet /dev/sdb            # (5) 作ったパーティションの確認
Device     Start     End Sectors  Size Type
/dev/sdb1   2048 3860479 3858432  1.8G Solaris /usr & Apple ZFS
#
```

パーティションの作成に`fdisk`では無く`sfdisk`を使っていますが、シェルスクリプト等のバッチ的な処理ではfdiskよりもsfdiskのほうが使いやすいからです。sfdiskをあまりご存じ無い方は「[FreeBSDユーザーがsfdiskを使いこなす](https://qiita.com/belgianbeer/items/cc0093ea6ca06a3a4f6c)」を参照してください。

実際にはFreeBSDの場合と同様にシェルスクリプトを作って実行しました。またパーティションラベルにはFreeBSDと同様にusb0～usb3を割り当てています。

```console
# cat usbmem-init-linux
#!/bin/sh

j=0
for i in b c d e
do
    wipefs --quiet --all /dev/sd${i}
    sleep 1
    sfdisk --quiet /dev/sd${i} <<- EOT
        label: gpt
        type=6A898CC3-1DD2-11B2-99A6-080020736631, name=usb${j}, size=1884M
    EOT
    # sleep 1
    # zpool labelclear -f /dev/sd${i}1
    j=$((j + 1))
done
# ./usbmem-init-linux
# ls /dev/disk/by-partlabel/usb?     # 作成したパーティションの存在を確認
/dev/disk/by-partlabel/usb0  /dev/disk/by-partlabel/usb2
/dev/disk/by-partlabel/usb1  /dev/disk/by-partlabel/usb3
#
```

これでテスト用のストレージデバイス(ここではUSBメモリ)の準備ができました。以降はこのデバイスを使ってZFSプールを構成します。

## RAIDZ を構成する

ここでは「やっぱり Sun がスキ！」の記事に従ってRAIDZのプールを作成してみます。詳細は割愛しますがRAIDZは一般的なRAID 5に相当し冗長性を持たせることでストレージのうち1台が壊れても動作しつづけることができるRAID構成です。

### FreeBSDでRAIDZをセットアップ

FreeBSDでRAIDZのZFSプールを作る場合は、次のように操作します。

```console
# zpool create -O atime=off -O compression=lz4 upool raidz gpt/usb0 gpt/usb1 gpt/usb2 gpt/usb3
#
```

ここではプール名としてupoolを指定していますが、プールはそのまま一つのZFSファイルシステムとなります。
`-O`はupoolのZFSファイルシステムとしてのプロパティを設定するオプションです。

- atime=off: ZFSはCopy on Writeで動作するため、atime(ファイルのアクセス時間)を記録するとファイルを読む際もatimeの更新のためにストレージへの書き込みが発生してパフォーマンス面で不利になるので、atimeの記録を禁止しています。atimeを記録しない場合、稀にで正しい動作を期待できなくなることもありますが[^tcsh]、通常は問題ありません。
- compression=lz4: ZFSのlz4による圧縮機能を有効にしています。

[^tcsh]:cshでは新着メールの確認にatimeを利用しているようです。

プール名に続いてデバイス名を指定します。通常デバイス名は「/dev/…」で始まる形で指定しますが、zpoolでは「/dev/」を省略できるので「gpt/usb0」のように指定します。

作ったプールを確認するには、`zpool list`コマンドを使います。

```console
# zpool list upool
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
upool     7G   197K  7.00G        -         -     0%     0%  1.00x    ONLINE  -
```

-vオプションを指定するとプールの構成まで含めて表示されます。

```console
# zpool list -v upool
NAME           SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
upool            7G   197K  7.00G        -         -     0%     0%  1.00x    ONLINE  -
  raidz1-0       7G   197K  7.00G        -         -     0%  0.00%      -    ONLINE
    gpt/usb0  1.84G      -      -        -         -      -      -      -    ONLINE
    gpt/usb1  1.84G      -      -        -         -      -      -      -    ONLINE
    gpt/usb2  1.84G      -      -        -         -      -      -      -    ONLINE
    gpt/usb3  1.84G      -      -        -         -      -      -      -    ONLINE
#
```

実際の総容量はzpoolでは考慮されないので、zfsコマンドで確認します。

```console
# zfs list upool
NAME    USED  AVAIL  REFER  MOUNTPOINT
upool   147K  5.07G  32.9K  /upool
#
```

RAIDZではストレージ1台分がパリティとして利用されるため、全容量の合計から1台分少ない容量が実際に利用できる容量となります。

### LinuxでRAIDZをセットアップ

Linuxの場合はデバイス名の形式の違いを除けばFreeBSDと同様です。Linuxではラベル名が/dev/disk/by-partlabel/usb0等になるため、zpoolでは次のように指定します。

```console
# zpool create -O atime=off -O compression=lz4 upool raidz disk/by-partlabel/usb0 disk/by-partlabel/usb1 disk/by-partlabel/usb2 disk/by-partlabel/usb3
#
```

プールを確認すると次のようになります。

```console
# zpool list upool
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
upool     7G   254K  7.00G        -         -     0%     0%  1.00x    ONLINE  -
# 
```

次はzpool list -v で詳細を確認しているところです。

```console
# zpool list -v upool
NAME         SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
upool          7G   203K  7.00G        -         -     0%     0%  1.00x    ONLINE  -
  raidz1-0     7G   203K  7.00G        -         -     0%  0.00%      -    ONLINE
    usb0    1.84G      -      -        -         -      -      -      -    ONLINE
    usb1    1.84G      -      -        -         -      -      -      -    ONLINE
    usb2    1.84G      -      -        -         -      -      -      -    ONLINE
    usb3    1.84G      -      -        -         -      -      -      -    ONLINE
#
```

zfs listで見ると次の通り。

```console
# zfs list upool
NAME    USED  AVAIL  REFER  MOUNTPOINT
upool   147K  5.07G  32.9K  /upool
#
```

## 実際に使ってみる

プールを作ると、プール名はそのまま一つのファイルシステムとなりデフォルトではルート直下にマウントされます。この場合プールは`/upool`にマウントされているので、ファイルを書き込んでみます。

```console
# cp -r /usr/bin /upool
```

このようにLEDが光って、書き込みの様子がわかります。

![DSC07199.GIF](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/130638/20258830-ccf2-459c-8937-a41ac6ae0c9f.gif)

## RAIDを構成するストレージは必ずGPTでパーティションにラベル名をつけて使う

最初にも少し触れましたが、重要なことなのであらためてもう一度書きます。FreeBSDでもLinuxでもZFSを構成する場合にパーティションを作らずストレージそのものを示すデバイスを使うこともできます。

FreeBSD:

```console
# zpool create -O atime=off -O compression=lz4 -f upool da0 da1 da2 da3
#
```

Linux:

```console
# zpool create -O atime=off -O compression=lz4 -f upool sdb sdc sdd sde
#
```

しかしストレージそのもののデバイス名を直接使用すると、**障害時に故障したストレージデバイスを特定することが困難**になります。GPTでパーティションにラベル名があれば、故障したストレージデバイスを容易に特定できるようになります。詳細は「[RAIDの障害時にストレージを取り違えないために気をつけること](https://qiita.com/belgianbeer/items/3b927310c206c3d9f78c)」の記事を参照してください。
