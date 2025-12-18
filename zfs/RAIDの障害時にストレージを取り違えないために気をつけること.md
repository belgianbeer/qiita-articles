<!-- https://qiita.com/belgianbeer/items/3b927310c206c3d9f78c -->
# RAIDの障害時にストレージを取り違えないために気をつけること

## はじめに

ここで記述する内容は過去に投稿したZFS関連の投稿で何度も書いていることですが、最近知人で実際にハマった話を聞いたので、改めて独立した記事にまとめてみます。

## RAIDを構成する

RAIDを構成する場合、当然ですがストレージのデバイス名を指定します。ここではシステム用とは別のデータ用に同容量の3台のストレージでRAID 5を構成する場合を例にします。FreeBSDの場合3台のストレージのデバイス名はSATA接続で/dev/ada1～/dev/ada3、Linuxの場合は/dev/sdb～/dev/sddです。

RAID構成ストレージのデバイス名:

| RAID構成 | FreeBSD     | Linux     |
|----------|-------------|-----------|
| 1台目    | /dev/ada1   | /dev/sdb  |
| 2台目    | /dev/ada2   | /dev/sdc  |
| 3台目    | /dev/ada3   | /dev/sdd  |

この状態で運用していて2台目のストレージが故障した場合を考えてみましょう。障害の切り分けなどで再起動したりすると、FreeBSDでもLinuxでも2台目のストレージが応答しない場合3台目だったデバイスが繰り上がってしまいます。

2台目故障後に再起動したときのデバイス名:

| RAID構成 | FreeBSD     | Linux     | 注                                   |
|----------|-------------|-----------|--------------------------------------|
| 1台目    | /dev/ada1   | /dev/sdb  |                                      |
| 2台目    | /dev/ada2   | /dev/sdc  | 実際は3台目なのに2台目に見える      |
| 3台目    | 無し        | 無し      |                                      |

この状態では、3台目のストレージに障害が起きたように**人間が勘違い**してしまいます。正しく2台目のストレージを交換してリビルドができればRAID 5の冗長性で問題無いわけですが、勘違いのまま3台目を交換するとさらなる障害につながってしまいます。

## 障害時でも確実にストレージを識別するために

前述のような勘違いを防ぐ、つまり障害を起こしたストレージを識別するための手段は何通りかあるのですが、簡単なのはストレージをそのまま使うのではなくGPT(GUID Partition Table)でパーティションを作成し、ラベル名をつけて利用することです。

### FreeBSDでパーティションを作る例

FreeBSDでパーティションを作る場合は、gpartでaddの際に-lでラベル名を指定します。それぞれのラベル名は、storage1, storage2, storage3 とすると次のようになります。

```console
# gpart create -s gpt ada1
ada1 created
# gpart add -t freebsd-zfs -a 1m -l storage1 ada1
ada1p1 added
# gpart create -s gpt ada2
ada2 created
# gpart add -t freebsd-zfs -a 1m -l storage2 ada2
ada2p1 added
# gpart create -s gpt ada3
ada3 created
# gpart add -t freebsd-zfs -a 1m -l storage3 ada3
ada3p1 added
#
```

これでそれぞれのストレージに /dev/ada1p1、/dev/ada2p1、/dev/ada3p1 のパーティションが作成できます。


### Linuxでパーティションを作る例

Linuxでラベル名をつけたパーティションを作るなら、sfdiskを使うのが手っ取り早いでしょう[^fdisk][^type]。

[^fdisk]:fdiskでもラベル名をつけたパーティションを作成できますが、sfdiskよりかなり手順が多くなります。

[^type]:ここでは説明を簡単にするため`name=`だけを指定していますが、場合によっては`type=`等が必要になるかもしれません。`type=`を含めsfdiskの使い方については「[FreeBSDユーザーがsfdiskを使いこなす](https://qiita.com/belgianbeer/items/cc0093ea6ca06a3a4f6c)」の記事が参考になると思います。

```console
# cat s1
label: gpt
name="storage1"
# cat s2
label: gpt
name="storage2"
# cat s3
label: gpt
name="storage3"
#
# sfdisk /dev/sdb < s1
...........
# sfdisk /dev/sdc < s2
...........
# sfdisk /dev/sdd < s3
...........
#
```

この操作によって、/dev/sdb1, /dev/sdc1, /dev/sdd1 のパーティションが用意できます。

## RAIDを構成するときはパーティションを指定する

これができたら、デバイス名としてストレージ全体ではなくパーティション名を指定してRAIDを構成します。

用意したパーティションでRAIDを構成:

| RAID構成 | FreeBSD       | Linux       | ラベル名  |
|----------|---------------|-------------|---------|
| 1台目    | /dev/ada1p1   | /dev/sdb1   | storage1 |
| 2台目    | /dev/ada2p1   | /dev/sdc1   | storage2 |
| 3台目    | /dev/ada3p1   | /dev/sdd1   | storage3 |

この構成であれば最初に書いたように2台目のストレージが故障して3台目が2台目に変わったとしても、正常動作しているパーティションのラベル名を確認することで本当に故障したストレージデバイスを特定できます。

なおラベル名を確認するには、FreeBSDであれば gpart show -l 、Linuxであれば fdisk -x やsfdisk --dump を使います。
