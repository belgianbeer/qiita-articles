# 令和の今頃になってSSDを使うのを止めた話

## それは電源の故障から始まった

ある日曜日のこと、自宅でPC(Windows)から自宅のFreeBSDサーバーをアクセスしようとして、なぜか反応しないことに気付きました。pingでも反応が無く「え？まさか…ひょっとして…電源が逝っちゃった?」と思いながらサーバーの設置場所に行くとやっぱり電源が落ちてました。こりゃダメだなというのはすぐにわかったのですが、一応と思い電源スイッチを押してみても電源は入らない。ということでサーバーの電源が壊れました。

## SSDだけで仮運用

サーバーには次のように3台のストレージデバイスがあり、SSDがFreeBSD
のシステム、2台のHDDがデータ用のRAID 1(mirror)で、いずれもZFSです。

```text
     SSD
    +----------------+
    | FreeBSD System | 250GB  pool名はzroot 使用量は23GB程度
    +----------------+

      HDD1     HDD2
    +--------+--------+
    | Data   | Data   |  6TB x 2 mirror  pool名はzvol0
    +--------+--------+
```

```console
$ zpool list -v
NAME             SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zroot            228G  23.0G   205G        -         -    15%    10%  1.00x    ONLINE  -
  ada0p3         229G  23.0G   205G        -         -    15%  10.1%      -    ONLINE
zvol0           5.45T  2.77T  2.68T        -         -     1%    50%  1.00x    ONLINE  -
  mirror-0      5.45T  2.77T  2.68T        -         -     1%  50.8%      -    ONLINE
    gpt/sdisk1  5.46T      -      -        -         -      -      -      -    ONLINE
    gpt/sdisk2  5.46T      -      -        -         -      -      -      -    ONLINE
$
```

自宅のサーバーではあるものの外部からCVSやGitのリポジトリなどを利用する関係で常時稼働していないと困るので、一時的な措置としてSSDを抜き出し、SATA/USB変換ケーブルで余っていたノートPCに取り付けてサーバーが復活するまでの代替機として稼働させました。もちろんHDD内の必要なデータ(常時必要なものはごく一部)はSSDにコピーしています。

## サーバーの復活と不調

3週間ぐらいで新しい電源を入手できたので、臨時運用のノートPCからSSDを外してサーバの筐体に戻し、元通りの構成にして稼働させました。しかし稼働して翌日には

```console
ahcich0: Timeout on slot 20 port 0
ahcich0: is 00000002 cs 00000000 ss 00000000 rs 00100000 tfd 50 serr 00000000 cmd 00047417
(aprobe0:ahcich0:0:0:0): ATA_IDENTIFY. ACB: ec 00 00 00 00 40 00 00 00 00 00 00
(aprobe0:ahcich0:0:0:0): CAM status: Command timeout
(aprobe0:ahcich0:0:0:0): Error 5, Retries exhausted
```

などのメッセージ出力されてサーバーとして正常な動作が出来ない状態に陥りました。

こうなるとリセットするしか手はありません。リセットすればしばらく問題は無いのですが、短い場合は数時間、長くてもせいぜい翌々日には同様の状態に陥ります。

## 対処方法を検討

状況を整理してみます。

- ahcich0はSSDが接続してあるコントロールチャンネルなので、SSDのアクセスでトラブルが起きている。実際HDDへの読み書きではトラブルは発生しない。
- 交換した電源は以前と同じ容量の250Wであるが、電源が壊れる以前はまったく問題が無かった。

SSDが不良になったというのは考えにくいので、今回のSSDの接続し直しでケーブルの接触などに不具合が発生したのもありうると思いケーブルを交換してみましたが状況は改善しませんでした。

今回交換した電源は同じ250Wなのに実際には容量が不足していて、SSDの書き込み時に電力不足に陥っているのかもしれません(SSDの書き込み時の電力消費はHDDよりも大きい)。そうなると新たに別の電源ユニットを購入して交換する方法が考えられます。ただ別の電源を購入するにも微妙に入手しにくいタイプ(Frex ATX)で、交換したところで実際に消費している電力を計測したわけでは無いので、まったく違うものが原因である可能性も残っています。

## 解決策

数日経過して、ふと「SSDへのアクセスでトラブルが起きるのだから、SSDを止めればいい」ということに気付きました。SSDはほぼ純粋にFreeBSDのシステムとして利用しているだけで、データ領域は元々HDDを使っていますし、自分のホームもHDDにあります。そもそも本サーバーの当初はHDDだけで、通常の利用ではパフォーマンス面での問題はまったくありませんでした。ただHDDだけではfreebsd-updateに時間がかかるので、システム部だけをSSDにして独立させたという経緯があります。本サーバーの現状を考えるとSSDを止める(サーバーから抜く)のがトラブル対処として間違いないという判断となりました。

## 作業手順

SSDを抜くためには、HDDだけでサーバーが動作できるようにSSD上のシステムをHDDに移す必要があります。また現状HDDはデータ保存しか考慮していなかったので全体を1パーティションで構成していて、FreeBSDの起動に必要なブートコードを置く領域が有りません。

```console
$ gpart show ada1    # HDDのパーティションの確認 (ada1も同じ構成)
=>         40  11721045088  ada1  GPT  (5.5T)
           40  11721045088     1  freebsd-zfs  (5.5T)

$
```

そこでSSDを抜くには、2台のHDDを、HDD1、HDD2として次の手順を実行する必要があります。

1. ZFSミラーを解除してHDD1をフリーにする
2. HDD1のパーティションを作り直し、ブート、データ、スワップを用意する。
3. SSDのシステムデータをHDD1にコピーする
4. HDD1にブートコードを書き込む
5. 電源を切ってSSDを抜く
6. HDD2のデータをHDD1の空き領域にコピーする
7. HDD2のパーティションを作り直してHDD1と同じにする
8. HDD1とHDD2のデータ領域をZFSでミラーにする

具体的な作業については「ZFSルートのシステムディスクを交換する」で説明します。





```text
      SSD
    +--------------+
    | freebsd-boot | 512KB
    +--------------+
    | freebsd-swap | 2GB
    +--------------+
    | freebsd-zfs  | 223GB
    +--------------+

      HDD
    +--------------+
    | freebsd-zfs  | 5.5TB
    +--------------+
```


```console
$ zpool detach zvol1 gpt/sdisk1
$ zpool labelclear /dev/ada1p1
```

```console
$ gpart show ada1
=>         40  11721045088  ada1  GPT  (5.5T)
           40  11721045088     1  freebsd-zfs  (5.5T)

$ gpart delete -i 1 ada1
ada1p1 deleted
$ gpart add -a 4k -t freebsd-boot -s 512K -l gptboot0 ada1
ada1p1 added
$ gpart add -a 4k -t efi          -s 260M -l efiboot0 ada1
ada1p2 added
$ gpart add -a 4k -t freebsd-swap -s 4G   -l swap0    ada1
ada1p3 added
$ gpart add -a 4k -t freebsd-zfs  -s 5367G  -l hdpool0     ada1
ada1p4 added
$ gpart show ada1
=>         40  11721045088  ada1  GPT  (5.5T)
           40         1024     1  freebsd-boot  (512K)
         1064       532480     2  efi  (260M)
       533544      8388608     3  freebsd-swap  (4.0G)
      8922152  11255414784     4  freebsd-zfs  (5.2T)
  11264336936    456708192        - free -  (218G)

$
```

```console
$ gpart bootcode -b /boot/pmbr -p /boot/gptzfsboot -i 1 ada1
partcode written to ada1p1
bootcode written to ada1
$ zpool create -o altroot=/mnt -O compress=lz4 -O atime=off -m none -f zroot2 gpt/hdpool0
$ zpool list
NAME     SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zroot    228G  23.0G   205G        -         -    15%    10%  1.00x    ONLINE  -
zroot2  5.23T   384K  5.23T        -         -     0%     0%  1.00x    ONLINE  /mnt
zvol0   5.45T  2.77T  2.68T        -         -     1%    50%  1.00x    ONLINE  -
$
```

```console
$ zfs snapshot -r zroot@copy
$ zfs send -R zroot@copy | zfs receive -v -F -u zroot2
$
```

```console
$ zfs snapshot -r zroot@copy1
$ zfs send -R -I zroot@copy zroot@copy1 | zfs receive -v -u zroot2
```

USBでbootする

```console
$ zpool import -R /mnt -N zroot2 zroot
$ zpool export zroot
```

reboot

```console
$ zpool list -v
NAME            SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zroot          5.23T  22.9G  5.21T        -         -     0%     0%  1.00x    ONLINE  -
  gpt/hdpool0  5.24T  22.9G  5.21T        -         -     0%  0.42%      -    ONLINE
zvol0          5.45T  2.77T  2.68T        -         -     1%    50%  1.00x    ONLINE  -
  gpt/sdisk2   5.46T  2.77T  2.68T        -         -     1%  50.8%      -    ONLINE
```

```console

$ zfs create zroot/zvol0
$ zfs send -R zvol0@_daily_2025-03-07 | zfs receive -v -F -u zroot/zvol0
    29134.60 real         2.76 user      3231.93 sys
```

```console
$ zfs snapshot -r zvol0@moving
$ zfs send -R -I zvol0@_daily_2025-03-07 zvol0@moving | zfs receive -v -u zroot/zvol0
      278.42 real         1.69 user        50.69 sys
```

```console
$ kill 1
```

```console
$ zpool export zvol0

zfs rename zroot/zvol0/annually  zroot/annually || exit
sleep 1
zfs rename zroot/zvol0/archive   zroot/archive || exit
sleep 1
zfs rename zroot/zvol0/backup    zroot/backup || exit
sleep 1
zfs rename zroot/zvol0/backupold zroot/backupold || exit
sleep 1
zfs rename zroot/zvol0/home      zroot/home || exit
sleep 1
zfs rename zroot/zvol0/opt       zroot/opt || exit
```

```console
$ mkdir /zvol0
$ zpool import -R /zvol0 -N zvol0
$ zpool destroy zvol0
```

$ gpart delete -i 1 ada1
$ gpart add -a 4k -t freebsd-boot -s 512K -l gptboot1 ada1
ada1p1 added
$ gpart add -a 4k -t efi          -s 260M -l efiboot1 ada1
ada1p2 added
$ gpart add -a 4k -t freebsd-swap -s 4G   -l swap1    ada1
ada1p3 added
$ gpart add -a 4k -t freebsd-zfs  -s 5367G  -l hdpool1     ada1
ada1p4 added
$
```

```console
$ zpool attach zroot gpt/hdpool0 gpt/hdpool1
```

```console
$ zpool status
  pool: zroot
 state: ONLINE
status: One or more devices is currently being resilvered.  The pool will
        continue to function, possibly in a degraded state.
action: Wait for the resilver to complete.
  scan: resilver in progress since Fri Mar  7 23:42:43 2025
        4.39G / 2.79T scanned at 321M/s, 0B / 2.79T issued
        0B resilvered, 0.00% done, no estimated completion time
config:

        NAME             STATE     READ WRITE CKSUM
        zroot            ONLINE       0     0     0
          mirror-0       ONLINE       0     0     0
            gpt/hdpool0  ONLINE       0     0     0
            gpt/hdpool1  ONLINE       0     0     0

errors: No known data errors
$
```
