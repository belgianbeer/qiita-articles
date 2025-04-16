# USBメモリで ZFS ！

## ZFSのプール管理にはUSBメモリが最適

時々ZFSを勉強するにはどうすればいいの？と聞かれることがあるのですが、筆者のおすすめはUSBメモリで試すことです。

ZFSによるRAIDを構成する場合複数のストレージデバイスを用意する必要がありますが、SSDやHDDでRAID構成を試せる台数をそろえるのは大変です。USBメモリであればUSBハブと組み合わせることで複数のデバイスを試せて価格とスペースの点で有利で、さらに稼働中にいきなり抜く等の乱暴な試験も実現でき、ZFSでのRAID構成を習得するにはある意味うってつけと言えます。特にLED付きのUSBメモリであればアクセスの様子も目視で確認できるので見ていても楽しいです。🙂

ここではUSBメモリを組み合わせて、ZFSによる様々なRAIDを構成毎の特徴を含めて設定してみます。

なおZFSにUSBメモリを使うという元々のアイデアは、過去にサン・マイクロシステムズの「やっぱり Sun がスキ！」というSolarisの機能などを紹介するブログで、本記事と同じく「USBメモリで ZFS ！」のタイトルで投稿されたものです[^yappari]。

[^yappari]:2007年5月に投稿されたものでサイトはすでに閉鎖されていますが、当時の記事は[インターネットアーカイブ](https://web.archive.org/web/20071013152120/http://blogs.sun.com/yappri/entry/usb_zfs)で見ることができます(投稿時は写真もあったのですが、インターネットアーカーイブには残って無いようです)。当時この記事を見て実際に自分で試したのはその数年後ですが、USB HDDが十分な数あったのでUSBメモリでは無く[USB HDDで実験](https://www.facebook.com/photo?fbid=685848231475753&set=a.148549968538918)しました。

## USBメモリ 4個 と USBハブの準備

実験に使うUSBメモリですが、筆者の場合少し前に100円ショップでUSBのメモリカードリーダを入手していました。使ってみるとアクセスLED付きで便利だったので、今回3個追加購入して手元にあったSDカードやmicroSDカードと組み合わせてUSBメモリにします。4個あればRAID 6やRAID 10(イチゼロ)も試せるので、実験にはもってこいとなります。

USBハブは手持ちのものを利用しました。この形だとメモリカードリーダが相互に干渉せずに配置できます(笑)。筆者の場合はmicroSDカード、SDカード共2GBのものを使っています。

実際にPCに接続すると、次のアニメーションのようにデバイスを1個づつ認識するのがわかります。

![DSC07194.GIF](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/130638/cac6d3a9-41e0-403a-8f60-cc7732a14f8b.gif)

## USBメモリの初期化

新品のUSBメモリのパーティションは容量にもよりますが多くはMBR(DOS)形式で全体をFAT32あるいはexFATでフォーマットされた状態となっています。MBR形式のパーティションをそのまま使うこともできますが、柔軟な管理のためにGPTでパーティションを作り直します。GPTであれば、それぞれのパーティションにラベル(名前)をつけられるので、usb0～usb3を割り当てます。

### FreeBSDでのUSBメモリの初期化

FreeBSDの場合、通常USBメモリは /dev/da0 ～ /dev/da3 として認識されるはずです。もしすでにda0やda1のデバイスが存在するなら、番号はその後ろにずれます。ここではda0からda3までで認識されたとして説明します。

FreeBSDでのUSBメモリの初期化をda0に対して行う場合は次のようになります。

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

ここではda0のmicroSDカードを初期化していますが、da1のSDカードに対して同様に初期化したところ公称は同容量ですが実容量に少し差がありました。

```console
# gpart show da1                        # da1のSDカードのパーティションの確認
=>     40  3862448  da1  GPT  (1.8G)
       40  3862448    1  freebsd-zfs  (1.8G)

#
```

RAID Z(後述)を構成する場合は同容量でない場合に注意が表示されるので、容量の小さいSDカードに合わせてmicroSDカードのパーティションサイズを少し減らすため、`gpart add`に`-s`オプションを使ってSDカードの1884Mに制限します。

実際には4個のUSBメモリを初期化するため、次のようにシェルスクリプトを作って実行しました。

```console
# cat usbmem-init-freebsd     # シェルスクリプトの内容の表示
#! /bin/sh
[ $(uname -s) = FreeBSD ] || exit

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

この通り、da0～da3に対して、ラベル名usb0～usb3を割り当てています。また各コマンドの間に`sleep 1`を入れているのは、このような操作ではコマンドの実行はすぐに終了するのにUSBメモリへの書き込みに時間がかかることがあって、シェルスクリプトで連続して実行するとトラブルになることがあるの回避しています。さらに後述するLinuxの場合と物理的に同じ場所にパーティションが作られるように、gpartのパラメーターを調整しています。

### LinuxでのUSBメモリの初期化

Linuxの場合、通常USBメモリは /dev/sdb ～ /dev/sde として認識されるはずなので(すでにsdbやsdcのデバイスが存在するならアルファベット順にずれる)、それを前提に説明します。

LinuxでのUSBメモリの初期化をsdbに対して行う場合は次のようになります。

```console
# wipefs -a /dev/sdb                        # (1) 既存のパーティションテーブルのクリア
# echo 'label: gpt' | sfdisk --quiet /dev/sdb    # (2) gpt形式のパーティションの初期化
# echo "type=6A898CC3-1DD2-11B2-99A6-080020736631, name=usb0, size=1884M" | sfdisk  --quiet /dev/sdb  # (3) ZFS用パーティションの作成 ラベル名は「usb0」
# zpool labelclear -f /dev/sdb1             # (4) ZFSのプールラベルの消去 (初めて使うUSBメモリの場合は不要)
# sfdisk --list --quiet /dev/sdb            # (5) 作ったパーティションの確認
Device     Start     End Sectors  Size Type
/dev/sdb1   2048 3860479 3858432  1.8G Solaris /usr & Apple ZFS
#
```

パーティションの作成で`fdisk`では無く`sfdisk`を使っていますが、sfdiskをあまりご存じで無い方は「[FreeBSDユーザーがsfdiskを使いこなす](https://qiita.com/belgianbeer/items/cc0093ea6ca06a3a4f6c)」を参照してください。

実際にはFreeBSDの場合と同様にシェルスクリプトを作って実行しました。FreeBSDの場合と同様にパーティションラベルにはusb0～usb3を割り当てています。

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

## RAID 0 ストライプ

### RAID 0の特徴

最初はRAID 0、つまりストライプです。RAID 0では複数のデバイスを単純に結合することで容量を増加させるとともに、データを適当なサイズに分割して組み合わせたデバイス(ここでは4台)に分散して書き込みや読み込みを行うことで、負荷を分散する効果があります。デバイスがSSDやHDDであれば(USBメモリではあまり効果が無い)、読み書きの負荷が複数デバイスに分散されることで1台で利用する場合に比べて高速になります。

### FreeBSDでRAID 0をセットアップ

FreeBSDでRAID 0のZFSプールを作る場合は、次のように操作します。

```console
# zpool create -O atime=off -O compression=lz4 -f upool gpt/usb0 gpt/usb1 gpt/usb2 gpt/usb3
#
```

ここではプール名としてupoolを指定していますが、プールはそのまま一つのファイルシステムとなります。
`-O`はupoolのファイルシステムとしてのプロパティを設定するオプションです。

- atime=off: ZFSはCopy on Writeで動作するため、ファイルのアクセス時間を記録すると通常のファイルシステムに比べてパフォーマンス面で不利になるため、アクセス時間の記録しないように設定します。
- compression=lz4: ZFSのファイル圧縮機能を有効にしています。複数の圧縮方式が指定できますが、ここではlz4を使います。

プール名に続いてデバイス名を指定します。通常デバイス名は「/dev/…」で始まる形で指定しますが、zpoolでは「/dev/」を省略できるので「gpt/usb0」のように指定します。

出来上がったプールを確認するには、`zpool list`コマンドを使います。-vオプションを指定するとプールの構成まで含めて表示されます。

```console
# zpool list -v upool
NAME         SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
upool          7G   128K  7.00G        -         -     0%     0%  1.00x    ONLINE  -
  gpt/usb0  1.84G  39.5K  1.75G        -         -     0%  0.00%      -    ONLINE
  gpt/usb1  1.84G    35K  1.75G        -         -     0%  0.00%      -    ONLINE
  gpt/usb2  1.84G    25K  1.75G        -         -     0%  0.00%      -    ONLINE
  gpt/usb3  1.84G    28K  1.75G        -         -     0%  0.00%      -    ONLINE
#
```

### デバイス名を指定する場合の注意

デバイス名としては/dev/gpt/usb0, /dev/gpt/usb1, /dev/gpt/usb2, /dev/gpt/usb3はそれぞれ/dev/da0p1, /dev/da1p1, /dev/da2p1, /dev/da3p1なので、次のように指定することもできます。

```console
# zpool create -O atime=off -O compression=lz4 -f upool da0p1 da1p1 da2p1 da3p1
#
```

しかしこのように物理的なデバイス名を使用すると、どれかのデバイスが故障した場合に困ります。例えば/dev/da1が故障して認識できなくなったときに再起動すると、/dev/da2だったものが/dev/da1, /dev/da3だったものが/dev/da2と繰り上がるため、本当に故障したものがどれだかわからなくなります。ラベル名で管理していれば、無くなったラベル名からどのデバイスであったかが特定できます。

## LinuxでRAID 0セットアップ

Linuxの場合はデバイス名の形式の違いを除けばFreeBSDと同様です。Linuxではラベル名が/dev/disk/by-partlabel/usb0等になるため、zpoolでは次のようにしていします。

```console
# zpool create -O atime=off -O compression=lz4 -f upool disk/by-partlabel/usb0 disk/by-partlabel/usb1 disk/by-partlabel/usb2 disk/by-partlabel/usb3
# zpool list -v upool
NAME        SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
upool         7G   122K  7.00G        -         -     0%     0%  1.00x    ONLINE  -
  usb0     1.84G  44.5K  1.75G        -         -     0%  0.00%      -    ONLINE
  usb1     1.84G  32.5K  1.75G        -         -     0%  0.00%      -    ONLINE
  usb2     1.84G      0  1.75G        -         -     0%  0.00%      -    ONLINE
  usb3     1.84G  44.5K  1.75G        -         -     0%  0.00%      -    ONLINE
#
```




![DSC07199.GIF](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/130638/20258830-ccf2-459c-8937-a41ac6ae0c9f.gif)


```console
# zpool create -O atime=off -O compression=lz4 -f upool da0p1 da1p1 da2p1 da3p1  # FreeBSDの場合

# zpool create -O atime=off -O compression=lz4 -f upool sdb1 sdc1 sdd1 sde1      # Linuxの場合
```

