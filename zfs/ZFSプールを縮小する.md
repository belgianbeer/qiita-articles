<!-- 
https://blog.naa0yama.com/p/02w17-srcjj0yk/
-->
# ZFSプールを縮小する

## はじめに

ZFSのミラー構成の容量を大きいものに交換する手順は以前に「[ZFSミラープールのディスク交換による容量増設](https://qiita.com/belgianbeer/items/8df197588462cd7f6b45)」で紹介しました。今回は逆で、プールのサイズを小さくする例を紹介します。実際に使うことは少ないかもしれませんが、500GBクラスのSSDを使ってるシステムで240GBクラスに変更したい場合等には役に立つと思います。

## ZFSでのプールの縮小

ZFSプールを扱うzpoolコマンドにはaddとremoveというサブコマンドがあります。`zpool add`は指定したデバイスを結合してプールサイズを拡張し、`zpool remove`はプールから指定したデバイスを取り除くコマンドです。これらを使うことでプールのサイズを縮小できます(もちろんプールの使用量より小さくは出来ません)。

例えば500GBのプールサイズの使用量が200GB程度なので250GBのプールサイズに縮小したい場合、次の手順で行います。

1. 別のデバイスで250GBのパーティションを用意する。
2. 現在のプールに用意した250GBのデバイスを`zpool add`で追加する。この時点でプールのサイズは一時的に750GBとなる。
3. このプールから500GBのデバイスを`zpool remove`で取り除くと、プールサイズは250GBとなる。

ここではプールサイズを小さくする場合を例にしていますが、同様の手順でプールサイズの拡大もできます。

通常ZFSでドライブを交換する場合 `zfs send`と`zfs receive`を組み合わせてデータコピーを行うことも多いですが、send/receiveでは別デバイスへのファイルシステムの複製となるため作業完了後デバイスの入れ替えが必要になるのに対し、本手順では**ファイルシステムは変更なくOSに接続されたままサイズ変更やデバイス交換が行える**のがポイントです。動作中のプログラムには影響を与えないので、例え対象がシステムディスクであってもOSを再起動することなくプールサイズを変更できます。

## ZFSルートミラープールのサイズ変更の実例

ZFSプールの縮小方法を理解したところで、実際にルートファイルシステムをZFSミラーで構成したシステムのプールサイズの縮小をやってみます。

### ZFSミラールートの構成

次のzrootはFreeBSDのインストーラでRoot-on-ZFSでインストールしたシステムディスクをさらにミラーで構成したものです。ここではda0、da1とも250GBのHDDを使っています。

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

パーティションの様子 (da0とda1は同じサイズのストレージ)

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

da1p4がOSから外されてフリーの状態になったので、`gpart resize`を使って小さいパーティションを用意します。ここでは100GBにします。

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
  214444032  273953096       - free -  (131G)
$
```

### プールの一時的な拡張

da1p4が100GBになったので、`gpart add`でzrootに追加して一時的にプールを拡張します(約330GB)。

```Console
$ zpool add zroot da1p4
$ zpool list -v
NAME        SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zroot       330G   699M   329G        -         -     0%     0%  1.00x    ONLINE  -
  da0p4     231G   699M   229G        -         -     0%  0.29%      -    ONLINE
  da1p4     100G    68K  99.5G        -         -     0%  0.00%      -    ONLINE
$ 
```

この通りzrootはda0p4とda1p4で構成された330GBのプールとなっていることがわかります。

### zpool removeの実行

拡張したプールから`zpool remove`を使ってda0p4を一旦プールから外します。

```Console
$ zpool remove zroot da0p4
$ 
```

コマンドラインだけだとすぐに終了しているように見えますが、実際は内部でデータコピーが行われるのでスとレージの使用量と転送速度に応じた時間がかかり、しばらく待たされます。

zpool removeの実行中に別途ログインして`zpool status`を実行すると、次のようにデバイス削除のためのresilverが行われていることがわかります。

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

パーティションサイズは100GBに縮小されていますが、indirect-0というエントリーが見えます。どうやらindirect-0はda0p4が使われていた残骸のようで、現状消すことはできないようです。

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

`zpool attach`はすぐに終了しますが、実際には内部でミラーの同期が行われ、同期が完了すれば作業終了となります。

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

