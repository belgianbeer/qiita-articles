<!-- https://qiita.com/belgianbeer/items/b751c2036c7ee698fde2 -->
# ZFSルートのストレージを交換する

## はじめに

「[令和の今頃になってSSDを使うのを止めた話](https://qiita.com/belgianbeer/items/37ca20884b29b0e8514e)」で書いた通り、筆者の自宅のFreeBSDサーバーからSSD撤去することになりました。ここでは実際に作業を行ったときの記録を通じてZFSルートのストレージを交換する手順について書いてみます。

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

これを次の手順で作業を行ってSSDを撤去します。なお以降の説明では2台のHDDを区別するため、一方(ada1側)をHDD1、他方(ada2側)をHDD2と呼びます。

1. HDD1とHDD2で構成されているZFSミラーを解除してHDD1をフリーにする
2. HDD1のパーティションを作り直し、ブート、データ、スワップの各領域を用意する
3. HDD1にブートコードを書き込む
4. SSDのシステムデータをHDD1にコピーする
5. 電源を切ってSSDをシステムから撤去する
6. HDD2のデータをHDD1の残りの領域にコピーする
7. HDD2のパーティションを作り直してHDD1と同じにする
8. HDD1とHDD2のデータ領域をZFSでミラーにする

2から4の手順は、インストーラを使って新たなFreeBSDをHDD1にインストールし、SSDの既存の設定内容をHDD1にコピーする方法でもできますが、既存の設定を網羅するにはSSDの内容を直接コピーする方が確実と言えます。

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
# gpart show ada0    # SSDのパーティション情報の表示
=>       40  488397088  ada0  GPT  (233G)
         40       1024     1  freebsd-boot  (512K)
       1064        984        - free -  (492K)
       2048    8388608     2  freebsd-swap  (4.0G)
    8390656  480006144     3  freebsd-zfs  (229G)
  488396800        328        - free -  (164K)

#
```

当サーバーは少々古いものであるためUEFI非対応のBIOSです。そのためESP(EFI System Partition)が無く、freebsd-boot(512KB)、freebsd-swap(4GB)、freebsd-zfs(229GB)の3個のパーティションから成っています。

一方HDD側は次の通りで、全領域がデータ用のZFSパーティションです。

```console
# gpart show ada1    # HDDのパーティション情報の表示 (ada2も同じ構成)
=>         40  11721045088  ada1  GPT  (5.5T)
           40  11721045088     1  freebsd-zfs  (5.5T)

#
```

ZFSプールの構成は次の通りです。

```console
# zpool list -v
NAME             SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zroot            228G  23.0G   205G        -         -    15%    10%  1.00x    ONLINE  -
  ada0p3         229G  23.0G   205G        -         -    15%  10.1%      -    ONLINE
zvol0           5.45T  2.77T  2.68T        -         -     1%    50%  1.00x    ONLINE  -
  mirror-0      5.45T  2.77T  2.68T        -         -     1%  50.8%      -    ONLINE
    gpt/sdisk1  5.46T      -      -        -         -      -      -      -    ONLINE
    gpt/sdisk2  5.46T      -      -        -         -      -      -      -    ONLINE
#
```

zrootがSSD側のZFSプール、zvol0が2台のHDDで構成されたZFSのミラープールとなります。

## HDD1のパーティションの作り直し

前述の通りHDDには空き領域が無いので、そのままではブート用のパーティションを用意することができません。そこで一旦ミラープールのミラーを解除して、HDD1をフリーにします。

```console
# zpool detach zvol1 gpt/sdisk1
#
```

これでzvol0は単純なZFSプールになり、HDD1のパーティションは自由に変更できます。もちろんミラーを解除したことで万が一HDD2側にエラーがあるとデータを失うリスクがありますから、別途バックアップを保存してあります。

まずはHDD1のパーティションを作り直します。

```console
# gpart show ada1                                             # HDD1のパーティションの確認
=>         40  11721045088  ada1  GPT  (5.5T)
           40  11721045088     1  freebsd-zfs  (5.5T)

# gpart delete -i 1 ada1                                      # データ用のパーティションの削除
ada1p1 deleted
# gpart add -a 4k -t freebsd-boot -s 512K  -l gptboot0 ada1   # freebsd-bootパーティションの作成
ada1p1 added
# gpart add -a 4k -t efi          -s 260M  -l efiboot0 ada1   # ESP用パーティションの作成
ada1p2 added
# gpart add -a 4k -t freebsd-swap -s 4G    -l swap0    ada1   # スワップ用パーティションの作成
ada1p3 added
# gpart add -a 4k -t freebsd-zfs  -s 5367G -l hdpool0  ada1   # システム、データ用のZFS用パーティションの作成 
ada1p4 added
# gpart show ada1                                             # 作ったパーティションの確認
=>         40  11721045088  ada1  GPT  (5.5T)
           40         1024     1  freebsd-boot  (512K)
         1064       532480     2  efi  (260M)
       533544      8388608     3  freebsd-swap  (4.0G)
      8922152  11255414784     4  freebsd-zfs  (5.2T)
  11264336936    456708192        - free -  (218G)

#
```

将来サーバーを交換した場合にもHDDをそのまま使い続けられるように、今回新たにESPを割り当てています。新たなサーバーはUEFI対応のものになるのは確実なので、OSの起動にはESPが必要になります。もしESPが無いと起動のために再びパーティションの作り直す羽目になるので、それを防ぐ目的です。他にも少し予備領域を用意しておこうと考え200GB程度残してあります。

## HDD1へのブートコードの書き込み

パーティションが構成できたので、HDD1からFreeBSDを起動できるようにブートコードを書き込みます。ブートコードの書き込み自体は後で作業しても構いませんが、忘れないようにさっさと済ませておきます。

HDD1はGPTで構成していますがBIOSがUEFIを非対応なので、「/boot/gptzfsboot」をfreebsd-bootパーティションに書き込みます。なおUEFI BIOSの場合のブートコードの処理は本記事の最後に付録としてつけておきます。

```console
# gpart bootcode -b /boot/pmbr -p /boot/gptzfsboot -i 1 ada1
partcode written to ada1p1
bootcode written to ada1
#
```

## SSDのシステムデータのHDD1へのコピー

システムデータをコピーするため、HDD1に新たなZFSプールを作成します。本当はインストーラのデフォルトの名前である「zroot」で作りたいのですが、すでに「zroot」があるため後で名前を変更することにして一旦「zroot2」という名前で作っています。またaltrootを指定しているのは、これからコピーするのは「zroot」にあるFreeBSDのOS部分なので、ディレクトリ構成が衝突しないようにしています。

```console
# zpool create -o altroot=/mnt -O compress=lz4 -O atime=off -m none -f zroot2 gpt/hdpool0
# zpool list
NAME     SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zroot    228G  23.0G   205G        -         -    15%    10%  1.00x    ONLINE  -
zroot2  5.23T   384K  5.23T        -         -     0%     0%  1.00x    ONLINE  /mnt
zvol0   5.45T  2.77T  2.68T        -         -     1%    50%  1.00x    ONLINE  -
#
```

次はシステムのコピーです。これにはzfsのsendとreceive (recv)で、ZFSのデータセットを効率よく転送できます。特にsendの「-R」オプションでは、指定したデータセット配下の全てを転送してくれるので、多数のZFSのデータセットがあってもまとめて処理できます。`zfs receive`で「-u」は転送されたファイルシステムをマウントしないオプションです。デフォルトでは転送が終わったファイルシステムから随時マウントするため、単純に実行して既存のファイルシステム上に同じファイルシステムをマウントするトラブルになることを防ぐ目的で指定しています。

```console
# zfs snapshot -r zroot@copy                            # 転送元になるsnapshotの作成
# zfs send -R zroot@copy | zfs receive -v -F -u zroot2  # sendとrecvでzrootからzroot2へコピー
.....
#
```

コピー時間を正確には計測しなかったのですが、15分程度だったと思います。

これで大部分の内容がコピーできましたが、snapshotを作成した時点からコピー終了までの時間に書き込まれた分(syslogによるもの等)のコピーが出来ていません。確実に全データをコピーするため、ここからはシングルユーザーモードで作業を行います。シングルユーザーモードに移るには、**コンソールで**プロセス番号1のinitにkillを実行します。いくつかのプロセス停止のメッセージが表示されたあとに「/bin/sh」の起動を促されるのでEnterを入力してシングルユーザーモードに移ります。

```console
# kill 1
......
Enter full pathname of shell or RETURN for /bin/sh:
```

この状態であればストレージにバックグラウンドで書き込むプロセスは完全に停止しています。あとはSSDの前回のsnapshotからの差分をHDD1に転送します。

```console
# zfs snapshot -r zroot@copy1      # snapshotの作成
# zfs send -R -I zroot@copy zroot@copy1 | zfs receive -v -u zroot2 # 前回のsnapshotからの差分を転送
#
```

以上でSSDからHDD1へのシステムデータのコピーが完了しました。

## 起動ファイルシステムの指定

ZFSルートの起動のためのブートコードの書き込みはすでに済ませていますが、もう一つ起動のために重要な設定が/bootのあるZFSファイルシステムの指定です。これはZFSプールのbootfsプロパティに設定します。

```console
# zpool set bootfs=zroot2/ROOT/default zroot2
#
```

HDD1の準備ができたのでサーバーを停止します。

```console
# poweroff
```

## SSDの撤去とプール名の変更

ここまでの作業でSSDは不要になったので、筐体からSSDを撤去してHDD1、HDD2のSATAのSATAの接続を1個づつずらします(接続をずらすのは必須では無い)。これによって以降はHDD1がada0、HDD2がada1となります。

次に「zroot2」で作ったプールの名前を「zroot」に変更します。プール名は`zpool import`で変更できるのですが、import済みのプール名は変更できないので、ＦreeBSDのインストール用イメージ(USBメモリ)で起動する必要があります。

起動したら最初のメニューでShellを選択します。

あとは「zroot2」をimportして名前を変更します。

```console
# zpool import -R /mnt -N -f zroot2 zroot   # zroot2のプールをzrootという名前でimport
# zpool export zroot                        # 名前が変更できたのでexport
#
```

以上でプール名の変更は完了です。この時/bootのファイルシステムを指定したbootfsの値は自動的に変わるので、改めて設定する必要はありません[^bootfs]。

[^bootfs]:内部ではシンボリックな名前ではなくプールのGUIDで管理していると思われます。

リブートするとHDDからFreeBSDが起動します。

起動後にプールの状況を見ると、次のように2台のHDDでのサーバーとして動作していることが確認できます。

```console
# zpool list -v
NAME            SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zroot          5.23T  22.9G  5.21T        -         -     0%     0%  1.00x    ONLINE  -
  gpt/hdpool0  5.24T  22.9G  5.21T        -         -     0%  0.42%      -    ONLINE
zvol0          5.45T  2.77T  2.68T        -         -     1%    50%  1.00x    ONLINE  -
  gpt/sdisk2   5.46T  2.77T  2.68T        -         -     1%  50.8%      -    ONLINE
#
```

ZFSルートのストレージ交換としての作業はここまでで終了です。

## HDD2のデータをHDD1へ転送しZFSミラープールを再構成する

ここからはデータ領域をZFSミラープールに変更するための作業となります。今のままではデータはzvol0、システムがzrootなので、そのままではミラープールを構成することができません。そこでまずzvol0の内容をzrootに転送して、HDD2のパーティションを変更します。

転送はSSDの内容を転送したのと同様の手順で次の2段階の作業となります。

1. zvol0のsnapshotからzrootへ転送
2. 1の完了後に、その後の更新分の転送

当サーバーでは毎日夜中にZFSのスナップショットを作成しているので、直近のスナップショットを使ってzrootへ転送します。

```console
# zfs create zroot/zvol0      # HDD1のzrootにzvol0を転送するZFSファイルシステムの作成
# zfs send -R zvol0@_daily_2025-03-07 | time zfs receive -v -F -u zroot/zvol0 # 夜中に作成しているsnapshotからデータを転送
    29134.60 real         2.76 user      3231.93 sys
#
```

この転送にかかった時間は29000秒ほどなので約8時間かかったことになります。もちろんこの転送中にもzvol0へのデータの書き込みはありました。そこで改めて追加書き込み分の転送を行います。なおzvol0にデータの書き込みを行うのは筆者自身なので、自分が書き込む操作を行わない限り更新データは無いわけで、シングルユーザーモードにする必要はありません。

```console
# zfs snapshot -r zvol0@moving      # 次のスナップショットの作成
# zfs send -R -I zvol0@_daily_2025-03-07 zvol0@moving | zfs receive -v -u zroot/zvol0  # 残りの更新分の転送
      278.42 real         1.69 user        50.69 sys
#
```

これでzvol0の内容はすべてzrootへコピーされました。ただ転送先はzroot/zvol0なので、zvol0をexport後にzroot/zvol0配下のZFSファイルシステムの構成を手作業で変更しました。例えば「zvol0/home/\*」は転送後に「zroot/zvol0/home/\*」になっているので、`zfs rename zroot/zvol0/home zroot/home`で構成の修正を行っています。

全てのデータがHDD1へ転送できたので、HDD2の内容を初期化します。

既存プールの情報を消すために、直接`zpool labelclear /dev/ada1p1`を実行しても良いのですが、zroot配下のファイルシステム構成とzvol0配下のファイルシステムを構成の差異を確認したかったため、一旦importしてから内容比較後にdestroyという運びになりました。

```console
# mkdir /zvol0
# zpool import -R /zvol0 -N zvol0
----- (zvol0とzrootのZFSの構成を比較) -----
# zpool destroy zvol0
#
```

続いてHDD2側のパーティションを作り直しHDD1と同じ構成にします。またHDD1が故障したときに備えて、HDD2側にもブートコードを書き込んでおきます。

```console
# gpart delete -i 1 ada1
# gpart add -a 4k -t freebsd-boot -s 512K  -l gptboot1 ada1
ada1p1 added
# gpart add -a 4k -t efi          -s 260M  -l efiboot1 ada1
ada1p2 added
# gpart add -a 4k -t freebsd-swap -s 4G    -l swap1    ada1
ada1p3 added
# gpart add -a 4k -t freebsd-zfs  -s 5367G -l hdpool1  ada1
ada1p4 added
# gpart show ada1
=>         40  11721045088  ada1  GPT  (5.5T)
           40         1024     1  freebsd-boot  (512K)
         1064       532480     2  efi  (260M)
       533544      8388608     3  freebsd-swap  (4.0G)
      8922152  11255414784     4  freebsd-zfs  (5.2T)
  11264336936    456708192        - free -  (218G)

# gpart bootcode -b /boot/pmbr -p /boot/gptzfsboot -i 1 ada1
partcode written to ada1p1
bootcode written to ada1
#
```

パーティションが用意できたので、ZFSミラープールを再構成します。

```console
# zpool attach zroot gpt/hdpool0 gpt/hdpool1
#
```

ミラーの同期を待って、作業は終了です。

## UEFI用ブートコードの書き込み

本サーバーでは意味が無いのですが、UEFI対応BIOS向けにESPにブートコードを用意してみます。

まずESPをFATでフォーマットします。

```console
# newfs_msdos /dev/gpt/efiboot0
/dev/gpt/efiboot0: 532288 sectors in 16634 FAT16 clusters (16384 bytes/cluster)
BytesPerSec=512 SecPerClust=32 ResSectors=1 FATs=2 RootDirEnts=512 Media=0xf0 FATsecs=65 SecPerTrack=63 Heads=16 HiddenSecs=0 HugeSectors=532480
#
```

出来たパーティションを/mntにマウントして、必要なファイルをコピーします。

```console
# mount_msdosfs /dev/gpt/efiboot0 /mnt
# mkdir /mnt/efi /mnt/efi/freebsd /mnt/efi/boot
# cp /boot/loader.efi /mnt/efi/freebsd
# cp /boot/loader.efi /mnt/efi/boot/bootx64.efi
# find /mnt
/mnt
/mnt/efi
/mnt/efi/freebsd
/mnt/efi/freebsd/loader.efi
/mnt/efi/boot
/mnt/efi/boot/bootx64.efi
#
```

ここでは「loader.efi」を2か所にコピーしていますが、「/mnt/efi/freebsd/loader.efi」がFreeBSDを起動するためのブートコード、「/mnt/efi/boot/bootx64.efi」はUEFI BIOSのデフォルトのブートコードです。通常はUEFIのブート変数に起動するOSのブートコードを指定するので、後者が使われることはほとんどありません。

ブート変数の設定には「efibootmgr」を使いますが、本サーバーはUEFI非対応のBIOSですのでブート変数を設定することはできません。実際にUEFI対応BIOSのサーバーであれば、次の要領で設定できます。

```console
# efibootmgr -a -c -l /mnt/efi/freebsd/loader.efi -L FreeBSD
```

ただしすでにFreeBSDが起動するためのブート変数が設定されている場合は「efbootmgr」の「-B」オプションで古いブート変数を削除する必要があります。これらは「efbootmgr」のマニュアルに記載されているので、そちらを参照してください。
