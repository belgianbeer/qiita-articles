<!-- https://qiita.com/belgianbeer/items/8df197588462cd7f6b45 -->

# ZFSミラープールのディスク交換による容量増設

## はじめに

自宅サーバーのハードディスクの使用量がそろそろ80%を超えている[^perf]のと使用開始から10年以上経過していることもあり、さすがにそろそろ交換したほうが安全だろうという判断で、新しいハードディスクを購入して交換しました。この記事は実際にハードディスクを交換したときの記録です。

なお筆者のサーバーのOSはFreeBSDですが、LinuxでZFSを使っている場合でもZFSに関する操作は同様となります(筆者はLinuxでもZFSを利用しています)。

[^perf]: ZFSではCopy on Writeでデータ更新を行なう関係で、空き容量が少なくなると他のファイルシステムより極端にパフォーマンスが低下する傾向があるため(ハードディスクで顕著)、容量ギリギリまで利用するのは得策ではありません。どのぐらいまで使うとどうパフォーマンスが低下するのかに関してはワークセットとの兼ね合いなので単純には言えませんが、他のファイルシステムなら影響の無い空き容量であってもZFSではパフォーマンスの低下に繋がることがあります。

## サーバーのストレージ構成

対象サーバーのOSはFreeBSDで、ストレージとしてシステム用にSSD(256GB)が1台、データ用にハードディスク(2TB)が2台接続してあって、全てZFSで構成してあります。2台のハードディスクはデータ保全のためZFSでミラーを構成しているので、2台あっても容量としては1台分となります。今回は2TBのハードディスク2台をそれぞれ6TBのものに交換します。

## 交換作業前の確認

実際に交換作業をおこなう前に、現在の状態を確認します。`zpool list`でハードディスクの使用量などを確認すると次の通りでした。

```console
$ zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
zroot   228G  16.8G   211G        -         -    12%     7%  1.00x  ONLINE  -
zvol0  1.81T  1.49T   333G        -         -     5%    82%  1.00x  ONLINE  -
$
```

zrootがシステム用のSSDです。そしてzvol0が今回の交換対象のハードディスクで、ZFSミラー構成の使用量が82%になっていることがわかります。

構成の詳細は、`zpool list -v`や`zpool status`で確認できます。

```console
$ zpool list -v zvol0
NAME             SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  
zvol0           1.81T  1.49T   333G        -         -     5%    82%  1.00x    ONLINE  -
  mirror-0      1.81T  1.49T   333G        -         -     5%  82.1%      -    ONLINE
    gpt/ndisk2      -      -      -        -         -      -      -      -    ONLINE
    gpt/ndisk1      -      -      -        -         -      -      -      -    ONLINE
$
```

```console
$ zpool status zvol0
  pool: zvol0
 state: ONLINE
  scan: scrub repaired 0B in 05:36:58 with 0 errors on Tue Jan 17 08:40:57 2023
config:

        NAME            STATE     READ WRITE CKSUM
        zvol0           ONLINE       0     0     0
          mirror-0      ONLINE       0     0     0
            gpt/ndisk2  ONLINE       0     0     0
            gpt/ndisk1  ONLINE       0     0     0

errors: No known data errors
$
```

これらからndisk1とndisk2のディスクパーティションでミラーを構成してあることが分かります。実際3台のストレージデバイスはada0(SSD)、ada1(ハードディスク)、ada2(ハードディスク)という構成で、ハードディスクのパーティションは次のようになっています。

```console
$ gpart show ada1 ada2
=>        40  3907029088  ada1  GPT  (1.8T)
          40  3907029088     1  freebsd-zfs  (1.8T)

=>        40  3907029088  ada2  GPT  (1.8T)
          40  3907029088     1  freebsd-zfs  (1.8T)

$ gpart show -l ada1 ada2
=>        40  3907029088  ada1  GPT  (1.8T)
          40  3907029088     1  ndisk1  (1.8T)

=>        40  3907029088  ada2  GPT  (1.8T)
          40  3907029088     1  ndisk2  (1.8T)

$
```

ada1、ada2ともGPTで、ada1のZFS用パーティションにndisk1、ada2ではndisk2と名前をつけてあるのがわかります。

## ミラー構成でのハードディスク交換手順

ZFSに限らずミラー(RAID 1)構成のハードディスクを2台とも入れ替える手順としては、ざっくり次の2通りの方法が考えられます。

* 新しいハードディスクでミラーを用意して既存のミラーからデータコピーし、コピー完了後に入れ替える。あるいは先に交換して元のハードディスクからデータをコピーする。
* 片方のハードディスクを交換してミラーの機能でデータを同期し、完了後にもう一方のハードディスクを交換して改めてデータを同期する

前者の方法であればデータコピーは1回で済みますが同時に3台または4台のハードディスクを接続する必要がありSATAインターフェースの数なども気になります。この場合ハードディスクはUSBで接続しても構わないのですが、USB 2.0の転送速度では遅くてコピーに要する時間が長くなるためUSB 3.0にしたいところです。

筆者の場合は後者の1台づつ交換する方法を取りました。このやり方であれば交換作業時はともかく、同期中でも(パフォーマンスの低下はあるものの)通常通りサーバーを使い続けられるというメリットがあります。

## ハードディスクの交換とデータの同期

### ZFSミラーでのドライブ交換時の注意

ZFSのミラーでドライブを交換する場合に注意することは、交換前にZFSプールの操作、例えば `zpool detach zvol0 gpt/ndisk2` などを **実行しないことが望ましいです**。なぜなら`zpool detach`でndisk2をプールから外すとndisk2内のデータは消去された扱いになり、もしも新しいハードディスクとのデータの同期中にndisk1の読み出しでエラーが発生した場合にndisk2に交換して同期をやり直すことができなくなってしまいます[^undestroy]。電源を切ってndisk2を外すだけであればndisk2内のデータはndisk1と同じままなので、万が一のときにndisk2を使ってやり直しができます。

[^undestroy]: detachしてもndisk2のデータは`zpool import -D`を使えば復元できる気がしますが、試したことはありません。

### 1台目のハードディスクの交換

まず1台目のハードディスクの交換ですが、ndisk2のパーティションのあるハードディスクを外して交換します。サーバーをシャットダウンして電源を切り、古いハードディスクを撤去して新しいハードディスク接続後に電源を入れます。

起動後にzvol0の状態を確認します。

```console
$ zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zroot   228G  16.8G   211G        -         -    12%     7%  1.00x    ONLINE  -
zvol0  1.81T  1.49T   333G        -         -     5%    82%  1.00x  DEGRADED  -
$
```

```console
$ zpool status zvol0
  pool: zvol0
 state: DEGRADED
status: One or more devices could not be opened.  Sufficient replicas exist for
        the pool to continue functioning in a degraded state.
action: Attach the missing device and online it using 'zpool online'.
   see: https://openzfs.github.io/openzfs-docs/msg/ZFS-8000-2Q
  scan: scrub repaired 0B in 05:36:58 with 0 errors on Tue Jan 17 08:40:57 2023
config:

        NAME                      STATE     READ WRITE CKSUM
        zvol0                     DEGRADED     0     0     0
          mirror-0                DEGRADED     0     0     0
            12897545936258916783  UNAVAIL      0     0     0  was /dev/gpt/ndisk2
            gpt/ndisk1            ONLINE       0     0     0

errors: No known data errors
$
```

2台のハードディスクでミラーを構成しているはずが片側のgpt/ndisk2が無く1台だけになっているため、HEALTHやSTATEでDEGRADEDと表示されています。また`zpool status`では構成を示すconfig:のところでgpt/ndisk2が無く替わりに「12897545936258916783」というIDが表示されていて、行末に「was /dev/gpt/ndisk2」とあります。このIDは後ほど使用します。

### 交換用パーティションの作成

交換用のハードディスクの準備としては、GPTで初期化してZFS用のパーティションを用意します。

まず、GPTでハードディスクを初期化します。

```console
$ gpart show ada1
gpart: No such geom: ada1.
$ gpart create -s gpt ada1
ada1 created
$ gpart show ada1
=>         40  11721045088  ada1  GPT  (5.5T)
           40  11721045088        - free -  (5.5T)

$
```

続いてZFS用のパーティションを割り当てます。新しいパーティションには古いものと区別するためsdisk1と名前をつけました。

```console
$ gpart add -t freebsd-zfs -a 4k -l sdisk1 ada1
ada1p1 added
$ gpart show ada1
=>         40  11721045088  ada1  GPT  (5.5T)
           40  11721045088     1  freebsd-zfs  (5.5T)

$ gpart show -l ada1
=>         40  11721045088  ada1  GPT  (5.5T)
           40  11721045088     1  sdisk1  (5.5T)

$
```

### 1台目のデータの同期

新しいパーティションを用意できたので、旧ndisk2をsdisk1で置き換えます。このための操作には`zpool replace`を使います。

通常であれば`zpool replace`の元になるデバイスにはプールを構成しているデバイスを指定するのですが、今回の場合は交換のためすでに取り外しているので該当デバイスは存在しません。このような場合は`zpool status zvol0`で表示されているIDを指定します。

```console
$ zpool replace zvol0 12897545936258916783 gpt/sdisk1
$
```

これでsdisk1に対してndisk1からのミラーの同期が始まります。

同期はバックグラウンドで行われて、前述したようにこの間通常通りサーバーを利用できます。データ量は1.5TB程度あるため同期にはそれなりの時間がかかるわけですが、その間状態がわからないと少しばかり不安になります。ZFSでは同期の進行状況を`zpool status`で確認できるので終了時間などの目安を得られます。

```console
$ zpool status zvol0
  pool: zvol0
 state: DEGRADED
status: One or more devices is currently being resilvered.  The pool will
        continue to function, possibly in a degraded state.
action: Wait for the resilver to complete.
  scan: resilver in progress since Sat Jan 28 14:25:44 2023
        12.8G scanned at 596M/s, 600K issued at 27.3K/s, 1.49T total
        0B resilvered, 0.00% done, no estimated completion time
config:

        NAME                        STATE     READ WRITE CKSUM
        zvol0                       DEGRADED     0     0     0
          mirror-0                  DEGRADED     0     0     0
            replacing-0             DEGRADED     0     0     0
              12897545936258916783  UNAVAIL      0     0     0  was /dev/gpt/ndisk2
              gpt/sdisk1            ONLINE       0     0     0
            gpt/ndisk1              ONLINE       0     0     0

errors: No known data errors
$
```

status:で`One or more devices is currently being resilvered`とresilver[^resilver]中(同期中)であることを示しています。action:にはresilverが終わるのを待つように指示が表示されています。そしてscan:では処理量や予想時間が表示されるのですが`no estimated completion time`と表示しているように、同期開始直後は予想時間が不明なため表示されません。

[^resilver]: 「resilver」という言葉はZFSを使うようになって知ったのですが、本来は銀の装飾品を磨くという意味でZFSではRAIDの復旧中であることを示しています。

少し経つと終了予想時間が表示されるようになりますが、経験的にこの段階ではまったくあてになりません。

```console
$ zpool status zvol0
===== <省略> =====
  scan: resilver in progress since Sat Jan 28 14:25:44 2023
        265G scanned at 664M/s, 6.51G issued at 16.3M/s, 1.49T total
        6.51G resilvered, 0.43% done, 1 days 02:27:43 to go
===== <省略> =====
$ 
```

さらに時間を置いてから確認すると終了までに14時間程度と表示していました。

```console
$ zpool status zvol0
===== <省略> =====
  scan: resilver in progress since Sat Jan 28 14:25:44 2023
        289G scanned at 488M/s, 18.6G issued at 31.4M/s, 1.49T total
        18.6G resilvered, 1.22% done, 13:38:30 to go
===== <省略> =====
$
```

1時間後に確認したときは6時間程度まで減っていましたが、結局1台目のミラーの同期には15時間半程かかりました。以下は1台目の同期が完了したときに`zpool list`と`zpool status`を確認した結果です。

```console
$ zpool list -v zvol0
NAME             SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zvol0           1.81T  1.49T   333G        -         -     5%    82%  1.00x    ONLINE  -
  mirror-0      1.81T  1.49T   333G        -         -     5%  82.1%      -    ONLINE
    gpt/sdisk1      -      -      -        -         -      -      -      -    ONLINE
    gpt/ndisk1      -      -      -        -         -      -      -      -    ONLINE
$
```

```console
$ zpool status zvol0
  pool: zvol0
 state: ONLINE
  scan: resilvered 1.49T in 15:34:45 with 0 errors on Sun Jan 29 06:00:29 2023
config:

        NAME            STATE     READ WRITE CKSUM
        zvol0           ONLINE       0     0     0
          mirror-0      ONLINE       0     0     0
            gpt/sdisk1  ONLINE       0     0     0
            gpt/ndisk1  ONLINE       0     0     0

errors: No known data errors
$
```

HEALTHならびにSTATEはすべてONLINEで、エラーは0と発生しておらず、1台目の同期は問題なく完了したことがわかります。
### 2台目のハードディスクの同期

2台目のハードディスクの交換と同期の手順は1台目とまったく同じなので、コマンドの実行と結果のみに留めます。

2台目を交換して起動したところ。

```console
$ zpool status zvol0
  pool: zvol0
 state: DEGRADED
status: One or more devices could not be opened.  Sufficient replicas exist for
        the pool to continue functioning in a degraded state.
action: Attach the missing device and online it using 'zpool online'.
   see: https://openzfs.github.io/openzfs-docs/msg/ZFS-8000-2Q
  scan: resilvered 1.49T in 15:34:45 with 0 errors on Sun Jan 29 06:00:29 2023
config:

        NAME                      STATE     READ WRITE CKSUM
        zvol0                     DEGRADED     0     0     0
          mirror-0                DEGRADED     0     0     0
            gpt/sdisk1            ONLINE       0     0     0
            13953654332474917058  UNAVAIL      0     0     0  was /dev/gpt/ndisk1

errors: No known data errors
$
```

ミラー用パーティションの準備

```console
# gpart create -s gpt ada2
ada2 created
# gpart show ada2
=>         40  11721045088  ada2  GPT  (5.5T)
           40  11721045088        - free -  (5.5T)

# gpart add -t freebsd-zfs -a 4k -l sdisk2 ada2
ada2p1 added
# gpart show ada2
=>         40  11721045088  ada2  GPT  (5.5T)
           40  11721045088     1  freebsd-zfs  (5.5T)

# gpart show -l ada2
=>         40  11721045088  ada2  GPT  (5.5T)
           40  11721045088     1  sdisk2  (5.5T)

#
```

2台目の同期開始

```console
$ zpool replace zvol0 13953654332474917058 gpt/sdisk2
$ zpool status zvol0
  pool: zvol0
 state: DEGRADED
status: One or more devices is currently being resilvered.  The pool will
        continue to function, possibly in a degraded state.
action: Wait for the resilver to complete.
  scan: resilver in progress since Sun Jan 29 13:38:15 2023
        12.8G scanned at 692M/s, 492K issued at 25.9K/s, 1.49T total
        0B resilvered, 0.00% done, no estimated completion time
config:

        NAME                        STATE     READ WRITE CKSUM
        zvol0                       DEGRADED     0     0     0
          mirror-0                  DEGRADED     0     0     0
            gpt/sdisk1              ONLINE       0     0     0
            replacing-1             DEGRADED     0     0     0
              13953654332474917058  UNAVAIL      0     0     0  was /dev/gpt/ndisk1
              gpt/sdisk2            ONLINE       0     0     0

errors: No known data errors
```

2台目の同期は14時間程でした。

```console
$ zpool status zvol0
  pool: zvol0
 state: ONLINE
  scan: resilvered 1.49T in 14:11:00 with 0 errors on Mon Jan 30 03:49:15 2023
config:

        NAME            STATE     READ WRITE CKSUM
        zvol0           ONLINE       0     0     0
          mirror-0      ONLINE       0     0     0
            gpt/sdisk1  ONLINE       0     0     0
            gpt/sdisk2  ONLINE       0     0     0

errors: No known data errors
$
```

## 容量の拡張

交換とデータの同期は無事終わりましたが、まだやるべきことが残っています。というのはこのままではハードディスクを交換しただけで、2TB→6TBの容量の拡張が有効になっていません。

あらためてzvol0の容量を確認するとSIZEは1.81Tと、交換前と同じ数字を示しています。しかしよく見るとEXPANSZに3.62Tと表示されていて、あと3.62TB拡張できることもわかります。

```console
$ zpool list zvol0
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zvol0  1.81T  1.49T   333G        -     3.62T     5%    82%  1.00x    ONLINE  -
$
```

この容量拡張を行う設定がZFSプールのautoexpandプロパティです。まず現状のautoexpandの値を確認すると次のようにデフォルトのoffとなっています。

```
$ zpool get autoexpand zvol0
NAME   PROPERTY    VALUE   SOURCE
zvol0  autoexpand  off     default
$
```

autoexpandをonにしてみます。

```console
$ zpool set autoexpand=on zvol0
$ zpool get autoexpand zvol0
NAME   PROPERTY    VALUE   SOURCE
zvol0  autoexpand  on      local
$
```

しかしautoexpandをonにしただけでは容量は変わりません。

```console
$ zpool list zvol0
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zvol0  1.81T  1.49T   333G        -     3.62T     5%    82%  1.00x    ONLINE  -
$
```

容量を拡張するためには、`autoexpand=on`を設定したうえで、`zpool online -e`を実行する必要があります。

```console
$ zpool online -e zvol0
missing device name
usage:
        online [-e] <pool> <device> ...
$
```

デバイス名の指定を忘れてエラーになりました😅が、本来の`zpool online`であればデバイス名の指定が必須なのは当然として、`-e`オプションのときはデバイス名の指定は無くてもよさそうな気がします。それはともかく`zpool online -e`ではプール内のデバイス名のどれか1つを指定すれば問題ありません。

あらためてデバイス名を指定して実行すると、次のように容量が5.45TB[^size]まで増えました。

```console
$ zpool online -e zvol0 gpt/sdisk1
$ zpool list zvol0
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zvol0  5.45T  1.49T  3.97T        -         -     1%    27%  1.00x    ONLINE  -
$
```

今回はautoexpandプロパティの変更をハードディスクの交換後に行いましたが、交換前に設定しておくと、2台目の同期完了と同時に容量が拡張されるようになります。またautoexpandはミラーだけでは無くraidz等の構成でも同様に利用できます。

[^size]: ハードディスクでのTBは10進数での容量表記なので6TBは6,000,000,000,000バイト、2進数での1TBは1,099,511,627,776なので5.45TBとなります。

**2023-07-16 追記**：`zpool online -e`で拡張する場合は autoexpandプロパティの設定は必要ないという指摘をいただきました。autoexpandプロパティは自動的に容量を拡張するための設定でした。

## 交換完了後

新しいハードディスクでは転送速度があがっているため、全体としてパフォーマンスが向上しているのを体感できました。すでに10年以上経ったCPUですが、主な用途はファイルサーバーということもあり、まだ当面使い続けられそうです。
