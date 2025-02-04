<!--
https://qiita.com/belgianbeer/items/7483f17548ad217f18a0
https://blog.naa0yama.com/p/02w17-srcjj0yk/
-->
# ZFSプールを縮小する

## はじめに

以前に「[ZFSミラープールのディスク交換による容量増設](https://qiita.com/belgianbeer/items/8df197588462cd7f6b45)」でZFSのミラー構成の容量を大きいものに交換する手順を紹介しました。今回はプールのサイズを小さくする例を紹介します。実際に使うことは余りないかもしれないですが、ストレージデバイスの交換時にも使えるので覚えておいて損はありません。

## ZFSでのプールの縮小

ZFSプールを扱うzpoolコマンドにはaddとremoveというサブコマンドがあります。`zpool add`は指定したデバイスを結合してプールサイズを拡張し[^misc1]、`zpool remove`はプールから指定したデバイスを取り除くコマンドです。これらを使うことでプールのサイズを縮小できます。ただしここで例にするようなプールサイズの縮小に`zpool remove`が使えるようになったのは、FreeBSDの場合13.0-RELEASE以降になります[^misc2]。

[^misc1]:プールサイズの拡張の他に、スペアデバイスの追加等の使い方もあります。

[^misc2]:その前までの`zpool remove`では、スペアやミラープールからのデバイスの削除のような場合に限り利用可能でした。

例えば500GBのプールサイズの使用量が200GB程度なので250GBのプールサイズに縮小したい場合、次の手順で行います。

1. 別のデバイスで250GBのパーティションを用意する。
2. 現在のプールに用意した250GBのデバイスを`zpool add`で追加する。この時点でプールのサイズは一時的に750GBとなる。
3. このプールから500GBのデバイスを`zpool remove`で取り除くと、プールサイズは250GBとなる。

ここではプールサイズを小さくする場合を例にしていますが、同様の手順でプールサイズの拡大もできます。

通常ZFSでデバイスを交換する場合 `zfs send`と`zfs receive`を組み合わせてデータコピーを行うことも多いですが、send/receiveでは別デバイスへのファイルシステムの複製となるため作業完了後デバイスの入れ替えが必要になるのに対し、本手順では**ファイルシステムは変更なくOSに接続されたままサイズ変更やデバイス交換が行える**のがポイントです。動作中のプログラムには影響を与えないので、例え対象がシステムディスクであってもOSを再起動することなくプールサイズを変更できます。

## ZFSルートミラープールのサイズ変更の実例

ZFSプールの縮小方法を理解したところで、実際にルートファイルシステムをZFSミラーで構成したシステムのプールサイズの縮小をやってみます。

### ZFSミラールートの構成

次のzrootはFreeBSDのインストーラでRoot-on-ZFSでインストールしたシステムディスクをさらにミラーで構成したものです。ここでda0は250GB、da1は320GBのUSB HDDを使っています。

```Console
$ zpool list
NAME   SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zroot  230G   699M   229G        -         -     0%     0%  1.00x    ONLINE  -
$ zpool list -v zroot
NAME         SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zroot        230G   699M   229G        -         -     0%     0%  1.00x    ONLINE  -
  mirror-0   230G   699M   229G        -         -     0%  0.29%      -    ONLINE
    da0p4    231G      -      -        -         -      -      -      -    ONLINE
    da1p4    231G      -      -        -         -      -      -      -    ONLINE
$
```

この通りzrootはda0p4とda1p4のミラーとなっています。またパーティションは次のようになっています。

パーティションの様子 (da0側)

```Console
$ gpart show da0
=>       40  488397088  da0  GPT  (233G)
         40     532480    1  efi  (260M)
     532520       1024    2  freebsd-boot  (512K)
     533544        984       - free -  (492K)
     534528    4194304    3  freebsd-swap  (2.0G)
    4728832  483667968    4  freebsd-zfs  (231G)
  488396800        328       - free -  (164K)

$
```
<!-- 
$ gpart show da1
=>       34  625142381  da1  GPT  (298G)
         34          6       - free -  (3.0K)
         40     532480    1  efi  (260M)
     532520       1024    2  freebsd-boot  (512K)
     533544        984       - free -  (492K)
     534528    4194304    3  freebsd-swap  (2.0G)
    4728832  483667968    4  freebsd-zfs  (231G)
  488396800  136745615       - free -  (65G)

-->

### ZFSミラーの解除

まず最初にzrootからda1p4を外してZFSミラーを解除します。もちろんこの状態ではミラー構成の冗長性が無くなるので、万が一のために作業前にバックアップをとることをおすすめします。

```Console
$ zpool detach zroot da1p4
$ zpool list -v zroot
NAME        SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zroot       230G   698M   229G        -         -     0%     0%  1.00x    ONLINE  -
  da0p4     231G   698M   229G        -         -     0%  0.29%      -    ONLINE
$
```

### 小さいパーティションの準備

da1p4がプールから外れてフリーになったので、`gpart resize`を使って小さいパーティションを用意します。ここでは100GBにします。

```Console
$ gpart resize -i 4 -s 100g da1
da1p4 resized
$ gpart show da1
=>       34  625142381  da1  GPT  (298G)
         34          6       - free -  (3.0K)
         40     532480    1  efi  (260M)
     532520       1024    2  freebsd-boot  (512K)
     533544        984       - free -  (492K)
     534528    4194304    3  freebsd-swap  (2.0G)
    4728832  209715200    4  freebsd-zfs  (100G)
  214444032  410698383       - free -  (196G)

$
```

### プールの一時的な拡張

da1p4が100GBにできたので、`zpool add`でzrootに追加して一時的にプールを拡張します。

```Console
$ zpool add zroot da1p4
$ zpool list -v
NAME        SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zroot       330G   699M   329G        -         -     0%     0%  1.00x    ONLINE  -
  da0p4     231G   699M   229G        -         -     0%  0.29%      -    ONLINE
  da1p4     100G    68K  99.5G        -         -     0%  0.00%      -    ONLINE
$ 
```

この通りzrootはda0p4とda1p4で構成された330GBのプールになっていることがわかります。

### zpool removeの実行

拡張したプールから`zpool remove`を使ってda0p4を一旦プールから外します。

```Console
$ zpool remove zroot da0p4
$
```

コマンドラインの記録だけではすぐに終了しているように見えますが、実際は内部でデータコピーが行われるのでスとレージの使用量と転送速度に応じた時間がかかり、コマンド入力後に次のプロンプトが表示されるまでにはしばらく待たされます。

zpool removeの実行中に別途ログインして`zpool status`を実行してプールの状況を確認すると、次のようにデバイス削除のためのresilverが行われていることがわかります。

```Console
$ zpool status
  pool: zroot
 state: ONLINE
  scan: resilvered 577M in 00:00:15 with 0 errors on Mon Jan 27 14:18:54 2025
remove: Evacuation of /dev/da0p4 in progress since Mon Jan 27 16:10:59 2025
        158M copied out of 698M at 39.4M/s, 22.57% done, 0h0m to go
config:

        NAME        STATE     READ WRITE CKSUM
        zroot       ONLINE       0     0     0
          da0p4     ONLINE       0     0     0  (removing)
          da1p4     ONLINE       0     0     0

errors: No known data errors
$
```

da0p4の削除が終了した状態で`zpool list`で確認すると次のようになります。

```Console
$ zpool list -v
NAME           SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zroot         99.5G   698M  98.8G        -         -     0%     0%  1.00x    ONLINE  -
  indirect-0      -      -      -        -         -      -      -      -    ONLINE
  da1p4        100G   698M  98.8G        -         -     0%  0.68%      -    ONLINE
$
```

パーティションサイズは100GBに縮小されてました。ただindirect-0というエントリーが見えますが、どうやda0p4が使われていた残骸のようで、現状消すことはできないようです。もしこうすれば消せるというような情報をお持ちの方は是非コメントをお願いします。

### ミラー構成の復元

da0p4がプールから外れたので、da1p4と同じく100GBに縮小します。

```Console
$ gpart resize -i 4 -s 100g da0
da0p4 resized
$ gpart show da0
=>       40  488397088  da0  GPT  (233G)
         40     532480    1  efi  (260M)
     532520       1024    2  freebsd-boot  (512K)
     533544        984       - free -  (492K)
     534528    4194304    3  freebsd-swap  (2.0G)
    4728832  209715200    4  freebsd-zfs  (100G)
  214444032  273953096       - free -  (131G)

$
```

サイズ変更したda0p4をzrootに`zpool attach`でミラー構成として追加します。

```Console
$ zpool attach zroot da1p4 da0p4
$ zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zroot  99.5G   698M  98.8G        -         -     0%     0%  1.00x    ONLINE  -
$ zpool list -v zroot
NAME           SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zroot         99.5G   698M  98.8G        -         -     0%     0%  1.00x    ONLINE  -
  indirect-0      -      -      -        -         -      -      -      -    ONLINE
  mirror-1    99.5G   698M  98.8G        -         -     0%  0.68%      -    ONLINE
    da1p4      100G      -      -        -         -      -      -      -    ONLINE
    da0p4      100G      -      -        -         -      -      -      -    ONLINE
$
```

`zpool attach`はすぐに終了しますが実際には内部でミラーの同期が行われ、同期が完了すれば作業終了となります。

<!--
```Console
$ zpool status
  pool: zroot
 state: ONLINE
status: One or more devices is currently being resilvered.  The pool will
        continue to function, possibly in a degraded state.
action: Wait for the resilver to complete.
  scan: resilver in progress since Mon Jan 27 16:17:09 2025
        698M / 698M scanned, 161M / 698M issued at 40.2M/s
        148M resilvered, 23.06% done, 00:00:13 to go
remove: Removal of vdev 0 copied 698M in 0h0m, completed on Mon Jan 27 16:11:11 2025
        10.1K memory used for removed device mappings
config:

        NAME          STATE     READ WRITE CKSUM
        zroot         ONLINE       0     0     0
          mirror-1    ONLINE       0     0     0
            da1p4     ONLINE       0     0     0
            da0p4     ONLINE       0     0     0  (resilvering)

errors: No known data errors
$
```
-->

## Linuxでのミラープールの縮小

いつもなら「LinuxのZFSでも同様に操作できます」で終わるところですが、今回は同じ作業をDebianでもやってみます。作業に使ったPCはFreeBSDとDebianをマルチブートでインストールしてあり、Debianも[OpenZFS](https://zfsonlinux.org/)を使って[ルートファイルシステムを含めてZFS](https://openzfs.github.io/openzfs-docs/Getting%20Started/Debian/index.html#root-on-zfs)になっています[^multi]。

対象とするZFSプールはシステムディスクでは無くFreeBSDで作業したUSB HDDをそのまま使い、FreeBSDでの作業で100GBになったパーティションをさらに縮小して50GBに変更します。

[^multiboot]:FreeBSDもDebianも内蔵SSDのそれぞれのパーティションにインストールしてあります。FreeBSDの例では内蔵のSSDを無効にして、USB接続のHDDをシステムディスクに設定して作業を行いました。

### LinuxでのZFSプールのインポート

最初にFreeBSDで使ったプールをインポートします。そのためにまずインポートできるプールを`zpool import`で確認します。

```Console
$ zpool import
   pool: zroot
     id: 2061217757516156705
  state: ONLINE
 action: The pool can be imported using its name or numeric identifier.
 config:

        zroot         ONLINE
          indirect-0  ONLINE
          mirror-1    ONLINE
            sdc       ONLINE
            sdb       ONLINE

   pool: zroot
     id: 10163499461212717814
  state: ONLINE
 action: The pool can be imported using its name or numeric identifier.
 config:

        zroot       ONLINE
          sda       ONLINE
$
```

この通り同じzrootの名前で2つのプールが確認できますが、一つがミラー構成でもう一つがシングルデバイスからなるプールであることがわかります。ミラープールがUSB HDDで構成されたプールで前述のFreeBSDでの操作で利用したプールであり、もう一方は本PCのマルチブート用のFreeBSDのルートファイルシステムがあるプールです。

`zpool import`でインポートするにはプール名を指定する必要がありますが、同じ名前のプールが複数ある場合プール名では区別ができません。そのような場合はプール名では無くidを指定すればよく、この場合は「2061217757516156705」を指定します。

もう一つの注意点は、これからインポートしようしているプールはFreeBSDのルートファイルシステムがあるプールであるということです。つまり /, /usr等Debianのファイルシステムと多くが同じディレクトリ構成なので、**単純にインポートするとDebianのシステムに重ねてマウントされる可能性があり、インポートと同時にトラブルになります**。このようなトラブルを防ぐために`zpool inport`のオプションに`-R /mnt`を指定してFreeBSDのファイルシステムが/mnt下に配置されるようにします。

それではインポートしてみます。

```Console
$ zpool import -R /mnt 2061217757516156705
cannot import 'zroot': pool was previously in use from another system.
Last accessed by zfstest.iot.plathome.co.jp (hostid=0) at Mon Jan 27 17:25:57 2025
The pool can be imported, use 'zpool import -f' to import the pool.
$
```

ZFSプールではインポート済の情報をデバイス内部に記録しているため、エクスポートしていないプールを別システムでそのままインポートしようとするとエラーメッセージが表示されてインポートに失敗します。ここではFreeBSDで使っていたプールをそのままインポートしようとしたので、このようにエラーとなったわけです。

今回はエラーを無視してインポートしても問題無いので、`-f`オプションを指定して強制的にインポートします。またインポートしたプールをマウントする必要は無いため`-N`オプションを追加しています。

```Console
$ zpool import -f -R /mnt -N 2061217757516156705
$ zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
bpool   480M   101M   379M        -      232G     9%    21%  1.00x    ONLINE  -
rpool    29G  1.28G  27.7G        -      203G    10%     4%  1.00x    ONLINE  -
zroot  99.5G   698M  98.8G        -      132G     0%     0%  1.00x    ONLINE  /mnt
$ zpool list -v zroot
NAME           SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zroot         99.5G   698M  98.8G        -      132G     0%     0%  1.00x    ONLINE  /mnt
  indirect-0      -      -      -        -         -      -      -      -    ONLINE
  mirror-1    99.5G   698M  98.8G        -      132G     0%  0.68%      -    ONLINE
    sdc        100G      -      -        -      198G      -      -      -    ONLINE
    sdb        100G      -      -        -      132G      -      -      -    ONLINE
$
```

この通りzrootがインポートできました[^deb]。sdbが250GB、sdcが320GBのUSB HDDです[^size]。

[^deb]:他のプールはbpoolがDebianのブート用のプール、rpoolがルートからのファイルシステム用プールとなります。
[^size]:LinuxのZFSでは`zpool list`ではEXPANDSZで残りのドライブのフリー領域を含んだ分が表示されるようです。

この状態で`zfs list`実行すると、次のようにzroot配下のFreeBSDのファイルシステムが /mntディレクトリ内に一式見えます(例示のため別システムでの結果を使用)。

```console
$ zfs list
NAME                     USED  AVAIL  REFER  MOUNTPOINT
bpool                    101M   251M    96K  /boot
bpool/BOOT              99.4M   251M    96K  none
bpool/BOOT/debian       99.3M   251M  99.3M  /boot
rpool                   1.28G  26.8G    96K  /
rpool/ROOT              1.18G  26.8G    96K  none
rpool/ROOT/debian       1.18G  26.8G  1.18G  /
rpool/home              1.28M  26.8G   104K  /home
rpool/home/minmin        952K  26.8G   856K  /home/minmin
rpool/home/root          252K  26.8G   252K  /root
rpool/usr                212K  26.8G    96K  /usr
rpool/usr/local          116K  26.8G   116K  /usr/local
rpool/var               87.0M  26.8G    96K  /var
rpool/var/cache         50.7M  26.8G  50.7M  /var/cache
rpool/var/lib            192K  26.8G    96K  /var/lib
rpool/var/lib/nfs         96K  26.8G    96K  /var/lib/nfs
rpool/var/log           35.7M  26.8G  35.7M  /var/log
rpool/var/mail            96K  26.8G    96K  /var/mail
rpool/var/spool          104K  26.8G   104K  /var/spool
rpool/var/tmp            120K  26.8G   120K  /var/tmp
zroot                    507M  5.32G    24K  /mnt/zroot
zroot/ROOT               507M  5.32G    24K  none
zroot/ROOT/default       507M  5.32G   507M  /mnt
zroot/home              54.5K  5.32G    24K  /mnt/home
zroot/home/minmin       30.5K  5.32G  30.5K  /mnt/home/minmin
zroot/tmp                 24K  5.32G    24K  /mnt/tmp
zroot/usr                 72K  5.32G    24K  /mnt/usr
zroot/usr/ports           24K  5.32G    24K  /mnt/usr/ports
zroot/usr/src             24K  5.32G    24K  /mnt/usr/src
zroot/var                159K  5.32G    24K  /mnt/var
zroot/var/audit           24K  5.32G    24K  /mnt/var/audit
zroot/var/crash           24K  5.32G    24K  /mnt/var/crash
zroot/var/log             39K  5.32G    39K  /mnt/var/log
zroot/var/mail            24K  5.32G    24K  /mnt/var/mail
zroot/var/tmp             24K  5.32G    24K  /mnt/var/tmp
$
```

### LinuxでのZFSミラーの解除

FreeBSDの時と同様に、まずはsdc(USB HDD 320GB)をプールから外してミラーを解除します

```Console
$ zpool detach zroot /dev/sdc4
$ zpool list -v zroot
NAME           SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zroot         99.5G   698M  98.8G        -      132G     0%     0%  1.00x    ONLINE  /mnt
  indirect-0      -      -      -        -         -      -      -      -    ONLINE
  sdb          100G   698M  98.8G        -      132G     0%  0.68%      -    ONLINE
$
```

`zpool list`で見えているのはsdcでしたが、単純にsdcを指定してもエラーになったので、`/dev/sdc4`を指定したところミラーを解除できました。

### Linuxでの小さいパーティションの準備

sdcがOSから外されてフリーの状態になったので、fdiskコマンドで4番目のパーティションサイズを50GBに縮小します。Linuxのfdiskには直接パーティションサイズを縮小するコマンドは無いため、一旦パーティションを削除してあらためて50GBのパーティションを作成しています。

```Console
$ fdisk /dev/sdc

Welcome to fdisk (util-linux 2.38.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

The device contains 'zfs_member' signature and it will be removed by a write command. See fdisk(8) man page and --wipe option for more details.

Command (m for help): p

Disk /dev/sdc: 298.09 GiB, 320072933376 bytes, 625142448 sectors
Disk model:
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: gpt
Disk identifier: 9F3BCC31-DC6D-11EF-A628-0025908D635C

Device       Start       End   Sectors  Size Type
/dev/sdc1       40    532519    532480  260M EFI System
/dev/sdc2   532520    533543      1024  512K FreeBSD boot
/dev/sdc3   534528   4728831   4194304    2G FreeBSD swap
/dev/sdc4  4728832 214444031 209715200  100G FreeBSD ZFS

Command (m for help): d
Partition number (1-4, default 4): 4

Partition 4 has been deleted.

Command (m for help): n
Partition number (4-128, default 4):
First sector (4728832-625142414, default 4728832):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (4728832-625142414, default 625141759): +50G

Created a new partition 4 of type 'Linux filesystem' and of size 50 GiB.

Command (m for help): p
Disk /dev/sdc: 298.09 GiB, 320072933376 bytes, 625142448 sectors
Disk model:
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: gpt
Disk identifier: 9F3BCC31-DC6D-11EF-A628-0025908D635C

Device       Start       End   Sectors  Size Type
/dev/sdc1       40    532519    532480  260M EFI System
/dev/sdc2   532520    533543      1024  512K FreeBSD boot
/dev/sdc3   534528   4728831   4194304    2G FreeBSD swap
/dev/sdc4  4728832 109586431 104857600   50G Linux filesystem

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.

$ 
```

これでsdc4に50GBのパーティションを作成できました。

### Linuxでのプールの一時的拡張

既存のzrootにsdc4の50GBを加えて、一旦150GBのプールとします。

```Console
$ zpool add zroot /dev/sdc4
$ zpool list -v zroot
NAME           SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zroot          149G   699M   148G        -      132G     0%     0%  1.00x    ONLINE  /mnt
  indirect-0      -      -      -        -         -      -      -      -    ONLINE
  sdb          100G   698M  98.8G        -      132G     0%  0.68%      -    ONLINE
  sdc4          50G    64K
$ 
```

### 元の100GBのデバイスの削除

次にプールからsdbを取り除きます。

```Console
$ zpool remove zroot /dev/sdb4
$ zpool list -v zroot
NAME           SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zroot         49.5G   698M  48.8G        -         -     0%     1%  1.00x    ONLINE  /mnt
  indirect-0      -      -      -        -         -      -      -      -    ONLINE
  indirect-1      -      -      -        -      132G      -      -      -    ONLINE
  sdc4          50G   698M  48.8G        -         -     0%  1.37%      -    ONLINE
$
```

sdbを取り除いたことにより、indirect-1のエントリーが残りました。

次にsdb4のサイズをfdiskで50GBに縮小しますが、手順はsdc4と同じなので実行結果は省略します。

### Linuxでのミラー構成の復元

最後にzrootにsdb4を組み込んでZFSミラープールを構成します。

```Console
$ zpool attach zroot sdc4 sdb4
$ zpool list -v zroot
NAME           SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zroot         49.5G   698M  48.8G        -         -     0%     1%  1.00x    ONLINE  /mnt
  indirect-0      -      -      -        -         -      -      -      -    ONLINE
  indirect-1      -      -      -        -      132G      -      -      -    ONLINE
  mirror-3    49.5G   698M  48.8G        -         -     0%  1.37%      -    ONLINE
    sdc4        50G      -      -        -         -      -      -      -    ONLINE
    sdb4        50G      -      -        -         -      -      -      -    ONLINE
$
```

こうして Linux (Debian) でプールサイズを縮小し、最終的に50GBのプールとなりました。
