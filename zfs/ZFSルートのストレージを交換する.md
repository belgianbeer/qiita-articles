<!-- https://qiita.com/belgianbeer/items/b751c2036c7ee698fde2 -->
# ZFSルートのストレージを交換する

## はじめに

「[令和の今頃になってSSDを使うのを止めた話](https://qiita.com/belgianbeer/items/37ca20884b29b0e8514e)」で書いた通り、筆者の自宅のFreeBSDサーバーからSSD撤去することになりました。ここでは実際に作業を行ったときの記録とともに、ZFSルートのストレージを交換する手順について説明します。

## 内蔵ストレージの構成変更

SSDを使っていたときのストレージ構成は次の表のようになっていました。

<table>
  <caption> SSD使用時のストレージ構成 </caption>
  <thead>
    <tr>
      <th align="center"> デバイス </th> <th align="center"> 種類 </th> <th align="center"> 容量 </th> <th align="center"> 用途 </th>
    </tr>
  </thead>
  <tr>
    <td align="center"> ada0 </td> <td align="center"> SSD </td> <td align="center"> 250 GB </td> <td> FreeBSDのシステム、ブート、スワップ </td>
  </tr>
  <tr>
    <td align="center"> ada1 </td> <td align="center"> HDD </td> <td align="center"> 6 TB </td> <td> /home を含むデータ領域 (ada2 と ZFSミラーを構成) </td>
  </tr>
  <tr>
    <td align="center"> ada2 </td> <td align="center"> HDD </td> <td align="center"> 6 TB </td> <td> ada1に同じ (ada1 と ZFSミラーを構成) </td>
  </tr>
</table>

これを次の手順で作業を行うことで、SSDを撤去します。なお以降の説明では2台のHDDを区別するため、一方(ada1側)をHDD1、他方(ada2側)をHDD2と呼びます。

1. HDD1とHDD2で構成されているZFSミラーを解除してHDD1をフリーにする
2. HDD1のパーティションを作り直し、ブート、データ、スワップの各領域を用意する
3. HDD1にブートコードを書き込む
4. SSDのシステムデータをHDD1にコピーする
5. 電源を切ってSSDをシステムから撤去する
6. HDD2のデータをHDD1の残りの領域にコピーする
7. HDD2のパーティションを作り直してHDD1と同じにする
8. HDD1とHDD2のデータ領域をZFSでミラーにする

ここで2から4の手順は、インストーラを使って新たなFreeBSDをHDD1にインストールし、SSDの既存の設定内容をHDD1にコピーする方法でも可能です。しかし既存の設定を確実に網羅するためにはSSDの内容を直接コピーする方が賢明と言えます。

最終的には次の構成になります。

<table>
  <caption> SSD撤去後のストレージ構成 </caption>
  <thead>
    <tr>
      <th align="center"> デバイス </th> <th align="center"> 種類 </th> <th align="center"> 容量 </th> <th align="center"> 用途 </th>
    </tr>
  </thead>
  <tr>
    <td align="center"> ada0 </td> <td align="center"> HDD </td>  <td align="center"> 6 TB </td> <td> FreeBSDのシステム、ブート、スワップ、ユーザーデータ</td>
  </tr>
  <tr>
    <td align="center"> ada1 </td> <td align="center"> HDD </td>  <td align="center"> 6 TB </td> <td> ada0に同じ (ada0 と ZFSミラーを構成) </td>
  </tr>
</table>

それでは順を追って、構成変更を説明します。

## 既存のパーティションテーブルの確認

最初に既存のパーティションテーブルを確認します。SSDのパーティションは次のようになっています。

```console
$ gpart show ada0    # SSDのパーティション情報の表示
=>       40  488397088  ada0  GPT  (233G)
         40       1024     1  freebsd-boot  (512K)
       1064        984        - free -  (492K)
       2048    8388608     2  freebsd-swap  (4.0G)
    8390656  480006144     3  freebsd-zfs  (229G)
  488396800        328        - free -  (164K)
```

当サーバーは2010年頃のもので、UEFI非対応のBIOSです。そのためESP(EFI System Partition)が無く、freebsd-boot(512KB)、freebsd-swap(4GB)、freebsd-zfs(229GB)の3個のパーティションから成っています。

一方HDD側は次の通りで、全領域がデータ用のZFSパーティションです。

```console
$ gpart show ada1    # HDDのパーティション情報の表示 (ada2も同じ構成)
=>         40  11721045088  ada1  GPT  (5.5T)
           40  11721045088     1  freebsd-zfs  (5.5T)

$
```

ZFSプールの構成は次の通りです。

```console
$ zpool list -v
NAME             SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zroot            228G  23.0G   205G        -         -    15%    10%  1.00x    ONLINE  -
  ada0p3         229G  23.0G   205G        -         -    15%  10.1%      -    ONLINE
zvol0           5.45T  2.77T  2.68T        -         -     1%    50%  1.00x    ONLINE  -
  mirror-0      5.45T  2.77T  2.68T        -         -     1%  50.8%      -    ONLINE
    gpt/sdisk1  5.46T      -      -        -         -      -      -      -    ONLINE
    gpt/sdisk2  5.46T      -      -        -         -      -      -      -    ONLINE
```

zrootがSSD側のZFSプール、zvol0が2台のHDDで構成されたZFSのミラープールとなります。

## HDD1のパーティションの作り直し

前述の通りHDDには空き領域が無いので、そのままではブート用のパーティションを用意することができません。そこで一旦ミラープールからミラーを解除して、HDD1をフリーにします。

```console
$ zpool detach zvol1 gpt/sdisk1
$ zpool labelclear /dev/ada1p1     # 今回は不要
```

これでzvol0は単純なZFSプールになり、HDD1のパーティションを変更できます。もちろんミラーを解除したので、万が一HDD2側にエラーがあるとデータを失うリスクがあります。

それではHDD1のパーティションを作り直します。

```console
$ gpart show ada1                                             # HDD1のパーティションの確認
=>         40  11721045088  ada1  GPT  (5.5T)
           40  11721045088     1  freebsd-zfs  (5.5T)

$ gpart delete -i 1 ada1                                      # データ用のパーティションの削除
ada1p1 deleted
$ gpart add -a 4k -t freebsd-boot -s 512K  -l gptboot0 ada1   # freebsd-bootパーティションの作成
ada1p1 added
$ gpart add -a 4k -t efi          -s 260M  -l efiboot0 ada1   # ESP パーティションの作成
ada1p2 added
$ gpart add -a 4k -t freebsd-swap -s 4G    -l swap0    ada1   # スワップ用パーティションの作成
ada1p3 added
$ gpart add -a 4k -t freebsd-zfs  -s 5367G -l hdpool0  ada1   # システム、データ用のZFS用パーティションの作成 
ada1p4 added
$ gpart show ada1                                             # 作ったパーティションの確認
=>         40  11721045088  ada1  GPT  (5.5T)
           40         1024     1  freebsd-boot  (512K)
         1064       532480     2  efi  (260M)
       533544      8388608     3  freebsd-swap  (4.0G)
      8922152  11255414784     4  freebsd-zfs  (5.2T)
  11264336936    456708192        - free -  (218G)

$
```

今回新たにESPを割り当てているのは、もし将来サーバーを交換してHDDをそのまま使い続けることになった場合、新たなサーバーはUEFI対応のものになるのは確実なので、ESPが無いと起動のために再びパーティションの作り直す羽目になるのを防ぐ目的です。他にも少し予備領域を用意しておこうと考え200GB程度残してあります。

## HDD1へのブートコードの書き込み

パーティションが構成できたので、HDD1からFreeBSDを起動できるようにブートコードを書き込みます。
HDD1はGPTで構成したので、「/boot/zfsgptboot」をfreebsd-bootパーティションに書き込みます。

```console
$ gpart bootcode -b /boot/pmbr -p /boot/gptzfsboot -i 1 ada1
partcode written to ada1p1
bootcode written to ada1
$
```

## SSDのシステムデータのHDD1へのコピー

システムデータをコピーするため、HDD1に新たなZFSプールを作成します。本当はインストーラのデフォルトの名前である「zroot」で作りたいのですが、すでに「zroot」があるため後で名前を変更することにして一旦「zroot2」という名前で作っています。またaltrootを指定しているのは、これからコピーするのは「zroot」にあるFreeBSDのOS部分なので、ディレクトリ構成が衝突しないようにしています。

```console
$ zpool create -o altroot=/mnt -O compress=lz4 -O atime=off -m none -f zroot2 gpt/hdpool0
$ zpool list
NAME     SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zroot    228G  23.0G   205G        -         -    15%    10%  1.00x    ONLINE  -
zroot2  5.23T   384K  5.23T        -         -     0%     0%  1.00x    ONLINE  /mnt
zvol0   5.45T  2.77T  2.68T        -         -     1%    50%  1.00x    ONLINE  -
$
```

次はシステムのコピーです。これにはzfsのsendとreceive (recv)を使います。send/receiveはZFSのデータセットを効率よく転送できます。特にsendの「-R」オプションでは、指定したデータセット配下の全てを転送してくれるので、多数のZFSのデータセットがあってもまとめて処理できます。

```console
$ zfs snapshot -r zroot@copy                            # 転送元になるsnapshotの作成
$ zfs send -R zroot@copy | zfs receive -v -F -u zroot2  # sendとrecvでzrootからzroot2へコピー
.....
$
```

コピー時間は計測しなかったのですが、15分程度だったと思います。これで大部分の内容がコピーできましたが、snapshotを作成した時点からコピー終了までの時間に書き込まれた分(syslogによるもの等)のコピーが出来ていません。確実に全データをコピーするため、ここからはシングルユーザーモードで作業を行います。

シングルユーザーモードに移るには、プロセス番号1のinitにkillを実行します。

```console
$ kill 1
```

しばらくすると「/bin/sh」の起動を促されるのでEnterを入力してシングルモードに移ります。


```console
# zfs snapshot -r zroot@copy1      # snapshotの作成
# zfs send -R -I zroot@copy zroot@copy1 | zfs receive -v -u zroot2 # 前回のsnapshotからの差分を転送
```

以上でSSDからのコピーが完了したので、そのまま電源を切ります。

```console
$ zfs snapshot -r zroot@copy
$ zfs send -R zroot@copy | zfs receive -v -F -u zroot2
$
```



```console
$ gpart bootcode -b /boot/pmbr -p /boot/gptzfsboot -i 1 ada1
partcode written to ada1p1
bootcode written to ada1
$
```



```console
$ gpart show ada0
=>       40  488397088  ada0  GPT  (233G)
         40       1024     1  freebsd-boot  (512K)
       1064        984        - free -  (492K)
       2048    8388608     2  freebsd-swap  (4.0G)
    8390656  480006144     3  freebsd-zfs  (229G)
  488396800        328        - free -  (164K)

$ 
```


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

```console
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
