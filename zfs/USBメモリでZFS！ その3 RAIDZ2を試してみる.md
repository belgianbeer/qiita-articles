<!-- https://qiita.com/belgianbeer/items/035227ac9cc3e59747da -->
# USBメモリでZFS！ その3 RAIDZ2を試してみる

## はじめに

[USBメモリでZFS！](https://qiita.com/belgianbeer/items/156251d0675d456c2207)で、USBメモリを使ったRAIDZのストレージセットを作り、RAIDZの耐障害性と修復については[その2](https://qiita.com/belgianbeer/items/9726a3e8be624c4cedf2)に記載しました。今回は一般的なRAID 6に相当するRAIDZ2で、さらなる信頼性のあるストレージシステムを試します。

## RAIDZとRAIDZ2

RAIDZでは組み合わせているストレージの1台分の容量の冗長性を確保し、いずれかの1台が故障してもデータの継続性を確保できます。しかし冗長性は1台分しか無いため、ある1台が故障し、修復(ZFSではresilver)中万が一別のストレージにトラブルが発生するとファイルシステムが利用できなくなります。このような状況でも運用し続けられるようさらに1台分、つまり合計2台分の容量の冗長性を持たせるRAIDがRAIDZ2です。

RAIDZやRAIDZ2についての詳細は、[RAIDを導入する前に考えること](https://qiita.com/belgianbeer/items/42bd1dd5c35c96d808a8#raidz)の[RAIDZ](https://qiita.com/belgianbeer/items/42bd1dd5c35c96d808a8#raidz)
の段落に記述してあるので、当記事と合わせて見てください。

## RAIDZで使ったUSBメモリを解放

[前回の記事](https://qiita.com/belgianbeer/items/9726a3e8be624c4cedf2)でUSBメモリはRAIDZを構成したままなので、RAIDZ2を構成するためにRAIDZのプールを一旦削除します。プールの削除には`zpool destroy`を使います。実運用であればプールの削除は必要なデータがあるかどうかを見極めて慎重に実施すべきものですが、今回は当初から実験であるためサクっと削除を実行します。

```console
# zpool list upool
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
upool     7G  1.25G  5.75G        -         -     0%    17%  1.00x    ONLINE  -
# zpool destory upool
# zpool list upool
cannot open 'upool': no such pool
#
```

これで既存のRAIDZのプールを削除できました。

## RAIDZ2を構成する

USBメモリの準備ができたら、次のコマンドでRAIDZ2を構成します。[RAIDZを構成するコマンド](https://qiita.com/belgianbeer/items/156251d0675d456c2207#raidz-%E3%82%92%E6%A7%8B%E6%88%90%E3%81%99%E3%82%8B)の`raidz`の部分を`raidz2`に変更するだけです。

```console
# zpool create -O atime=off -O compression=lz4 -f upool raidz2 gpt/usb0 gpt/usb1 gpt/usb2 gpt/usb3
#
```

Linuxで試す場合はデバイス名を指定する際の"gpt/"が不要になります。

> RAIDZを構成せず直接RAIDZ2を試す場合はUSBメモリを初期化する必要がありますが、初期化については[USBメモリでZFS！](https://qiita.com/belgianbeer/items/156251d0675d456c2207)の通りなので、そちらをご覧ください。

出来上がったプールの構成を`zpool list -v`で確認すると、次のようにupoolの構成がraidz2であることを示しています。

```console
# zpool list -v upool
NAME           SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
upool            7G   322K  7.00G        -         -     0%     0%  1.00x    ONLINE  -
  raidz2-0       7G   322K  7.00G        -         -     0%  0.00%      -    ONLINE
    gpt/usb0  1.84G      -      -        -         -      -      -      -    ONLINE
    gpt/usb1  1.84G      -      -        -         -      -      -      -    ONLINE
    gpt/usb2  1.84G      -      -        -         -      -      -      -    ONLINE
    gpt/usb3  1.84G      -      -        -         -      -      -      -    ONLINE
#
```

できたZFSの容量を確認すると、次のように3.36GBの空きを確認できます。

```console
# zfs list upool
NAME    USED  AVAIL  REFER  MOUNTPOINT
upool   161K  3.36G  32.9K  /upool
# 
```

RAIDZのときの空き容量は5.07GBでしたが、1個あたり1884MBに設定したUSBメモリを4個使っていますから、4個の合計容量に比べてRAIDZでは1個分少ない3個分の容量、RAIDZ2では2個分少ない2個分の容量となります。実際の容量はファイルシステムとしての管理に必要な領域を除いた残りになるため単純に個数分とはなりません。

## RAIDZ2を壊して修復してみる

### RAIDZ2のプールへデータを書き込む

プールができたので、テストのためにファイルをコピーします。ここでは`/usr/local`を`/upool`へコピーします。

```console
# ls /usr/local               # まず/usr/localの内容をlsで確認
bin     etc     include lib     libdata libexec sbin    share
#
# cp -rp /usr/local /upool    # /usr/localを/upoolへコピー
#
# ls /upool/local             # コピーできていることを確認
bin     etc     include lib     libdata libexec sbin    share
#
```

続いてzfs listとzpool listでupoolの使用量を確認します。

```console
# zfs list upool
NAME    USED  AVAIL  REFER  MOUNTPOINT
upool   965M  2.42G   964M  /upool
# zpool list upool
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
upool     7G  1.89G  5.11G        -         -     1%    27%  1.00x    ONLINE  -
#
```

RAIDZ2のプールが正常に動作していることがわかります。

### RAIDZ2に障害を発生させる

ファイルがコピーできたので、実際にRAIDZ2のプールを構成するUSBメモリを抜いて障害をシミュレーションしてみます。あらためてプールの状態を確認します。

```console
# zpool status upool
  pool: upool
 state: ONLINE
config:

        NAME          STATE     READ WRITE CKSUM
        upool         ONLINE       0     0     0
          raidz2-0    ONLINE       0     0     0
            gpt/usb0  ONLINE       0     0     0
            gpt/usb1  ONLINE       0     0     0
            gpt/usb2  ONLINE       0     0     0
            gpt/usb3  ONLINE       0     0     0

errors: No known data errors
#
```

ここでusb3のパーティションのある/dev/da3を抜きました。zpool statusで確認すると、次のようになります。

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
          raidz2-0    DEGRADED     0     0     0
            gpt/usb0  ONLINE       0     0     0
            gpt/usb1  ONLINE       0     0     0
            gpt/usb2  ONLINE       0     0     0
            gpt/usb3  REMOVED      0     0     0

errors: No known data errors
#
```

`state:`がDEGRADEDとなり、usb3はREMOVEDと抜かれた状態で障害が発生していることがわかります。RAIDZ2なので、この状態でもデータの整合性は保証されます。1個壊れた状態で修復を行えば、修復中に別のストレージデバイスが故障してもデータとしては問題なくアクセスし続けられます。

```console
# ls -l /upool/local
total 349
drwxr-xr-x    2 root wheel  816 Jan  3 17:13 bin
drwxr-xr-x   14 root wheel   29 Dec  1 09:55 etc
drwxr-xr-x  101 root wheel  265 Dec 12 23:07 include
drwxr-xr-x   27 root wheel 1108 Jan  3 17:13 lib
drwxr-xr-x    4 root wheel    4 Jan 20  2023 libdata
drwxr-xr-x    6 root wheel   10 Dec  1 09:55 libexec
drwxr-xr-x    2 root wheel   35 Jan  3 17:13 sbin
drwxr-xr-x   39 root wheel   39 Dec  1 09:55 share
#
```

### RAIDZ2の修復

RAIDZ2の修復方法はRAIDZと同じです。新しいデバイスを接続して`zpool replace`を実行すればresilverが動きしばらくすれば修復が完了します。ここでは別途用意したusb4で修復します。

```console
# zpool replace upool gpt/usb3 gpt/usb4
# zpool status upool
  pool: upool
 state: DEGRADED
status: One or more devices is currently being resilvered.  The pool will
        continue to function, possibly in a degraded state.
action: Wait for the resilver to complete.
  scan: resilver in progress since Wed Jan 14 10:56:45 2026
        1.89G / 1.89G scanned, 84.4M / 1.89G issued at 14.1M/s
        5.73M resilvered, 4.37% done, 00:02:11 to go
config:

        NAME             STATE     READ WRITE CKSUM
        upool            DEGRADED     0     0     0
          raidz2-0       DEGRADED     0     0     0
            gpt/usb0     ONLINE       0     0     0
            gpt/usb1     ONLINE       0     0     0
            gpt/usb2     ONLINE       0     0     0
            replacing-3  DEGRADED     0     0     0
              gpt/usb3   REMOVED      0     0     0
              gpt/usb4   ONLINE       0     0     0  (resilvering)

errors: No known data errors
#
```

このまま少し待てば修復が完了しますが、ここでは修復中にさらに別のデバイスが故障した場合を想定しusb0(/dev/da0)を抜いてみます。`zpool status`で確認すると次のようになりました。

```console
# zpool status upool
  pool: upool
 state: DEGRADED
status: One or more devices is currently being resilvered.  The pool will
        continue to function, possibly in a degraded state.
action: Wait for the resilver to complete.
  scan: resilver in progress since Wed Jan 14 10:56:45 2026
        1.89G / 1.89G scanned, 559M / 1.89G issued at 3.14M/s
        130M resilvered, 28.88% done, no estimated completion time
config:

        NAME             STATE     READ WRITE CKSUM
        upool            DEGRADED     0     0     0
          raidz2-0       DEGRADED     0     0     0
            gpt/usb0     FAULTED      0     0     0  too many errors
            gpt/usb1     ONLINE       0     0     0
            gpt/usb2     ONLINE       0     0     0
            replacing-3  DEGRADED     0     0     0
              gpt/usb3   REMOVED      0     0     0
              gpt/usb4   ONLINE       0     0     0  (resilvering)

errors: No known data errors
#
```

usb0はFAULTEDとなりましたが、プール全体の状態を示す`state:`としてはDEGRADEDで動作可能なことを示しています。RAIDZなら修復中のさらなる障害はファイルシステムとして致命的なエラーとなりますが、RAIDZ2であるため/upool配下への読み書きは問題無く行えます。

```console
# ls -l /upool/local
total 349
drwxr-xr-x    2 root wheel  816 Jan  3 17:13 bin
drwxr-xr-x   14 root wheel   29 Dec  1 09:55 etc
drwxr-xr-x  101 root wheel  265 Dec 12 23:07 include
drwxr-xr-x   27 root wheel 1108 Jan  3 17:13 lib
drwxr-xr-x    4 root wheel    4 Jan 20  2023 libdata
drwxr-xr-x    6 root wheel   10 Dec  1 09:55 libexec
drwxr-xr-x    2 root wheel   35 Jan  3 17:13 sbin
drwxr-xr-x   39 root wheel   39 Dec  1 09:55 share
```

しばらく待つとresilverは終了しusb4はプールに組み込まれた状態になりますが、`state:`はDEGRADEDでusb0はFAULTEDのままです。

```console
# zpool status upool
  pool: upool
 state: DEGRADED
status: One or more devices are faulted in response to persistent errors.
        Sufficient replicas exist for the pool to continue functioning in a
        degraded state.
action: Replace the faulted device, or use 'zpool clear' to mark the device
        repaired.
  scan: resilvered 486M in 00:12:24 with 0 errors on Wed Jan 14 11:09:09 2026
config:

        NAME          STATE     READ WRITE CKSUM
        upool         DEGRADED     0     0     0
          raidz2-0    DEGRADED     0     0     0
            gpt/usb0  FAULTED      0     0     0  too many errors
            gpt/usb1  ONLINE       0     0     0
            gpt/usb2  ONLINE       0     0     0
            gpt/usb4  ONLINE       0     0     0

errors: No known data errors
#
```

正常な状態に戻すため、usb0もusb5で置き換えて修復します。

```console
# zpool replace upool gpt/usb0 gpt/usb5
#
# zpool status upool
  pool: upool
 state: DEGRADED
status: One or more devices is currently being resilvered.  The pool will
        continue to function, possibly in a degraded state.
action: Wait for the resilver to complete.
  scan: resilver in progress since Wed Jan 14 11:19:25 2026
        1.89G / 1.89G scanned, 531M / 1.89G issued at 3.66M/s
        134M resilvered, 27.47% done, no estimated completion time
config:

        NAME             STATE     READ WRITE CKSUM
        upool            DEGRADED     0     0     0
          raidz2-0       DEGRADED     0     0     0
            replacing-0  DEGRADED     0     0     0
              gpt/usb0   FAULTED      0     0     0  too many errors
              gpt/usb5   ONLINE       0     0     0  (resilvering)
            gpt/usb1     ONLINE       0     0     0
            gpt/usb2     ONLINE       0     0     0
            gpt/usb4     ONLINE       0     0     0

errors: No known data errors
#
```

しばらく待つと、次の通りusb1、usb2、usb4、usb5で組み合わせたプールとして修復が完了していました。

```console
# zpool status upool
  pool: upool
 state: ONLINE
  scan: resilvered 486M in 00:07:52 with 0 errors on Wed Jan 14 11:27:17 2026
config:

        NAME          STATE     READ WRITE CKSUM
        upool         ONLINE       0     0     0
          raidz2-0    ONLINE       0     0     0
            gpt/usb5  ONLINE       0     0     0
            gpt/usb1  ONLINE       0     0     0
            gpt/usb2  ONLINE       0     0     0
            gpt/usb4  ONLINE       0     0     0

errors: No known data errors
#
```

## RAIDZ3

多くの場合、RAIDZ2でストレージとして十分な信頼性が得られますが、さらに高い信頼性を求める場合、ZFSにはRAIDZ3が用意されています。RAIDZ2では最大2台の故障に耐えられますが、RAIDZ3では最大3台までの故障に対応できます。ただRAIDZ3を構成するには最低5台のストレージデバイスが必要となり、5台で3台を冗長性に割り当てると実容量は2台分しか確保できず、全容量の50%を超える容量を冗長性に割り当てるのは得策とは言えません。5台であればせいぜいRAIDZ2までにとどめ、大規模なストレージシステム（例えばHDDを8台以上でRAIDを構成する）なら、RAIDZ3の採用を検討するべきでしょう。

## シリーズ「USBメモリでZFS！」

- [USBメモリでZFS！](https://qiita.com/belgianbeer/items/156251d0675d456c2207)
- [USBメモリでZFS！ その2 RAIDZを壊して修復してみる](https://qiita.com/belgianbeer/items/9726a3e8be624c4cedf2)
- [USBメモリでZFS！ その3 RAIDZ2を試してみる](https://qiita.com/belgianbeer/items/035227ac9cc3e59747da)
- 続く
