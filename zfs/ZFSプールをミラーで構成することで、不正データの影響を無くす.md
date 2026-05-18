<!-- https://qiita.com/belgianbeer/items/0e69cf3c3f0fc3c89adc -->

# はじめに

前回投稿した「[ZFSプールを壊してみた](https://qiita.com/belgianbeer/items/477de8ddc64787442c0b)」という記事で、ZFSではディスク上の不正データをちゃんと検出していることがわかりました。

不正データが検出できることは、検出できないファイルシステムの不正データを読み込んで誤動作をするよりははるかにましです。しかし不正データが検出できるのであれば、それをリカバーすることはできないのか？と思うのも自然なことでしょう。

ZFSの場合は冗長性のあるプールを構成すれば、不正なデータが存在しても正しいデータを得ることができます。具体的にはミラーやRAID-Z、RAID-Z2などのプールを利用します。

ここでは[ZFSプールを壊してみた](https://qiita.com/belgianbeer/items/477de8ddc64787442c0b)と同様の実験を、ミラー構成のZFSプールで試してみます。

**2022-12-26 scrub実行後のディスクの状態を追記しました**

# ZFSのミラープールを用意する

実験のために2台のUSB接続のHDDを用意して、それぞれに 4GB のパーティションを作成してミラー構成のZFSプールを用意します。

![DSCF2553.JPG](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/130638/789a1ff3-ea8a-ab11-3f7b-f3db1c5364cb.jpeg)


HDDは/dev/da1, /dev/da2で、GPTでパーティションを作成し、それぞれに tt1 と tt2 のラベルのパーティションを作成しｓます。

```
$ gpart create -s gpt da1
da1 created
$ gpart create -s gpt da2
da2 created
$ gpart add -t freebsd-zfs -a 4k -s 4g -l tt1 da1
da1p1 added
$ gpart add -t freebsd-zfs -a 4k -s 4g -l tt2 da2
da2p1 added
$
$ gpart show da1
=>       40  488397088  da1  GPT  (233G)
         40    8388608    1  freebsd-zfs  (4.0G)
    8388648  480008480       - free -  (229G)

$ gpart show da2
=>       40  312581728  da2  GPT  (149G)
         40    8388608    1  freebsd-zfs  (4.0G)
    8388648  304193120       - free -  (145G)

$ gpart show -l da1
=>       40  488397088  da1  GPT  (233G)
         40    8388608    1  tt1  (4.0G)
    8388648  480008480       - free -  (229G)

$ gpart show -l da2
=>       40  312581728  da2  GPT  (149G)
         40    8388608    1  tt2  (4.0G)
    8388648  304193120       - free -  (145G)

$
```

これで、ミラー用に2つの2台のHDDにそれぞれパーティションができました。

この2つのパーティションを組み合わせてZFSミラーのプールを作成します。もちろん[前回の実験](https://qiita.com/belgianbeer/items/477de8ddc64787442c0b)と同様に、ZFSの圧縮は行いません。

```
$ zpool create -O atime=off ztest mirror gpt/tt1 gpt/tt2
$
```
次の通り、ミラー構成のZFSプールができました。

```
$ zpool status ztest
  pool: ztest
 state: ONLINE
config:

	NAME	     STATE     READ WRITE CKSUM
	ztest	     ONLINE	  0	0     0
	  mirror-0   ONLINE	  0	0     0
	    gpt/tt1  ONLINE	  0	0     0
	    gpt/tt2  ONLINE	  0	0     0

errors: No known data errors

$ zpool list -v ztest
NAME          SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
ztest        3.75G  3.62G   128M        -         -    72%    96%  1.00x    ONLINE  -
  mirror-0   3.75G  3.62G   128M        -         -    72%  96.7%      -    ONLINE
    gpt/tt1      -      -      -        -         -      -      -      -    ONLINE
    gpt/tt2      -      -      -        -         -      -      -      -    ONLINE

$ zfs list ztest
NAME	USED  AVAIL	REFER  MOUNTPOINT
ztest	420K  3.62G	  96K  /ztest

$ df /ztest/
Filesystem 512-blocks Used   Avail Capacity  Mounted on
ztest         7601576  192 7601384     0%    /ztest

```

# ダミーデータを書き込む

それでは前回の実験と同様にddコマンドを使ってファイルシステムをゼロデータで埋めます。

```
$ dd if=/dev/zero of=/ztest/dummy bs=1m
dd: /ztest/dummy: No space left on device
3711+0 records in
3710+1 records out
3891134464 bytes transferred in 151.249101 secs (25726662 bytes/sec)

$ df -h /ztest/
Filesystem    Size    Used   Avail Capacity  Mounted on
ztest         3.6G    3.6G      0B   100%    /ztest

$ zfs list -o space ztest
NAME   AVAIL   USED  USEDSNAP  USEDDS  USEDREFRESERV  USEDCHILD
ztest     0B  3.63G        0B   3.62G             0B       672K

$ ls -l /ztest/dummy
-rw-r--r--  1 root  wheel  3891134464 Dec 22 17:43 /ztest/dummy
```

# 書き込んだデータの破壊

次にゴミのデータを無理やり書き込んで、ZFSとしてはエラーになるように細工します。ミラーによる冗長性とZFSの修復を確認するためですので、da1 側のみにゴミデータを書き込みます。

通常ミラーで構成されたディスクからの単純な読み出しは、負荷分散と高速化のために両方のディスクを交互にアクセスしますが、ZFSのミラー構成でも同様の挙動となります。そのため前回のように1ブロック程度の破壊では、読み出し時に正常なドライブ側をアクセスして、該当のブロックをアクセスしない可能性があります。そこで今回はより多くの領域のデータでエラーを起こすよう数十MB程度のデータを書き込みます。ちょうどカーネルが30MB近くありますのでこれを書き込んで壊してみます。

```
$ zpool export ztest
$ ls -l /boot/kernel/kernel
-r-xr-xr-x  2 root  wheel  29343392 Nov  4 10:27 /boot/kernel/kernel
$ dd if=/boot/kernel/kernel of=/dev/gpt/tt1 oseek=1000000
dd: /dev/gpt/tt1: Invalid argument
57311+1 records in
57311+0 records out
29343232 bytes transferred in 21.647581 secs (1355497 bytes/sec)
$
```

# 壊したデータ部分の読み出し

エラーが起きることを確認するため、da1側のみを使ってインポートします。USBのHDDなので、da2を物理的に外してしまうのが簡単です。

![DSCF2555.JPG](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/130638/bd2d7cd7-fc2f-2fd1-4389-d554d26c438d.jpeg)


```
$ zpool import
   pool: ztest
     id: 6304409293823647838
  state: DEGRADED
status: One or more devices are missing from the system.
 action: The pool can be imported despite missing or damaged devices.  The
        fault tolerance of the pool may be compromised if imported.
   see: https://openzfs.github.io/openzfs-docs/msg/ZFS-8000-2Q
 config:

        ztest        DEGRADED
          mirror-0   DEGRADED
            gpt/tt1  ONLINE
            gpt/tt2  UNAVAIL  cannot open
```
この通り da2側のgpt/tt2 にアクセスできません。このまま ztest をインポートします。
```
$ zpool import ztest
$ zpool status ztest
  pool: ztest
 state: DEGRADED
status: One or more devices could not be opened.  Sufficient replicas exist for
        the pool to continue functioning in a degraded state.
action: Attach the missing device and online it using 'zpool online'.
   see: https://openzfs.github.io/openzfs-docs/msg/ZFS-8000-2Q
config:

        NAME                      STATE     READ WRITE CKSUM
        ztest                     DEGRADED     0     0     0
          mirror-0                DEGRADED     0     0     0
            gpt/tt1               ONLINE       0     0     0
            13953643250226400519  UNAVAIL      0     0     0  was /dev/gpt/tt2

errors: No known data errors

$ df /ztest
Filesystem 512-blocks    Used Avail Capacity  Mounted on
ztest         7601264 7601024   240   100%    /ztest
$ ls /ztest
dummy
#
```

片方のドライブだけですが、/ztest/dummy がちゃんとあります。ではこれを実際に読み込んでみましょう。読み込みにはデータの内容がわかるように hd コマンドを使います。

```
$ hd /ztest/dummy
00000000  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
hd: /ztest/dummy: Input/output error
9e1a0000
$
```
ファイルを読み込むことができず、エラーが発生しました。

# ミラー構成での読み出し


改めてミラー構成に戻して同様のことを行ってみます。

```
$ zpool export ztest
```
ここで外してあった da2 側の USB HDD を接続します。

```
$ zpool import
   pool: ztest
     id: 6304409293823647838
  state: ONLINE
 action: The pool can be imported using its name or numeric identifier.
 config:

        ztest        ONLINE
          mirror-0   ONLINE
            gpt/tt1  ONLINE
            gpt/tt2  ONLINE
```
gpt/tt2側がONLINEになっています。それではztestをインポートします。

```
$ zpool import ztest
$ zpool status ztest
  pool: ztest
 state: ONLINE
config:

        NAME         STATE     READ WRITE CKSUM
        ztest        ONLINE       0     0     0
          mirror-0   ONLINE       0     0     0
            gpt/tt1  ONLINE       0     0     0
            gpt/tt2  ONLINE       0     0     0

errors: No known data errors
$
```
正常にインポートできました、では先ほどと同じように /ztest/dummy を読み出してみましょう
```
$ hd /ztest/dummy
00000000  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
e7ee0000
$
```
この通り正常に読み出すことができ、すべての内容が0であることが確認できました。ztestの状態を確認してみると次の通りで問題無いように見えます。
```
$ zpool status ztest
  pool: ztest
 state: ONLINE
  scan: resilvered 432K in 00:00:01 with 0 errors on Fri Dec 23 11:14:23 2022
config:

        NAME         STATE     READ WRITE CKSUM
        ztest        ONLINE       0     0     0
          mirror-0   ONLINE       0     0     0
            gpt/tt1  ONLINE       0     0     0
            gpt/tt2  ONLINE       0     0     0

errors: No known data errors
$
```

# scrubによるディスクの整合性の確認
ZFS的には読み出せない部分があるはずなので、scrubを実行して整合性の確認を実施します。

```
$ zpool scrub ztest
$ zpool status ztest
  pool: ztest
 state: ONLINE
  scan: scrub in progress since Fri Dec 23 11:18:02 2022
        3.62G scanned at 530M/s, 378M issued at 54.0M/s, 3.62G total
        0B repaired, 10.18% done, 00:01:01 to go
config:

        NAME         STATE     READ WRITE CKSUM
        ztest        ONLINE       0     0     0
          mirror-0   ONLINE       0     0     0
            gpt/tt1  ONLINE       0     0     0
            gpt/tt2  ONLINE       0     0     0

errors: No known data errors
$
```
scrubが終了したのを見計らって、状態を見てみます。
```
$ zpool status ztest
  pool: ztest
 state: ONLINE
status: One or more devices has experienced an unrecoverable error.  An
        attempt was made to correct the error.  Applications are unaffected.
action: Determine if the device needs to be replaced, and clear the errors
        using 'zpool clear' or replace the device with 'zpool replace'.
   see: https://openzfs.github.io/openzfs-docs/msg/ZFS-8000-9P
  scan: scrub repaired 15.8M in 00:01:06 with 0 errors on Fri Dec 23 11:19:08 2022
config:

        NAME         STATE     READ WRITE CKSUM
        ztest        ONLINE       0     0     0
          mirror-0   ONLINE       0     0     0
            gpt/tt1  ONLINE       0     0   130
            gpt/tt2  ONLINE       0     0     0

errors: No known data errors
$
```
da1側のtt1は盛大にチェックサムエラーが発生しているのがわかります。

# 改めてデータの読み出し

最後に改めてデータを読み出してみます。OSのディスクバッファの影響を除くため、いったんエクスポートしてインポートしてから実行します。
```
$ zpool export ztest
$ zpool import ztest
$ hd /ztest/dummy
00000000  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
e7ee0000
$
```
この通りエラーが起きているディスクにも拘わらず、ミラー構成のおかげで正しいデータを読み出すことができました。

# 【追記】scrub実行後のda1の状態

記事公開後に「scrubによってda1側は修復されて読み出せるようになっているのでは？」という質問をいただきました。実際 scrub実行後は`scan: scrub repaired`と表示してますから、確認認してみました。

といっても実験環境はすでに破棄してしまったので、改めてのやり直しで一部省略してda1側のデータを壊しscrub後にda1側を読み出すことをやってみます。

ZFSミラープールの作成からda1側のデータを破壊しscrub実行後にプールをエクスポートするところまで。
```
$ zpool create -O atime=off ztest mirror gpt/tt1 gpt/tt2
$ dd if=/dev/zero of=/ztest/dummy bs=1m
dd: /ztest/dummy: No space left on device
3711+0 records in
3710+1 records out
3891134464 bytes transferred in 153.170356 secs (25403966 bytes/sec)
$ zpool export ztest
$ dd if=/boot/kernel/kernel of=/dev/gpt/tt1 oseek=1000000
dd: /dev/gpt/tt1: Invalid argument
57311+1 records in
57311+0 records out
29343232 bytes transferred in 21.643263 secs (1355767 bytes/sec)
$ zpool import ztest
$ zpool scrub ztest
$ zpool status ztest
  pool: ztest
 state: ONLINE
status: One or more devices has experienced an unrecoverable error.  An
        attempt was made to correct the error.  Applications are unaffected.
action: Determine if the device needs to be replaced, and clear the errors
        using 'zpool clear' or replace the device with 'zpool replace'.
   see: https://openzfs.github.io/openzfs-docs/msg/ZFS-8000-9P
  scan: scrub repaired 27.9M in 00:01:06 with 0 errors on Mon Dec 26 10:55:49 2022
config:

        NAME         STATE     READ WRITE CKSUM
        ztest        ONLINE       0     0     0
          mirror-0   ONLINE       0     0     0
            gpt/tt1  ONLINE       0     0   227
            gpt/tt2  ONLINE       0     0     0

errors: No known data errors
$ zpool export ztest
```
エクスポートできたので、da2側を物理的に外してda1側をインポートして読み出してみます。
```
$ zpool import
   pool: ztest
     id: 11275383091719095959
  state: DEGRADED
status: One or more devices are missing from the system.
 action: The pool can be imported despite missing or damaged devices.  The
        fault tolerance of the pool may be compromised if imported.
   see: https://openzfs.github.io/openzfs-docs/msg/ZFS-8000-2Q
 config:

        ztest        DEGRADED
          mirror-0   DEGRADED
            gpt/tt1  ONLINE
            gpt/tt2  UNAVAIL  cannot open
$ zpool import ztest
$ hd /ztest/dummy
00000000  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
e7ee0000
$ zpool export ztest

```
破壊した側のda1のデータを問題無く全データ読み出すことができました。つまりscrubによって書き戻されて修復できています。また実際にインポート後にstatusで確認したところ CKSUMの値も0でした(スクショとりそこなった…)。

改めて、エクスポートしたztestにda2を接続してインポートしてみます。
```
$ zpool import ztest
$ zpool status ztest
  pool: ztest
 state: ONLINE
  scan: scrub repaired 27.9M in 00:01:06 with 0 errors on Mon Dec 26 10:55:49 2022
config:

        NAME         STATE     READ WRITE CKSUM
        ztest        ONLINE       0     0     0
          mirror-0   ONLINE       0     0     0
            gpt/tt1  ONLINE       0     0     0
            gpt/tt2  ONLINE       0     0     0

errors: No known data errors
$
```
この通りエラーはクリアされています。

# まとめ

ZFSではディスクをミラーやRAID-Z構成にすることで、一部のデータエラーの回避ができることを確認できました。またscrubの実行によって修復できるデータに関しては元通りに修復されることもわかりました。

ZFSに限らず他のボリュームマネージャーやRAIDシステムでも、正しくミラーやRAIDを組むことによって同様に一部のデータエラーを回避できます。修復までできるかどうかについては実装に依存しそうです。

