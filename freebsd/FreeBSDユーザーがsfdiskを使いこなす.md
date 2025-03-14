# FreeBSDユーザーがsfdiskを使いこなす

## gpart vs fdisk / sfdisk

HDDやSSDのパーティションテーブルを編集するコマンドとして、FreeBSDではgpartを使います(fdiskもありますがFreeBSD 15から廃止)。個人的にgpartは使い勝手の良いコマンドだと思っています。

Linuxでは同様のコマンドにfdiskとsfdiskがあり、fdiskがインタラクティブな操作用、sfdiskがバッチ的な操作用のコマンドです。FreeBSDのgpartに慣れている筆者にとってfdiskで目的とするパーティション操作はできるのですが、どうも使い勝手が悪いと感じてしまいます。sfdiskはバッチ処理ができスクリプトに組み込む場合にも扱いやすいので、FreeBSDユーザー向きにgpartと比較しながらsfdiskについて説明してみたいと思います。もちろんLinuxユーザーにも役立つと思います。

## sfdiskの基本

gpartでのパーティション操作はコマンドの引数で指定しますが、sfdiskの場合は設定パラメータを標準入力から指定します。コマンドラインでsfdiskを使ってパーティションの変更を行う場合は、次の形の操作が基本となります。

fileを用意する

```console
$ vi file
$ sfdisk /dev/sdb < file
```

echo で直接指定する

```console
echo XXXXX | sfdisk /dev/sdb
```

shellのヒアドキュメントを使う

```console
sfdisk /dev/sdb < EOT
XXXXXX
EOT
```

sfdiskへ入力する行はヘッダーとフィールド部に分類でき、それぞれ独立に設定できます。

以上を理解した上で実際に例を使って操作してみます。以降の例は、(当然ですが)gpartはFreeBSDでの操作で、sfdiskはLinuxでの操作となります。

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

sfdiskの場合次のように標準入力から`label`ヘッダーでパーティション形式を指定します。

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

gpartと違いsfdiskでは、**既存のパーティションテーブルが残っていても初期化**されます。またこのようにsfdiskでのデフォルトでは設定を変更する毎に状況を表示するので、シェルスクリプトで使うような場合は`--quiet`(または`-q`)オプションで出力を抑止するのが良いでしょう。

```console
$ echo "label: gpt" | sfdisk --quiet /dev/sdb
$
```

またsfdiskには(fdiskにも)パーティションテーブルを完全にクリアする`gpart destroy`に相当する機能はありません。Linuxで同様のことを行うにはwipefsを使います。

```console
$ wipefs --all /dev/sdb
/dev/sdb: 8 bytes were erased at offset 0x00000200 (gpt): 45 46 49 20 50 41 52 54
/dev/sdb: 8 bytes were erased at offset 0x12a1f15e00 (gpt): 45 46 49 20 50 41 52 54
/dev/sdb: 2 bytes were erased at offset 0x000001fe (PMBR): 55 aa
/dev/sdb: calling ioctl to re-read partition table: Success
$
```

## パーティションテーブルの表示

gpartでパーティションテーブルを確認するにはshowコマンドを使います。

```console
$ gpart show da0
=>       40  312581728  da0  GPT  (149G)
         40       2008       - free -  (1.0M)
       2048   62914560    1  freebsd-zfs  (30G)
   62916608  249665160       - free -  (119G)

$
```

ただshowだけではパーティションラベルを見ることはできず、パーティションラベルを見るためには`-l`オプションを指定します。しかし`-l`の場合はパーティションの型が表示されなくなります。

```console
$ gpart show -l da0
=>       40  312581728  da0  GPT  (149G)
         40       2008       - free -  (1.0M)
       2048   62914560    1  part11  (30G)
   62916608  249665160       - free -  (119G)

$
```

sfdiskでパーティションテーブルを確認するには`--list`(または`-l`)オプションを指定します。

```console
$ sfdisk --list /dev/sdb
Disk /dev/sdb: 74.53 GiB, 80026361856 bytes, 156301488 sectors
Disk model: 0G9AT00
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 86430DEF-2B95-034E-A1CE-FD7A6988554D

Device     Start      End  Sectors Size Type
/dev/sdb1   2048 62916607 62914560  30G Linux filesystem
$
```

さらに`--quiet`オプションを追加した場合は、ヘッダー部分は表示されずパーティション情報だけとなります。

```console
$ sfdisk --list --quiet /dev/sdb
Device     Start      End  Sectors Size Type
/dev/sdb1   2048 62916607 62914560  30G Linux filesystem
$
```

またsfdiskには`--dump`(または`-d`)でより詳細なパーティションテーブルの情報を表示できます。

```console
$ sfdisk --dump /dev/sdb
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

`--dump`はパーティションテーブルを見ることもできるわけですが、出力をファイルに保存してパーティションテーブルのバックアップするオプションです(後述)。

またgpartで未使用領域を` - free - `として示しますが、sfdiskでは未使用領域は表示しません。sfdiskには未使用領域を表示するための`--list-fre`(または`-F`)が用意されていて、未使用領域だけの表示となります。

```console
$ sfdisk --list-free --quiet /dev/sdb
   Start       End  Sectors  Size
62916608 156301454 93384847 44.5G
$
```

## パーティションの追加

gpartでパーティションを追加するには`gpart add`を使います。パーティションの追加は未使用領域の最初の部分に割り当てられますが、現在のストレージの物理セクタはほとんどが4096バイトであるため(論理セクタサイズは従来同様512バイト)物理セクタサイズに合わせるように`-a 4k`を指定します。ここでは30Gのパーテイションを追加しています。

```console
$ gpart add -t freebsd-zfs -a 4k -s 30g -l data1 da0
da0p1 added
$
```

`-s`のサイズ指定を省略した場合は、残りの空き領域をすべて割り当てます。

sfdiskでパーティションを追加する場合は、`size=`や`type=`を指定して次のようになります。

```console
$ echo size=30G, type=0FC63DAF-8483-4772-8E79-3D69D8477DE4, name=data1 | sfdisk --append /dev/sdb
Checking that no-one is using this disk right now ... OK

Disk /dev/sdb: 74.53 GiB, 80026361856 bytes, 156301488 sectors
Disk model: 0G9AT00

--- 中略 ---

The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
$
```

`size=`等のパラメータはカンマで区切って指定します。

- `size=` パーティションサイズをセクタ数またはサイズで指定する。省略した場合は残りの領域全てが割り当てられる。
- `type=` パーティションの型を指定する。上記例で指定した`0FC63DAF-8483-4772-8E79-3D69D8477DE4`は、Linuxのファイルシステムを示すGUIDです。
- `name=` パーティションのラベル名を設定する。

他にもありますが、それらは`man sfdisk`で確認してください。

`--append`(または`-a`)オプションを指定していますが、sfdiskでは **`--append`が指定されていない場合は既存パーティションテーブルの置き換えになり、複数のパーティションテーブルがあっても指定した1つのパーティションだけに変更されます**。ですからパーティションの追加では必ず`--append`を指定するのを忘れないようにします。

追加では無く特定のパーティションを指定する場合、`gpart add`では`-i`でパーティションインデックスを指定しますが、`sfdisk`では`-N`で指定します。

```console
$ gpart add -t freebsd-zfs -a 4k -s 30g -l data3 -i 3 da0
da0p3 added
$ gpart show da0
=>       40  312581728  da0  GPT  (149G)
         40   62914560    3  freebsd-zfs  (30G)
   62914600  249667168       - free -  (119G)

$
```

```console
$ echo size=30G, type=0FC63DAF-8483-4772-8E79-3D69D8477DE4, name=data3 | sfdisk -N 3 --quiet /dev/sdb
$ sfdisk --list --quiet /dev/sdb
Device        Start       End  Sectors Size Type
/dev/sdb3  62916608 125831167 62914560  30G Linux filesystem
$
```

ところで`type=`ではパーティションの型を示すGUIDを指定する必要がありますが、この一覧は`--list-type`(または`-T`)オプションを使って``sfdisk --list-type --label gpt`で確認できます。次の例のようにFreeBSD用パーティションのGUIDも一式用意されています。

```console
$ sfdisk --list-type --label gpt | grep -i freebsd
516E7CB4-6ECF-11D6-8FF8-00022D09712B  FreeBSD data
83BD6B9D-7F41-11DC-BE0B-001560B84F0F  FreeBSD boot
516E7CB5-6ECF-11D6-8FF8-00022D09712B  FreeBSD swap
516E7CB6-6ECF-11D6-8FF8-00022D09712B  FreeBSD UFS
516E7CBA-6ECF-11D6-8FF8-00022D09712B  FreeBSD ZFS
516E7CB8-6ECF-11D6-8FF8-00022D09712B  FreeBSD Vinum
$
```

sfdiskでは複数のパーティションも同時に作成できます。

```console
$ sfdisk --quiet /dev/sdb << EOT
label: gpt
size=30G,name=part11
size=20G,type=516E7CBA-6ECF-11D6-8FF8-00022D09712B,name=freebsdzfs
EOT
$ sfdisk --quiet --list /dev/sdb
Device        Start       End  Sectors Size Type
/dev/sdb1      2048  62916607 62914560  30G Linux filesystem
/dev/sdb2  62916608 104859647 41943040  20G FreeBSD ZFS
$ sfdisk --dump /dev/sdb
label: gpt
label-id: 90DCD5DA-40FA-6F43-B523-905BC9757FBB
device: /dev/sdb
unit: sectors
first-lba: 2048
last-lba: 156301454
sector-size: 512

/dev/sdb1 : start=        2048, size=    62914560, type=0FC63DAF-8483-4772-8E79-3D69D8477DE4, uuid=9F35EE97-5367-5847-89BC-C1BC3B02C071, name="part11"
/dev/sdb2 : start=    62916608, size=    41943040, type=516E7CBA-6ECF-11D6-8FF8-00022D09712B, uuid=7ABE350B-CC00-3648-9C55-4934830B44A5, name="freebsdzfs"
$
```

入力行でパーティションインデックスを指定する場合は、行頭にデバイス名を「:」(コロン)で区切って設定します。

```console
$ sfdisk --quiet /dev/sdb << EOT
label: gpt
/dev/sdb1: size=30G,name=part11
/dev/sdb3: size=20G,type=516E7CBA-6ECF-11D6-8FF8-00022D09712B,name=freebsdzfs
EOT
$ sfdisk --list --quiet /dev/sdb
Device        Start       End  Sectors Size Type
/dev/sdb1      2048  62916607 62914560  30G Linux filesystem
/dev/sdb3  62916608 104859647 41943040  20G FreeBSD ZFS
$
```

## パーティションの削除

gpartでパーティションを削除する場合は次のように delete に続く`-i`でパーティションインデックスを指定します。

```console
$ gpart show da0
=>       40  312581728  da0  GPT  (149G)
         40   62914560    1  freebsd-ufs  (30G)
   62914600   41943040    3  freebsd-zfs  (20G)
  104857640  207724128       - free -  (99G)

$ gpart delete -i 1 da0
da0p1 deleted
$ gpart show da0
=>       40  312581728  da0  GPT  (149G)
         40   62914560       - free -  (30G)
   62914600   41943040    3  freebsd-zfs  (20G)
  104857640  207724128       - free -  (99G)

$
```

sfdiskでは`--delete`オプションで、デバイス名の後に削除するパーティションインデックスを指定します。

```console
$ sfdisk --quiet --list /dev/sdb
Device        Start       End  Sectors Size Type
/dev/sdb1      2048  62916607 62914560  30G Linux filesystem
/dev/sdb3  62916608 104859647 41943040  20G FreeBSD ZFS
$ sfdisk --quiet --delete /dev/sdb 1
$ sfdisk --quiet --list /dev/sdb
Device        Start       End  Sectors Size Type
/dev/sdb3  62916608 104859647 41943040  20G FreeBSD ZFS
$
```

## パーティションテーブルの修正

gpartでパーティションテーブルを修正する場合`gpart modify`を使い、`-l`でパーティションラベル、`-t`はパーティションの型、`-s`でサイズの変更ができます。

```console
$ gpart show -l da0
=>       40  312581728  da0  GPT  (149G)
         40   62914560    3  data3  (30G)
   62914600  249667168       - free -  (119G)

$ gpart modify -i 3 -l part3 da0
da0p3 modified
$ gpart show -l da0
=>       40  312581728  da0  GPT  (149G)
         40   62914560    3  part3  (30G)
   62914600  249667168       - free -  (119G)

$
```

sfdiskでは、`--part-label`、`--part-type`、`--part-uuid`等を使います。

```console
$ sfdisk --quiet --list /dev/sdb
Device     Start      End  Sectors Size Type
/dev/sdb1   2048 62916607 62914560  30G Linux filesystem
$ sfdisk --quiet --part-type /dev/sdb 1     # パーティションタイプの確認
0FC63DAF-8483-4772-8E79-3D69D8477DE4
$ sfdisk --quiet --part-type /dev/sdb 1 516E7CBA-6ECF-11D6-8FF8-00022D09712B # 変更するパーティションタイプを指定する
$ sfdisk --quiet --list /dev/sdb
Device     Start      End  Sectors Size Type
/dev/sdb1   2048 62916607 62914560  30G FreeBSD ZFS
$
```

sfdiskにはサイズを変更するオプションは用意されていませんが、`--dump`の出力結果を編集し、それをsfdiskで適応すればサイズ変更も出来ます。

```console
$ sfdisk --dump /dev/sdb | tee /tmp/sdb-dump
label: gpt
label-id: BEAA2987-0A6E-FA47-9128-46A30414AEC9
device: /dev/sdb
unit: sectors
first-lba: 2048
last-lba: 156301454
sector-size: 512

/dev/sdb1 : start=        2048, size=    62914560, type=516E7CBA-6ECF-11D6-8FF8-00022D09712B, uuid=61B5D641-7EBA-8842-B63F-FE5D0E766C70, name="part11"
$ vi /tmp/sdb-dump        # size=62914560 を size=20G に書き換える

$ cat /tmp/sdb-dump
label: gpt
label-id: BEAA2987-0A6E-FA47-9128-46A30414AEC9
device: /dev/sdb
unit: sectors
first-lba: 2048
last-lba: 156301454
sector-size: 512

/dev/sdb1 : start=        2048, size=20G, type=516E7CBA-6ECF-11D6-8FF8-00022D09712B, uuid=61B5D641-7EBA-8842-B63F-FE5D0E766C70, name="part11"
$
```

変更したパーティションテーブルをsfdiskでストレージに書き戻します。

```console
$ sfdisk --quiet /dev/sdb < /tmp/sdb-dump
$ sfdisk --quiet --list /dev/sdb
Device     Start      End  Sectors Size Type
/dev/sdb1   2048 41945087 41943040  20G FreeBSD ZFS
$
```

ここではサイズを修正しただけでしたが、他の項目も必要に応じて同時に修正できます。

## まとめ

個人的に単純にパーティションを作るだけならgpartのほうが使いやすいのですが、[凝った修正を行う場合](https://qiita.com/belgianbeer/items/b638c12150dc86911922)はsfdiskの方が便利です。特にgpartにはパーティションのuuidやアトリビュートを保存したり修正したりする機能が無いのが残念で、その点でsfdiskはパーティションテーブルの全てのパラメータをバックアップして編集できます。
