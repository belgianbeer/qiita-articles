<!-- https://qiita.com/belgianbeer/items/477de8ddc64787442c0b -->

# はじめに
ZFSにはファイルシステムそのものにチェックサムが用意されていて、データを読みだす際に常時チェックサムとの整合性を確認してデータが化けいてないかどうかを確認しています。

zpool scrub というコマンドを実行すると、書き込まれているデータ部をすべて読みだしてチェックサムとの整合性を確認します。さらにscrubでエラーが発見された場合には、修復可能(ミラーやRAID-Z等の冗長性ある構成の場合)であればエラーは自動的に修復されます。

ここでは、実験的に書き込んだデータをZFS管理外のところで書き換えることチェックサムとの不整合後を故意に発生させてデータエラーを起こしてみます。

この実験ではFreeBSDを利用しましたが、Ubuntu等のLinux系でのZFSであれば、パーティション作成やディスクのデバイス名が変わりますが、ZFSの操作は同じコマンドでできます。

# 淡々とZFSプールを作って壊してみる

## 実験用のディスクにパーティションを作成する

まずは実験用のプールの作成です。ここでは USB ディスクの da1 で実験を行います。

GPTラベルのディスクを用意します。
```
$ gpart create -s gpt da1
```
4GBのパーティションを作成します。

```
$ gpart add -t freebsd-zfs -a 4k -s 4g -l ttt da1
```

出来上がったパーティションを確認します

```
$ gpart show da1
=>       40  312581728  da1  GPT  (149G)
         40    8388608    1  freebsd-zfs  (4.0G)
    8388648  304193120       - free -  (145G)

$ gpart show -l da1
=>       40  312581728  da1  GPT  (149G)
         40    8388608    1  ttt  (4.0G)
    8388648  304193120       - free -  (145G)

```
## ZFSプールの作成

通常であればディスク容量節約のため ZFSプールを作成する場合 `-O compression=lz4` 等を指定して圧縮機能を有効にするのですが、今回はゼロデータを書き込むため指定しません。指定するとデータの圧縮機能によってデータサイズ分の書き込みが行われなくなってしまうからです。
```
$ zpool create -O atime=off ztest gpt/ttt
```
出来上がったZFSプールを確認します。

```
$ zpool status ztest
  pool: ztest
 state: ONLINE
config:

        NAME        STATE     READ WRITE CKSUM
        ztest       ONLINE       0     0     0
          gpt/ttt   ONLINE       0     0     0

errors: No known data errors
```
## ダミーのデータを書き込む

scrubでデータの整合性の確認が行われるのは、データが実際に書き込まれている部分だけです。書き込まれているデータを壊すために、あらかじめ内容が0のファイルを作成します。

今回用意したZFSプール内のディレクトリに移動します。
```
$ cd /ztest
```
ddコマンドを使って/dev/zeroから読みだしたダミーデータ(つまり0データ)でプール全体を埋めます。

```
$ dd if=/dev/zero of=dummy bs=4m
dd: dummy: No space left on device
928+0 records in
927+1 records out
3891134464 bytes transferred in 157.300331 secs (24736976 bytes/sec)
$ 
```
この通りプールを使い切っています。ここではブロック数を確認したかったので明示的に `-b` オプションを指定しています。


```
$ df -b /ztest
Filesystem 512-blocks    Used Avail Capacity  Mounted on
ztest         7601024 7601024     0   100%    /ztest
```

## データに不整合が起きるように壊してみる

プール内が埋まったところで、ZFSのデータ整合性にエラーが起きるように壊してみます。ZFS経由での変更では壊せませんので、ここではZFS外部からディスクにゴミデータを書き込んでみます。

一旦プールをエクスポートします

```
$ zpool export ztest
```

該当パーティションの適当なところに1ブロック分ゴミデータを書き込みます。

```
$ dd if=/dev/random of=/dev/gpt/ttt count=1 oseek=4000000
1+0 records in
1+0 records out
512 bytes transferred in 0.382043 secs (1340 bytes/sec)
$ 
```

## 壊れたかどうかの確認

プールをインポートして、先ほど作ったファイルを読みだしてみます

```
$ zpool import ztest
$ cat /ztest/dummy > /dev/null
cat: /ztest/dummy: Input/output error
$ 
```
この通りエラーが発生しました。プールの状態は次の通りで一見するとエラーは無いように見えます。

```
$ zpool status ztest
  pool: ztest
 state: ONLINE
config:

        NAME        STATE     READ WRITE CKSUM
        ztest       ONLINE       0     0     0
          gpt/ttt   ONLINE       0     0     0

errors: No known data errors
$ 
```

scrub を実行してみます。

```
$ zpool scrub ztest
$ zpool status ztest
  pool: ztest
 state: ONLINE
  scan: scrub in progress since Mon Aug 29 18:17:33 2022
        3.62G scanned at 412M/s, 550M issued at 61.1M/s, 3.62G total
        0B repaired, 14.81% done, 00:00:51 to go
config:

        NAME        STATE     READ WRITE CKSUM
        ztest       ONLINE       0     0     0
          gpt/ttt   ONLINE       0     0     0

errors: No known data errors
$ 
```

scrub が終了するのを待って、status で確認してみます。

```
$ zpool status ztest
  pool: ztest
 state: ONLINE
status: One or more devices has experienced an error resulting in data
        corruption.  Applications may be affected.
action: Restore the file in question if possible.  Otherwise restore the
        entire pool from backup.
   see: https://openzfs.github.io/openzfs-docs/msg/ZFS-8000-8A
  scan: scrub repaired 0B in 00:01:05 with 1 errors on Mon Aug 29 18:18:32 2022
config:

        NAME        STATE     READ WRITE CKSUM
        ztest       ONLINE       0     0     0
          gpt/ttt   ONLINE       0     0     2

errors: 1 data errors, use '-v' for a list
$
```
この通りCKSUMでエラーが発生していることがわかります。実際に壊れているファイルは -v オプションで確認できます。

```
$ zpool status -v ztest
  pool: ztest
 state: ONLINE
status: One or more devices has experienced an error resulting in data
        corruption.  Applications may be affected.
action: Restore the file in question if possible.  Otherwise restore the
        entire pool from backup.
   see: https://openzfs.github.io/openzfs-docs/msg/ZFS-8000-8A
  scan: scrub repaired 0B in 00:01:05 with 1 errors on Mon Aug 29 18:18:32 2022
config:

        NAME        STATE     READ WRITE CKSUM
        ztest       ONLINE       0     0     0
          gpt/ttt   ONLINE       0     0     2

errors: Permanent errors have been detected in the following files:

        /ztest/dummy
$
```
改めて /ztest/dummy を読みだしてみても、エラーはエラーのままです。

```
$ cat /ztest/dummy > /dev/null
cat: /ztest/dummy: Input/output error
$
```
# 壊した部分の修復と確認

では壊した部分に改めて0を書きこむことで本来のデータに戻してみます。

```
$ zpool export ztest
$ dd if=/dev/zero of=/dev/gpt/ttt count=1 oseek=4000000
1+0 records in
1+0 records out
512 bytes transferred in 0.382715 secs (1338 bytes/sec)
$
```
これで元のデータに書き戻せたはずなので、改めて読みだしてみます。

```
$ zpool import ztest
$ cat /ztest/dummy > /dev/null
$
```
この通りエラーも発生せずに読みだすことができました。zpool status ではエラーメッセージが残ったままになりますが、CKSUMのエラーカウントは0になっています。

```
$ zpool status ztest
  pool: ztest
 state: ONLINE
status: One or more devices has experienced an error resulting in data
        corruption.  Applications may be affected.
action: Restore the file in question if possible.  Otherwise restore the
        entire pool from backup.
   see: https://openzfs.github.io/openzfs-docs/msg/ZFS-8000-8A
  scan: scrub repaired 0B in 00:01:05 with 1 errors on Mon Aug 29 18:18:32 2022
config:

        NAME        STATE     READ WRITE CKSUM
        ztest       ONLINE       0     0     0
          gpt/ttt   ONLINE       0     0     0

errors: 1 data errors, use '-v' for a list
$ 
```
改めて scrub を実行してみます

```
$ zpool scrub ztest
```
scrub が終了したらプールの状態を確認してみます。

```
$ zpool status ztest
  pool: ztest
 state: ONLINE
  scan: scrub repaired 0B in 00:01:05 with 0 errors on Mon Aug 29 18:34:27 2022
config:

        NAME        STATE     READ WRITE CKSUM
        ztest       ONLINE       0     0     0
          gpt/ttt   ONLINE       0     0     0

errors: No known data errors
$
```
この通りエラーは無くなりました。
