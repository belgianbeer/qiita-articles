<!-- https://qiita.com/belgianbeer/items/9726a3e8be624c4cedf2 -->
# USBメモリでZFS！ その2 RAIDZを壊して修復してみる

## はじめに

[USBメモリでZFS！](https://qiita.com/belgianbeer/items/156251d0675d456c2207)で、USBメモリを使ったRAIDZのストレージセットができました。RAIDZは一般的なRAIDではRAID 5に相当します。RAID 5は複数台(最低で3台)のストレージを組み合わせ、うち1台分の容量をパリティに割り当てて冗長性を持たせることで、ストレージが1台壊れても動作しつづけられるRAIDの構成方式です。

今回はUSBメモリで構成したことを利用し、プール中のUSBメモリ1個を撤去することでプールを故意に壊して修復を試してみます。ここではFreeBSDで試しますが([FreeBSDのAdvent Calender](https://qiita.com/advent-calendar/2025/freebsd)ですからw)、Linuxでもデバイス名が変わるだけで操作は同様です。

## パターン1：USBメモリを抜いてプールを故障させ、抜いたUSBメモリを戻す

### 最初の状態

まずプールの現在の状況を確認します。

```console
# zpool status upool
  pool: upool
 state: ONLINE
config:

        NAME          STATE     READ WRITE CKSUM
        upool         ONLINE       0     0     0
          raidz1-0    ONLINE       0     0     0
            gpt/usb0  ONLINE       0     0     0
            gpt/usb1  ONLINE       0     0     0
            gpt/usb2  ONLINE       0     0     0
            gpt/usb3  ONLINE       0     0     0

errors: No known data errors
# zpool list -v upool
NAME           SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
upool            7G   630M  6.38G        -         -     0%     8%  1.00x    ONLINE  -
  raidz1-0       7G   630M  6.38G        -         -     0%  8.79%      -    ONLINE
    gpt/usb0  1.84G      -      -        -         -      -      -      -    ONLINE
    gpt/usb1  1.84G      -      -        -         -      -      -      -    ONLINE
    gpt/usb2  1.84G      -      -        -         -      -      -      -    ONLINE
    gpt/usb3  1.84G      -      -        -         -      -      -      -    ONLINE
#
```

4個のUSBメモリによるRAIDZのストレージを630MBほど消費した状態です。

### USBメモリを1個抜いてみる

ここでusb1が割り当ててある2個目のUSBメモリ(/dev/da1)が故障した場合を想定して、いきなり抜いてみます。プールの状態を確認すると次のようになります。

```console
# zpool status upool
  pool: upool
 state: DEGRADED
status: One or more devices have been removed.
        Sufficient replicas exist for the pool to continue functioning in a
        degraded state.
action: Online the device using zpool online' or replace the device with
        'zpool replace'.
config:

        NAME          STATE     READ WRITE CKSUM
        upool         DEGRADED     0     0     0
          raidz1-0    DEGRADED     0     0     0
            gpt/usb0  ONLINE       0     0     0
            gpt/usb1  REMOVED      0     0     0
            gpt/usb2  ONLINE       0     0     0
            gpt/usb3  ONLINE       0     0     0

errors: No known data errors
#
```

state:が`ONLINE`から`DEGRADED`になり状態が悪化していることがわかります。またstatus:のフィールドで状況が表示され、action:に対処方法が表示されています。USBメモリを抜いたことによりデバイス個別のSTATEは`REMOVED`となっていますが、実際の故障の場合は別の状態表示になる場合もあります。

この状態でもRAIDZには冗長性があるため、ファイルシステムへのアクセスは問題無く/upool配下のファイルへの読み書きは通常通り行えます。

### 抜いたUSBメモリを差し直す

usb1は単に抜いただけですので壊れているわけではありません。ですから再びUSBハブに差し込めば戻せそうな感じです。実際に差し直して状態を確認します。

```console
# zpool status upool
  pool: upool
 state: DEGRADED
status: One or more devices have been removed.
        Sufficient replicas exist for the pool to continue functioning in a
        degraded state.
action: Online the device using zpool online' or replace the device with
        'zpool replace'.
config:

        NAME          STATE     READ WRITE CKSUM
        upool         DEGRADED     0     0     0
          raidz1-0    DEGRADED     0     0     0
            gpt/usb0  ONLINE       0     0     0
            gpt/usb1  REMOVED      0     0     0
            gpt/usb2  ONLINE       0     0     0
            gpt/usb3  ONLINE       0     0     0

errors: No known data errors
#
```

この通り差し直すだけでは何も変わりません。なぜなら差し直したUSBメモリが元のものであるかどうかを勝手に確認したりしないので、そのままではなにもおきないわけです。そこで`zpool online`を使ってusb1が復帰したことを指定すればもとに戻せます。

```console
# zpool online upool gpt/usb1
# zpool status upool
  pool: upool
 state: ONLINE
  scan: resilvered 10.5K in 00:00:07 with 0 errors on Tue Dec 23 14:00:34 2025
config:

        NAME          STATE     READ WRITE CKSUM
        upool         ONLINE       0     0     0
          raidz1-0    ONLINE       0     0     0
            gpt/usb0  ONLINE       0     0     0
            gpt/usb1  ONLINE       0     0     0
            gpt/usb2  ONLINE       0     0     0
            gpt/usb3  ONLINE       0     0     0

errors: No known data errors
#
```

この通り、state:が`ONLINE`となり正常な状態に戻りました。

scan:には`resilvered`というあまり見かけない表現が使われていますが、resilverは銀の装飾品等を磨く意味です。ZFSではRAIDの修復にこの表現を使っています。usb1を差し直し`zpool online`によってresilverに7秒かかったことがわかります。

## パターン2：USBメモリを抜いてプールをDEGRADEし、新たなUSBメモリで修復する

今度はUSBメモリを抜くまでは前と同じですが、同じUSBメモリを戻すのでは無く、別のUSBメモリを使って修復してみます。新しいUSBメモリはusb4としてあらかじめ用意しています。

### USBメモリ(da0)を抜いてみる

まずusb0(da0)を抜いてみます。

```console
# zpool status upool
  pool: upool
 state: DEGRADED
status: One or more devices have been removed.
        Sufficient replicas exist for the pool to continue functioning in a
        degraded state.
action: Online the device using zpool online' or replace the device with
        'zpool replace'.
  scan: resilvered 10.5K in 00:00:07 with 0 errors on Tue Dec 23 14:00:34 2025
config:

        NAME          STATE     READ WRITE CKSUM
        upool         DEGRADED     0     0     0
          raidz1-0    DEGRADED     0     0     0
            gpt/usb0  REMOVED      0     0     0
            gpt/usb1  ONLINE       0     0     0
            gpt/usb2  ONLINE       0     0     0
            gpt/usb3  ONLINE       0     0     0

errors: No known data errors
#
```

結果を見るとusb0がREMOVEDとなり、status:に現在の状況が、action:に管理者が取るべきアドバイスが表示されています。

### 新たなUSBメモリを差してみる

ここで用意したusb4を差してみます。抜いたusb0はda0であったため、新たに差したUSBメモリは空きとなったda0に割り当てられます。

```console
# gpart show -l da0
=>     40  7710640  da0  GPT  (3.7G)
       40     2008       - free -  (1.0M)
     2048  3858432    1  usb4  (1.8G)
  3860480  3850200       - free -  (1.8G)

# 
```

もちろん新たに追加したusb4は新規のデバイスであるため、`zpool online`による復帰はできません。


```console
# zpool online upool gpt/usb4
couldn't find device "gpt/usb4" in pool "upool"
#
```

### usb0をusb4で置き換える

`zpool replace`を使い、usb0をusb4に置き換えることによってプールを修復します。

```console
# zpool replace upool gpt/usb0 gpt/usb4
# 
```

この直後にプールの状態を見ると、次のように修復中の状況が見えます。

```console
# zpool status upool
  pool: upool
 state: DEGRADED
status: One or more devices is currently being resilvered.  The pool will
        continue to function, possibly in a degraded state.
action: Wait for the resilver to complete.
  scan: resilver in progress since Wed Dec 24 10:58:00 2025
        630M / 630M scanned, 264M / 630M issued at 20.3M/s
        50.0M resilvered, 41.91% done, 00:00:18 to go
config:

        NAME             STATE     READ WRITE CKSUM
        upool            DEGRADED     0     0     0
          raidz1-0       DEGRADED     0     0     0
            replacing-0  DEGRADED     0     0     0
              gpt/usb0   REMOVED      0     0     0
              gpt/usb4   ONLINE       0     0     0  (resilvering)
            gpt/usb1     ONLINE       0     0     0
            gpt/usb2     ONLINE       0     0     0
            gpt/usb3     ONLINE       0     0     0

errors: No known data errors
#
```

しばらくすると修復が完了していました。

```console
# zpool status upool
  pool: upool
 state: ONLINE
  scan: resilvered 158M in 00:00:38 with 0 errors on Wed Dec 24 10:58:38 2025
config:

        NAME          STATE     READ WRITE CKSUM
        upool         ONLINE       0     0     0
          raidz1-0    ONLINE       0     0     0
            gpt/usb4  ONLINE       0     0     0
            gpt/usb1  ONLINE       0     0     0
            gpt/usb2  ONLINE       0     0     0
            gpt/usb3  ONLINE       0     0     0

errors: No known data errors
#
```

## パターン3：USBメモリをオフラインで置き換える

今まではプールはオンラインのまま、いきなりUSBメモリを抜いて実験しました。今度はプールを一旦システムから切り離して、USBメモリを1個別のものと交換したのちオンラインに復帰してみます。

パターン1、2はUSBメモリのようなホットプラグ可能なデバイスだったからこそできるオンラインのままでのRAIDのメンテナンスでしたが、実際にストレージトラブルの対処としてはこのパターンになることが多いと思います。

### upoolのエクスポート

現在接続しているupoolを`zpool export`でサーバーからオフラインにします。

```console
# zpool export upool
# zpool list upool
cannot open 'upool': no such pool
# 
```

エクスポート済であれば、必要に応じてサーバーから切り離してメンテナンスが行えます。

### da2のusb2をusb5に交換する

usb2(da2)を抜いて、新たなUSBメモリとしてusb5を差します。usb5はda2として認識されます。

```console
# gpart show -l da2
=>     40  3862448  da2  GPT  (1.8G)
       40     2008       - free -  (1.0M)
     2048  3858432    1  usb5  (1.8G)
  3860480     2008       - free -  (1.0M)

#
```

### upoolのインポート

オフライン状態であったupoolを`zpool import`でサーバーにインポートします。状態は次のようになります。

```console
# zpool import upool
# zpool status upool
  pool: upool
 state: DEGRADED
status: One or more devices could not be opened.  Sufficient replicas exist for
        the pool to continue functioning in a degraded state.
action: Attach the missing device and online it using 'zpool online'.
   see: https://openzfs.github.io/openzfs-docs/msg/ZFS-8000-2Q
  scan: resilvered 158M in 00:00:38 with 0 errors on Wed Dec 24 10:58:38 2025
config:

        NAME                      STATE     READ WRITE CKSUM
        upool                     DEGRADED     0     0     0
          raidz1-0                DEGRADED     0     0     0
            gpt/usb4              ONLINE       0     0     0
            gpt/usb1              ONLINE       0     0     0
            16031226974783295807  UNAVAIL      0     0     0  was /dev/gpt/usb2
            gpt/usb3              ONLINE       0     0     0

errors: No known data errors
#
```

稼働中に抜いた場合と変わり、RAIDZメンバーから`gpt/usb2`がなくなり、代わりに`16031226974783295807`というIDが表示されています。action:では無くなったデバイスを接続して`zpool online`を実行すべきとアドバイスがあります。

### upoolの修復

今回は交換が目的なので`zpool replace`を実行します。usb2は接続されていないため`gpt/usb2`を交換元のデバイス名には使えませんが、その場合IDを指定します。

```console
# zpool replace upool 16031226974783295807 gpt/usb5
# zpool status upool
  pool: upool
 state: DEGRADED
status: One or more devices is currently being resilvered.  The pool will
        continue to function, possibly in a degraded state.
action: Wait for the resilver to complete.
  scan: resilver in progress since Wed Dec 24 11:16:58 2025
        631M / 631M scanned, 73.5M / 630M issued at 10.5M/s
        2.29M resilvered, 11.66% done, 00:00:53 to go
config:

        NAME                        STATE     READ WRITE CKSUM
        upool                       DEGRADED     0     0     0
          raidz1-0                  DEGRADED     0     0     0
            gpt/usb4                ONLINE       0     0     0
            gpt/usb1                ONLINE       0     0     0
            replacing-2             DEGRADED     0     0     0
              16031226974783295807  UNAVAIL      0     0     0  was /dev/gpt/usb2
              gpt/usb5              ONLINE       0     0     0  (resilvering)
            gpt/usb3                ONLINE       0     0     0

errors: No known data errors
#
```

しばらくして修復(置き換え)が完了したのを確認します。

```console
# zpool status upool
  pool: upool
 state: ONLINE
  scan: resilvered 159M in 00:01:18 with 0 errors on Wed Dec 24 11:18:16 2025
config:

        NAME          STATE     READ WRITE CKSUM
        upool         ONLINE       0     0     0
          raidz1-0    ONLINE       0     0     0
            gpt/usb4  ONLINE       0     0     0
            gpt/usb1  ONLINE       0     0     0
            gpt/usb5  ONLINE       0     0     0
            gpt/usb3  ONLINE       0     0     0

errors: No known data errors
#
```

パターン3はトラブルとは関係なく、容量の大きなデバイスへの交換でも活用できます。

## RAIDZ2 - さらなる冗長性の確保

実験は以上で、RAIDZの冗長性とZFSのプール操作についてある程度理解できたと思います。はじめに書いたようにRAIDZでは組み合わせているストレージの1台分を冗長性を確保し、1台の故障までは耐えらえるようにしています。しかし冗長性は1台分しか無いため、RAIDの修復(ZFSではresilver)中にさらに別のストレージにトラブルが発生するとファイルシステムが利用できなくなります。このような状況でも運用し続けられるようさらに1台分つまり合計2台分の冗長性を持たせるRAID構成がRAIDZ2(一般的なRAIDではRAID 6)です。RAIDZやRAIDZ2についての詳細は、[RAIDを導入する前に考えること](https://qiita.com/belgianbeer/items/42bd1dd5c35c96d808a8)の[RAIDZ](https://qiita.com/belgianbeer/items/42bd1dd5c35c96d808a8#raidz)の段落に説明してあります。
