<!-- https://qiita.com/belgianbeer/items/9f963d6b33fd4124b45f -->
# ストレージインターフェースにおけるSATAとUSBの性能差

## はじめに

昨年のFreeBSDのAdvent Calenderで「[RAIDを導入する前に考えること](https://qiita.com/belgianbeer/items/42bd1dd5c35c96d808a8)」という記事を投稿しました。その後SATA接続とUSB接続を切り替えて試すことができたので、実際のパフォーマンスの違いを計測した結果報告です。

## ハードウェア構成

計測環境で実際に利用しているディスク箱は、センチュリーの「[裸族の集合住宅5Bay SATA6G USB3.0&eSATA Ver.2](https://www.century.co.jp/products/crsj535eu3s6g2.html)」で、5台のディスクをUSB 3.0またはeSATAで接続して利用できるものです。ただしeSATAで利用する場合はホスト側のSATAコントローラが **ポートマルチプライヤ** をサポートしている必要があります。この筐体に2TBのSATAのHDDを5台入れてますが、今回の計測では3台を使ってミラーを構成しています。

なお当記事での**USB接続**はあくまでも「[裸族の集合住宅5Bay SATA6G USB3.0&eSATA Ver.2](https://www.century.co.jp/products/crsj535eu3s6g2.html)」固有の話となり、他のUSB接続ではまた違う結果になる可能性があります。

またeSATAはSATAと同じインターフェースを筐体外部で接続することを前提にケーブルとコネクタが用意されているもので、性能面ではSATAとeSATAに差はありません。以降の文ではeSATAであってもSATAの表記で統一しています。

## シーケンシャルアクセスでの性能

まずディスク単体ではどのぐらいの性能があるのかを dd でシーケンシャル読み出しを行なって確認してみます。

```
$ dd if=/dev/ada0 of=/dev/null bs=16m
```

この実行中にiostatコマンドを使ってディスクの転送速度を確認したところ、100 〜 120 MB/s程度になることがわかりました。

次に3台のディスクを組み合わせZFSで3重ミラーのプールを作成します。

```
$ for i in 0 1 2 ; do gpart destroy -F ada${i} ; gpart create -s gpt ada${i} ; gpart add -t freebsd-zfs -l disk${i} ada${i} ; done

$ zpool create -O compression=lz4 -O atime=off zdisk mirror gpt/disk0 gpt/disk1 gpt/disk2
```

作成したプール上に5GB程度のファイルを用意して`cp largefile /dev/null`を実行して、その間の転送速度を`zpool iostat`で計測します。結果は次のようになりました。

| 接続方式 | 転送速度 |
|:-:|:-:|
|  SATA接続 |  150～200 MB/s |
| USB 3.0接続  | 100～120 MB/s  |

結果からSATA接続であれば単体ディクスよりも高速な転送速度を確保でき、ミラーを構成することでパフォーマンスアップにつながっていることがわかります。一方でUSB接続の場合、ミラーを構成してもディスク単体に比較してパフォーマンス面でのメリットはまったくありません。

この理由は、SATA接続であればインターフェースの帯域が許される範囲でディスクへの並列アクセスが可能であるのに対して、USB接続では1台のディスクとの間のコマンドのやりとりが完了しないと次のディスクのアクセスができないことによります。つまり複数台のディスクに対して並列アクセスができるのかどうかの違いがこの結果につながっています。

ディスクが並列アクセスできるかどうかは起動時のメッセージで確認できます。

```
ada0 at ahcich0 bus 0 scbus0 target 0 lun 0
ada0: <Hitachi HUA722020ALA330 JKAOA3EA> ATA8-ACS SATA 2.x device
ada0: Serial Number JK11A5YAK641GX
ada0: 300.000MB/s transfers (SATA 2.x, UDMA6, PIO 8192bytes)
ada0: Command Queueing enabled
ada0: 1907729MB (3907029168 512 byte sectors)
```
下から2行目のように`Command Queueing enabled`が表示されれば、並列アクセスが可能です。同じディスクをUSBで接続した場合、このメッセージは表示されません。

## 実利用での性能

シーケンシャルアクセスの性能は前述の通りですが、ランダムアクセスや実利用でのパフォーマンスは果たしてどうなるのでしょうか。ベンチマークテストプログラムを軽く試した範囲では顕著な差が得られなかったため、実利用でのパフォーマンスの差を知りたいこともあり、git pull の実行時間を計測してみました。

対象となるgitリポジトリとしてFreeBSDのportsのツリーを用意し、次の方法でgit pullにかかる時間を計測します。‘

1. freebsd-ports を git clone して zfs snapshotを作成
1. プールを一旦exportしてimport (OSのバッファリングの影響を消すため)
1. git pull を実行して時間を計測
1. 直前に作成した snapshot へ rollback
1. 接続方式を切り替えて、改めて計測

計測時の状況は次のようになります。

```
$ time git pull
remote: Enumerating objects: 1440, done.
remote: Counting objects: 100% (1201/1201), done.
remote: Compressing objects: 100% (494/494), done.
remote: Total 718 (delta 446), reused 484 (delta 217), pack-reused 0
Receiving objects: 100% (718/718), 146.73 KiB | 141.00 KiB/s, done.
Resolving deltas: 100% (446/446), completed with 231 local objects.
From github.com:freebsd/freebsd-ports
   cfe15eaebeff..fca8faf5bb0a  main       -> origin/main
   7a2b4e1ff1b7..03c599c1eb7a  2021Q3     -> origin/2021Q3
Updating cfe15eaebeff..fca8faf5bb0a
     151.77 real         1.82 user         8.69 sys
$
```

ここで見るべき値はrealで表示されるI/Oの時間を含んだトータルの実行時間です。この計測に関しては三重ミラーだけなく、ミラーを解除した状態の単体ディスクでの計測も行いました。これらの結果は次の通りです。

| ディスク構成 | 接続方式 | git pullの実行時間 (秒) |
|:-:|:-:|:-:|
| 三重ミラー | SATA接続 | 62.53 |
| 三重ミラー | USB 3.0接続 | 151.77  |
| 単体ディスク | SATA接続 | 65.3 |
| 単体ディスク | USB 3.0接続 | 95.5 |

結果が示す通り、USB接続はSATA接続に比較してパフォーマンス面でかなり不利であることがわかります。また単体ディスクでの計測時間の差から、USB接続はSATA接続に比べてディスクアクセスのオーバーヘッドが大きいこともわかります。

今回の計測ではHDDを使いましたが、SATAとUSBのパフォーマンスの違いはHDDに限った話ではなくSSDでも同様の結果となります。もちろんSSDの方がHDDに比べると圧倒的に転送速度は速くなりますが、それでもUSB接続ではSATA接続に比べて実利用でのパフォーマンスが低下します。

## まとめ

以上の実験結果から次のことが導けます。

- USB接続は複数ディスクアクセスでの並列性が無いため、複数ディスクを同時アクセスするような状況で利用しない方が良い
- USB接続は3.0以降であれば転送速度はそれなりに速いがディスクI/OにおいてはオーバーヘッドがSATA接続に比べて大きい
- SATA接続が選べる状況において、パフォーマンス面でUSB接続を選ぶメリットは無い

これらのことから、USB接続のストレージは日常的な作業領域には不向きで、アーカイブやバックアップ等の用途に限定して利用するのが無難でしょう。またパフォーマンス面からも単体で利用するのが得策で、RAIDにすると組み合わせる台数が増えるほどオーバーヘッドが蓄積されてアクセスタイムの面で不利となります。計測ではFreeBSDを使っていますが、WindowsやLinuxでも同じことが起きることは確認済です。

## 付録：ポートマルチプライヤ

今回実験に使ったディスク箱はSATA接続の場合ポートマルチプライヤのサポートが必須となりますが、SATAコントローラがポートマルチプライヤをサポートしているかどうかはFreeBSDの場合起動時のメッセージで確認できます。次の例ではahci0はポートマルチプライヤのサポート有り、ahci1はサポート無しです。

```
ahci0: <AMD SB7x0/SB8x0/SB9x0 AHCI SATA controller> port 0xb000-0xb007,0xa000-0xa003,0x9000-0x9007,0x8000-0x8003,0x7000-0x700f mem 0xfe7ffc00-0xfe7fffff irq 22 at device 17.0 on pci0
ahci0: AHCI v1.10 with 4 3Gbps ports, Port Multiplier supported
ahci0: quirks=0x22000<ATI_PMP_BUG,1MSI>

ahci1: <Intel Cannon Lake AHCI SATA controller> port 0x4090-0x4097,0x4080-0x4083,0x4060-0x407f mem 0xa1334000-0xa1335fff,0xa133a000-0xa133a0ff,0xa1339000-0xa13397ff at device 23.0 on pci0
ahci1: AHCI v1.31 with 6 6Gbps ports, Port Multiplier not supported
```

ところでホスト側のコントローラはMarvel 88SE9128で、仕様書で見るとポートマルチプライヤをサポートしていますし、実際に使えています。にもかかわらず起動時のメッセージでは`Port Multiplier not supported`と表示される少し不思議な状況です。

またポートマルチプライヤには、通信方式の違いでコマンドベーススイッチング(CBS)とFIS(Frame Information structure)ベーススイッチング(単にFBSと表記することもある)の2種類があります。説明は割愛しますがFBSであればSATAの直結と変わらないパフォーマンスが得られ、Marvel 88SE9128はFBSをサポートしています。
