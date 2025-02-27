# FreeBSDユーザーがsfdiskを使いこなす

## gpart vs fdisk / sfdisk

HDDやSSDのパーティションテーブルを編集するコマンドとして、FreeBSDではgpartを使います(fdiskもありますがFreeBSD 15から廃止)。個人的にgpartは使い勝手の良いコマンドだと思っています。

Linuxでは同様のコマンドにfdiskとsfdiskがあり、fdiskがインタラクティブな操作用、sfdiskがバッチ的な操作用のコマンドです。fdiskで目的とするパーティション操作はできるのですが、どうも使い勝手が悪い(手順が多く表示が冗長)と感じてしまいます。sfdiskはバッチ処理ができスクリプトに組み込む場合にも扱いやすいので、FreeBSDユーザー向きにsfdiskについて説明してみたいと思います。もちろんLinuxユーザーにも役立つと思います。

## sfdiskの基本

gpartでのパーティション操作はコマンドの引数で指定しますが、sfdiskの場合は設定パラメータを標準入力から指定します。コマンドラインでsfdiskを使ってパーティションの変更を行う場合は、次の形の操作が基本となります。

```console
echo XXXXX | sfdisk /dev/sdb
```

これを理解した上で実際に例を使って操作してみます。以降の例は、当然ですがgpartはFreeBSDでの操作で、sfdiskはLinuxでの操作となります。

## パーティションテーブルの初期化

gpartであれば、GPTでパーティションテーブルを初期化する場合は次のようになります。

```console
$ gpart create -s gpt da0
da0 created
$
```

MBR形式の場合は`-s`の引数に`mbr`を指定します。しかし既存のパーティションテーブルが残っている場合は、エラーになりパーティションテーブルの初期化はできません。その場合一旦`gpart destroy`でテーブルの無い状態に初期化します。

```console
$ gpart create -s gpt da0
da0 destroyed
$ gpart create -s gpt da0
da0 created
$
```

sfdiskの場合は次のように標準入力からパーティション形式を指定します。

```console
$ echo "label: gpt" | sfdisk /dev/sdb
Checking that no-one is using this disk right now ... OK

Disk /dev/sdb: 74.53 GiB, 80026361856 bytes, 156301488 sectors
Disk model: 0G9AT00
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

>>> Script header accepted.
>>> Done.
Created a new GPT disklabel (GUID: C05D850D-CDAE-644C-953F-B86544BEC526).

New situation:
Disklabel type: gpt
Disk identifier: C05D850D-CDAE-644C-953F-B86544BEC526

The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
$
```

gpartと違いsfdiskでは、**既存のパーティションテーブルが残っていても初期化**されます。またこのようにsfdiskでのデフォルトでは設定を変更する毎に状況を表示するので、シェルスクリプトで使うような場合は`-q`オプションで出力を抑止するのが良いでしょう。

```console
$ echo "label: gpt" | sfdisk -q /dev/sdb
$
```

またsfdiskには(fdiskにも)パーティションテーブルを完全にクリアする`gpart destroy`に相当する機能はありません。Linuxで同様のことを行うにはwipefsを使います。

```console
$ wipefs -a /dev/sdb
/dev/sdb: 8 bytes were erased at offset 0x00000200 (gpt): 45 46 49 20 50 41 52 54
/dev/sdb: 8 bytes were erased at offset 0x12a1f15e00 (gpt): 45 46 49 20 50 41 52 54
/dev/sdb: 2 bytes were erased at offset 0x000001fe (PMBR): 55 aa
/dev/sdb: calling ioctl to re-read partition table: Success
$
```

## パーティションテーブルの確認

gpartでパーティションテーブルを確認するにはshowコマンドを使います。

```console
$ gpart show da0
 /sbin/gpart show da0
=>       40  312581728  da0  GPT  (149G)
         40       2008       - free -  (1.0M)
       2048   62914560    1  freebsd-zfs  (30G)
   62916608  249665160       - free -  (119G)

$
```

ただshowだけではパーティションラベルを見ることはできません。パーティションラベルを見るためには`-l`オプションを指定します。しかし`-l`の場合はパーティションの型は表示されません。

```console
$ /sbin/gpart show -l  da0
=>       40  312581728  da0  GPT  (149G)
         40       2008       - free -  (1.0M)
       2048   62914560    1  part11  (30G)
   62916608  249665160       - free -  (119G)

$
```

sfdiskでパーティションテーブルを確認するには`-l`(または`--list`)オプションを指定します。

```console
$ sfdisk -l /dev/sdb
Disk /dev/sdb: 74.53 GiB, 80026361856 bytes, 156301488 sectors
Disk model: 0G9AT00
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 86430DEF-2B95-034E-A1CE-FD7A6988554D

Device     Start      End  Sectors Size Type
/dev/sdb1   2048 62916607 62914560  30G Linux filesystem
```

さらに`-q`オプションを追加した場合は、ヘッダー部分は表示されずパーティション情報だけとなります。

```console
$ sfdisk -l -q /dev/sdb
Device     Start      End  Sectors Size Type
/dev/sdb1   2048 62916607 62914560  30G Linux filesystem
```

またsfdiskには`-d`(または`--dump`)でより詳細なパーティションテーブルの情報を表示できます。

```console
$ sfdisk -d /dev/sdb
label: gpt
label-id: 86430DEF-2B95-034E-A1CE-FD7A6988554D
device: /dev/sdb
unit: sectors
first-lba: 2048
last-lba: 156301454
sector-size: 512

/dev/sdb1 : start=        2048, size=    62914560, type=0FC63DAF-8483-4772-8E79-3D69D8477DE4, uuid=61B5D641-7EBA-8842-B63F-FE5D0E766C70, name="part11"
$
```

`-d`はパーティションテーブルを見ることもできますが、本来はこの情報をファイルに保存してバックアップするためのものです。

## パーティションの追加

gpartでパーティションを追加するには`gpart add`を使います。パーティションの追加は未使用領域の最初の部分に割り当てられますが、現在のドライブは論理セクタが512バイト、物理セクタが4096バイトであるため物理セクタサイズに合わせるように必ず`-a 4k`を指定します。ここでは30Gのパーテイションを追加しています。

```console
$ gpart add -t freebsd-zfs -a 4k -s 30g -l data1 da0
da0p1 added
$
```

`-s`でのサイズ指定を省略した場合は、残りの空き領域をすべて割り当てます。

sfdiskでパーティションを追加する場合は、`size=`や`type=`を指定して次のようになります。

```console
$ echo size=30G, type=0FC63DAF-8483-4772-8E79-3D69D8477DE4, name=data1 | sfdisk -a /dev/sdb
Checking that no-one is using this disk right now ... OK

Disk /dev/sdb: 74.53 GiB, 80026361856 bytes, 156301488 sectors
Disk model: 0G9AT00

--- 中略 ---

The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
$
```

`size=`でパーティションサイズをセクタ数またはサイズで指定し、省略した場合は残りの領域全てが割り当てられます。`type`はパーティションの型を指定しますが、ここで指定した`0FC63DAF-8483-4772-8E79-3D69D8477DE4`はLinuxの標準的なファイルシステムを示すGUIDです。`name=`はパーティションのラベル名で、`gpart add`の`-l`に相当します。

`-a`(または`--append`)オプションを指定していますが、sfdiskでは **`-a`が指定されていない場合は既存パーティションテーブルの置き換え** になり、複数のパーティションをすでに用意してあっても指定した1つのパーティションだけに変更されます。ですからパーティションの追加では必ず`-a`を指定します。

