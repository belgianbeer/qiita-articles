<!-- https://qiita.com/belgianbeer/items/b638c12150dc86911922 -->

# ChromeOS FlexとWindowsやmacOS等のマルチブート環境を作る

【2024-05-30 追記】久しぶりにChromeOS Flexのデュアルブート環境を作ってみたところ、**2023年11月にChromeOS Flexのインストーラに変更がありました**。Macでは記載のままで問題ありません。Windows PCの場合は既存ESPの置き換えまではそのままですが、その後**Windowsのインストール後に起動しなくなると思われます**。また改めて作業して手順を減らせることにも気づいたので、現在記事の修正版を作成中です。

## はじめに

ChromeOS Flexは1台のPCで他のOSとのマルチブートをサポートしていません。というのもインストール時にストレージの使用量を調整する手段が無く、既存のOSの領域を削除した上で全領域をChromeOS Flex用として確保してしまいます。そのため他のOSをインストールするための空き領域を確保できないわけです。

ここでは裏技的な手法を使って、1台のPCにChromeOS Flexと他のOSとのマルチブート環境を構築する方法について説明します。あくまでも裏技なので将来にわたってこの方法が通用するかどうかは不明ですが、当面は問題無いでしょう。

しかしながら、**ChromeOS Flexと他のOSのマルチブート環境はあまりお勧めとは言えません**。というのもChromeOS Flexとのマルチブート環境のPCで**WindowsやmacOSがパーティションの変更を行うと、ChromeOS Flex用のパーティションテーブルが破壊される為です**。実際この記事を書きながら設定したPCでも、マルチブート環境構築後にパーティションテーブルが破壊されるという事態が発生しました(後で紹介します)。これはWindowsやmacOSが悪いというより、ChromeOS Flexのパーティションテーブルが少し変わった構成になっていることによります。もし破壊されてもパーティションテーブルのバックアップがあれば修復できるのですが、ここで紹介するマルチブート環境を使い続けるのであれば、トラブル時に速やかに対処を行うセンスが要求されます。

## 用意するもの

ChromeOS Flexとのマルチブート環境を作るのに必要なものは次の通りです。

1. UEFIをサポートしたWindows PCまたはMac(Intel CPU)本体
1. ChromeOS Flexのインストール用USBメモリ(8GB以上)
1. Windows PCであればWindows 10か11のインストール用USBメモリ(8GB以上)、Macの場合はmacOSのインストール用USBメモリ(16GB以上)
1. Debian Liveの起動用USBメモリ(2GB以上)
1. 記録用のUSBメモリ(100KB程度の空きがあれば十分)

UEFIをサポートしていないPCでもChromeOS Flexの動作は問題ありませんが、他OSとのブートの切り替えにUEFIで用意されているブートセレクタを使うことを前提にしているため、GPT(GUID Partition Table)を扱えるUEFI対応のPCが必須となります。Intel CPUのMacはUEFIをサポートしているので問題ありません。Windows PC、Macのいずれを利用するにしても、ChromeOS Flexをインストールすると既存データは削除されてしまいますので、事前のバックアップは必須です。さらにWindows PCの場合は[回復ドライブ](https://support.microsoft.com/ja-jp/windows/%E5%9B%9E%E5%BE%A9%E3%83%89%E3%83%A9%E3%82%A4%E3%83%96%E3%82%92%E4%BD%9C%E6%88%90%E3%81%99%E3%82%8B-abb4691b-5324-6d4a-8766-73fab304c246)を作成し、最悪の場合に出荷時の設定に戻せるよう準備しておくことをお勧めします。

また、対象のPCでChromeOS FLexが正常に動作することを事前に確認しておく必要があります。[ChromeOS Flexの認定モデル](https://support.google.com/chromeosflex/answer/11513094 "認定モデルリスト")であれば安心ですが、そうでない場合は[ChromeOS FlexのUSBインストーラ](https://support.google.com/chromeosflex/answer/11541904)を使って、インストール用USBメモリだけでの動作確認を行うのが良いでしょう。

WindowsやmacOSのインストール用のUSBメモリは、それぞれ「[Windows 10 のダウンロード](https://www.microsoft.com/ja-JP/software-download/windows10)」、「[Windows 11 をダウンロードする](https://www.microsoft.com/ja-jp/software-download/windows11)」、「[macOS の起動可能なインストーラを作成する](https://support.apple.com/ja-jp/HT201372)」を参照して、必要なものを用意します。

Debianが配布している[Debian Liveの起動用USBメモリ](https://www.debian.org/CD/live/index.ja.html)は、Linuxのsfdiskコマンドを使ってパーティションテーブルを編集する際に使用します。Debian Liveには様々なイメージが用意されていますが、今回の用途ではデスクトップ環境は不要なのでサイズの小さいstandard版(`debian-live-11.7.0-amd64-standard.iso` 2023年5月現在)で問題ありません。Linuxの起動用USBメモリでシェルが起動できてsfdiskとエディタ等が利用できるのであれば、Debian Liveでなくてもかまいません。

この他に記録用のUSBメモリも必要になります。Debian LiveのUSBメモリに記録できれば用が足りるのですがファイルシステムの関係で書き込むことができません。パーティションテーブルのテキストファイルを数個保存するだけなので、手頃なものを用意してください。

なお本作業では、インストール用USBメモリでPCを起動する方法や、OSのインストールに関する知識が必須となります。これらをご存じない方はマニュアルやメーカーのサポートサイト、動画等で確認してください。

## ChromeOS Flexのインストール

最初にChromeOS Flexをインストールします。インストール前に行うことは、前述のように既存データのバックアップやインストール用のUSBメモリを作ることの他に、Windows PCの場合はBIOSの設定でUEFIが有効になっているかどうかを確認します。少し古いPCではUEFI対応のBIOSであってもUEFIの起動が無効や従来のBIOS起動を優先する設定になっていることがあるので、必ずUEFIを優先して使う設定に変更します。Macの場合は事前に特別な作業はありません。

ChromeOS Flexのインストールに関しては、USBインストーラを作成して起動し、メニューにしたがって順に操作すればよいので難しいところは無いでしょう。Googleも[ChromeOS Flexインストールガイド](https://support.google.com/chromeosflex/answer/11552529)を用意していますし、検索すればインストールの記事や動画が多数見つかります。

通常であればインストール後に起動してアカウント設定等を行うのですが、設定してもこの後の作業で消去されてしまい改めて設定することになります。とは言えある程度のことを試しておくのも良いでしょう。

## 空き領域と置換用EFIシステムパーティション領域の確保

すでに説明したように、この時点ではChromeOS FLexのインストールによって内蔵ストレージの全領域がChromeOS Flex用に割り当てられています。このままでは他のOSをインストールできませんから、Debian Liveの環境を使ってパーティションテーブルを編集し、ChromeOS Flexの使用領域を減らしてWindowsやmacOS用の空き領域を確保します。

### Debian Liveの起動

パーティションテーブルの編集ではaptでパッケージを追加するため、PCがインターネットにアクセスできるようLANケーブルを接続してから、Debian LiveのUSBメモリを接続して起動します。Wi-Fi経由でもインターネット接続できると思いますが筆者は試したことがありません。

Debian Liveが起動するとシェルのコマンドラインが立ち上がりますが、少し古いバージョンのDebian Liveのイメージを使った場合はログインプロンプトになることがあります。その場合はユーザー名に「user」パスワードに「live」を指定してログインします。

その後記録用のUSBメモリをPCに接続します。この状態では、内蔵ストレージ、Debian LiveのUSBメモリ、記録用のUSBメモリの3個のストレージが接続された状態になりますが、多くの場合Debian環境下でのストレージデバイスの割り当ては次のようになるはずです。

<table>
  <caption>ストレージとデバイス名の対応</caption>
  <thead>
    <tr>
      <th>ストレージの種類</th> <th>デバイス名</th>
    </tr>
  </thead>
  <tr>
    <td> 内蔵ストレージ </td> <td>/dev/sda</td>
  </tr>
  <tr>
    <td> Debian LiveのUSBメモリ</td> <td>/dev/sdb</td>
  </tr>
  <tr>
    <td> 記録用のUSBメモリ</td> <td>/dev/sdc</td>
  </tr>
</table>

PC側のストレージ構成によってはこの表とは違う割り当てになることもあるので、実際のストレージとデバイス名の割り当てを必ず確認してください。以降はこの割り当てを前提に説明しますので、もしデバイスの割り当てが違う場合は適宜読み替えてください。

### Debian Liveでの事前準備

`sudo -i`でrootユーザーになります。

```command
user@debian:~$ sudo -i
root@debian:~# 
```

設定途中でmkdosfsコマンドを使うため、aptを使ってdosfstoolsをインストールします。

```command
root@debian:~# apt update
.....(省略).....
root@debian:~# apt install dosfstools
.....(省略).....
root@debian:~# 
```

必須ではないですが、openssh-serverのパッケージをインストールすると別のPCからログインできるようになります。`ip addr show`でIPアドレスを確認しsshで対象のPCにログインすれば、テキストのコピーやペーストができるようになって作業効率があがります。なおsshでログインするときのユーザー名は「user」、パスワードは「live」です。

```command
root@debian:~# apt install openssh-server
.....(省略).....
root@debian:~# ip addr show
.....(省略).....
root@debian:~# 
```

記録用のUSBメモリを/mntにマウントして、そこにcdします。これでUSBメモリにコマンドの出力結果などを残せるようになります。

```command
root@debian:~# mount /dev/sdc1 /mnt
root@debian:~# cd /mnt
root@debian:/mnt# 
```

### ChromeOS Flexの不思議なパーティションテーブル

最初に`sfdisk --list`を使ってChromeOS Flexのパーティションテーブルを確認するとともに、あとで変更内容を比較するためにp1-sda-listというファイルに結果を保存しています。なお本記事で使っているPCの内蔵ストレージは120GBのSSDです。

```command
root@debian:/mnt# sfdisk --list /dev/sda | tee p1-sda-list
Disk /dev/sda: 111.79 GiB, 120034123776 bytes, 234441648 sectors
Disk model: INTEL SSDSC2BW12
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: AC161E76-BF4B-924D-9C72-06CE3C6EABCF

Device        Start       End   Sectors   Size Type
/dev/sda1  17010688 234441599 217430912 103.7G Linux filesystem
/dev/sda2        69     32836     32768    16M ChromeOS kernel
/dev/sda3   8622080  17010687   8388608     4G ChromeOS root fs
/dev/sda4     32837     65604     32768    16M ChromeOS kernel
/dev/sda5    233472   8622079   8388608     4G ChromeOS root fs
/dev/sda6        65        65         1   512B ChromeOS kernel
/dev/sda7        66        66         1   512B ChromeOS root fs
/dev/sda8     69632    102399     32768    16M Linux filesystem
/dev/sda9        67        67         1   512B ChromeOS reserved
/dev/sda10       68        68         1   512B ChromeOS reserved
/dev/sda11       64        64         1   512B unknown
/dev/sda12   102400    233471    131072    64M EFI System

Partition table entries are not in disk order.
root@debian:/mnt#
```

ここで、Start、Endの各数字はストレージのLBA(Logical Block Addressing)、Sectorsはブロック数(セクタ数)を示していて、1ブロックは512バイトです。/dev/sda2を例にすると、LBAの始まりが69、最後がLBAで32836、ブロック数は32768(=32836-69+1)となります。ブロック数が32768ですから32768×512÷1024÷1024=16となり容量は16MBとなります。 

パーティションテーブルを見たことのある人なら違和感を覚えると思いますが、ChromeOS Flexではパーティションが12個もあり、更にパーティションテーブルのインデックス(/dev/sdaXのXで示す数字がパーティションテーブルのインデックス)とストレージ上の物理的順番が一致していません。通常パーティションを作成する場合先頭から順に割り当てるので、`sfdisk --list`等で見ればスタートセクタの値は小さいものから順に並びます。しかしどういうわけかChromeOS Flexではこのようにバラバラの順番でパーティションが並んでいます。そのため`Partition table entries are not in disk order.`というメッセージも表示されています。

### ChromeOS Flexのパーティションテーブルのバックアップ

パーティションのインデックスと物理的順番が一致していないことが、この後WindowsやmacOSをインストールしたときにChromeOS Flexのパーティションテーブルが壊れる原因です。この後パーティションテーブルを編集するため、ChromeOS Flexインストール直後のパーティションテーブルを保存します。次の例は`sfdisk --dump`でパーティションテーブルを`p1-sda-dump`というファイルに保存しています。

```command
root@debian:/mnt# sfdisk --dump /dev/sda | tee p1-sda-dump
label: gpt
label-id: AC161E76-BF4B-924D-9C72-06CE3C6EABCF
device: /dev/sda
unit: sectors
first-lba: 34
last-lba: 234441614
sector-size: 512

/dev/sda1 : start=    17010688, size=   217430912, type=0FC63DAF-8483-4772-8E79-3D69D8477DE4, uuid=08FD7E32-17D4-7341-934F-06FE44B1237F, name="STATE"
/dev/sda2 : start=          69, size=       32768, type=FE3A2A5D-4F32-41A7-B725-ACCC3285A309, uuid=A8FC6629-5055-A040-A9A5-415A8EA50C1B, name="KERN-A", attrs="GUID:48,56"
/dev/sda3 : start=     8622080, size=     8388608, type=3CB8E202-3B7E-47DD-8A3C-7FF2A13CFCEC, uuid=358D4F01-CDE0-5D4C-BB65-E91160A04FE2, name="ROOT-A"
/dev/sda4 : start=       32837, size=       32768, type=FE3A2A5D-4F32-41A7-B725-ACCC3285A309, uuid=9F96728D-96A8-6745-892D-91084EAF2481, name="KERN-B", attrs="GUID:49,56"
/dev/sda5 : start=      233472, size=     8388608, type=3CB8E202-3B7E-47DD-8A3C-7FF2A13CFCEC, uuid=1030FFBB-8B4C-5049-9B22-20878DC32FE7, name="ROOT-B"
/dev/sda6 : start=          65, size=           1, type=FE3A2A5D-4F32-41A7-B725-ACCC3285A309, uuid=3AB9CA80-136D-1641-B222-3C0BBEC2B66D, name="KERN-C", attrs="GUID:52,53,54,55"
/dev/sda7 : start=          66, size=           1, type=3CB8E202-3B7E-47DD-8A3C-7FF2A13CFCEC, uuid=170F4F46-6D5F-ED48-82EB-39FEEE521751, name="ROOT-C"
/dev/sda8 : start=       69632, size=       32768, type=0FC63DAF-8483-4772-8E79-3D69D8477DE4, uuid=9E8B7A51-B372-5B44-B4D4-EB4ABE5F77CF, name="OEM"
/dev/sda9 : start=          67, size=           1, type=2E0A753D-9E48-43B0-8337-B15192CB1B5E, uuid=32CF3879-ABD4-B74D-A864-7E734A2DB89E, name="reserved"
/dev/sda10 : start=          68, size=           1, type=2E0A753D-9E48-43B0-8337-B15192CB1B5E, uuid=7433F1DE-C8AF-7D42-8A58-996671E0753D, name="reserved"
/dev/sda11 : start=          64, size=           1, type=CAB6E88E-ABF3-4102-A07A-D4BB9BE3C1D3, uuid=C147F3D7-07F4-0048-97B5-B4DAC1DF7120, name="RWFW"
/dev/sda12 : start=      102400, size=      131072, type=C12A7328-F81F-11D2-BA4B-00A0C93EC93B, uuid=FF81FED5-A756-3C44-9693-3E41EE823552, name="EFI-SYSTEM", attrs="LegacyBIOSBootable"
root@debian:/mnt# 
```

このリストからわかるようにGPTの各パーティションエントリには、先頭のLBA、サイズ(記録されているのはパーティションの最後のLBA)、パーティションのタイプを示すUUID(GUID)、パーティション固有のUUID、パーティションの名前、アトリビュートから構成されています。なおGPTでは最大で128個のパーティションを扱えます。

### WindowsやmacOS用の空き領域の確保

次のリストは最初に確認した`sfdisk --list /dev/sda`のパーティションテーブルをスタートセクタの小さい順つまり物理的順番に並び変え、説明のためパーティションの順番をIDとして追加してあります。

```command
ID : Device        Start       End   Sectors   Size Type
 1 : /dev/sda6        65        65         1   512B ChromeOS kernel
 2 : /dev/sda7        66        66         1   512B ChromeOS root fs
 3 : /dev/sda9        67        67         1   512B ChromeOS reserved
 4 : /dev/sda2        69     32836     32768    16M ChromeOS kernel
 5 : /dev/sda11       64        64         1   512B unknown
 6 : /dev/sda10       68        68         1   512B ChromeOS reserved
 7 : /dev/sda4     32837     65604     32768    16M ChromeOS kernel
 8 : /dev/sda8     69632    102399     32768    16M Linux filesystem
 9 : /dev/sda12   102400    233471    131072    64M EFI System
10 : /dev/sda5    233472   8622079   8388608     4G ChromeOS root fs
11 : /dev/sda3   8622080  17010687   8388608     4G ChromeOS root fs
12 : /dev/sda1  17010688 234441599 217430912 103.7G Linux filesystem
```

現時点ではストレージの全領域がChromeOS Flexに割り当てられているので、このままでは他のOSをインストールするための空き領域がありません。そこでポイントとなるのがこの中で最大の領域を使用している12番目にあるLinux filesystemのパーティションです。このパーティションはChromeOS Flexのユーザーデータ用で、パーティションインデックスでは/dev/sda1が示すように最初に割り当てられていますが、物理的には最後にあります(以降の説明では/dev/sdaXの/dev/は省略します)。sda1の領域を縮小できれば、残りを空き領域として他のOS等に割り当てられるようになります。

実は**sda1のファイルシステムはLinuxで一般的なEXT4で、サイズを変更して作り直しても問題なくChoromOS Flexが立ち上がることを確認しています。そこでsda1を縮小することで他のOS用の領域を確保します**。もちろん縮小して作り直すと書き込み済みのデータを失いますが、この時点では保存しなければならないようなデータを書き込んでいないので問題ないわけです。これが最初に書いた裏技ということになります。

### EFIシステムパーティションの変更

次にポイントとなるのが9番目にあるsda12のEFIシステムパーティション(ESP)です。ESPにはOSのブートローダーやカーネルなどが保存されます。ChromeOS FlexではESPに64Mバイト割り当てられていて物理的には9番目なのにデバイス名のsda12が示すようにパーティションインデックスは12で一番最後になります。

テストした結果ChromeOS FlexのESPの設定には二つの問題がありました。一つ目がパーティションサイズで、64MBではChromeOS Flexだけなら問題ないようですがWindowsとのマルチブートを行おうとすると不足気味です。実際Windowsのブートローダーと必要なファイルをインストールしてみると残りは2.7MB程度になりました。もし今後OSのアップデート等によってESPに保存するファイルが増えたり何かのファイルサイズが大きくなったりすると容量不足に陥る可能性があるため、将来に備えて容量を増やす必要があります。

二つ目はESPのパーティションインデックスと物理的順番が一致していないChromeOS Flexならでは問題で、macOS、WindowsともESPのストレージ上の配置は両方が一致していないと起動できないなどのトラブルにつながります。

そこで、ESPのパーティションの位置を変更するとともに容量を増やします。パーティションインデックスでは12で一番最後となっていますから、物理的にも12番目になるよう変更するわけです。移動させるといっても容量も変更するので、一旦容量の大きなESP用のFATパーティション(ESPのファイルシステムはFAT)を作成し、元のESPの内容をコピーします。

### パーティションテーブルの編集

パーティションテーブルの最初の編集は、ChromeOS Flexのユーザーデータ用パーティションを縮小し、それによって空いた領域の先頭部分にESPを置き換えるためのFATのパーティションを作成します。

まず、先ほど保存したp1-sda-dumpを別のファイルにコピーします。そしてFAT用パーティションを作るためにUUIDを1個用意します。

```command
root@debian:/mnt# cp p1-sda-dump p2-sda-dump
root@debian:/mnt# uuidgen
f2b1b3fc-81da-4ef8-9494-32dd9c0b20a0
root@debian:/mnt#
```

次にテキストエディタを使って、コピーしたp2-sda-dumpを編集します。元のp1-sda-dumpと編集後のp2-sda-dumpの差分は次のようになります。

```
root@debian:/mnt# vi p2-sda-dump
.....(省略).....
root@debian:/mnt# diff -U1 p1-sda-dump p2-sda-dump
--- p1-sda-dump 2023-04-08 22:51:03.041333000 +0900
+++ p2-sda-dump 2023-04-08 22:51:03.042949000 +0900
@@ -8,3 +8,3 @@

-/dev/sda1 : start=    17010688, size=   217430912, type=0FC63DAF-8483-4772-8E79-3D69D8477DE4, uuid=08FD7E32-17D4-7341-934F-06FE44B1237F, name="STATE"
+/dev/sda1 : start=    17010688, size=    33554432, type=0FC63DAF-8483-4772-8E79-3D69D8477DE4, uuid=08FD7E32-17D4-7341-934F-06FE44B1237F, name="STATE"
 /dev/sda2 : start=          69, size=       32768, type=FE3A2A5D-4F32-41A7-B725-ACCC3285A309, uuid=A8FC6629-5055-A040-A9A5-415A8EA50C1B, name="KERN-A", attrs="GUID:48,56"
@@ -20 +20,2 @@
 /dev/sda12 : start=      102400, size=      131072, type=C12A7328-F81F-11D2-BA4B-00A0C93EC93B, uuid=FF81FED5-A756-3C44-9693-3E41EE823552, name="EFI-SYSTEM", attrs="LegacyBIOSBootable"
+/dev/sda13 : start=    50565120, size=      524288, type=EBD0A0A2-B9E5-4433-87C0-68B6B72699C7, uuid=f2b1b3fc-81da-4ef8-9494-32dd9c0b20a0, name="DOS"
root@debian:/mnt# 
```

sda1の変更は単純にパーティションサイズを小さくするだけで、元は100GB以上あるので、16GBに変更するため 16 * 2^30 / 512 = 33554432 に設定しています。ChromeOS Flexにどのぐらいの容量を割り当てるべきなのかは使い方によって変わりますが、筆者の場合はChromeOS Flexのローカールストレージはほとんど使用しないので16GBで十分という判断です。sda1のサイズの縮小によって他のOSで87GB程利用できるようになります。

次にESPを置き換えるためのFATパーティション(ESPのファイルシステムはFAT16やFAT32)を一旦sda13に用意します。sda13のstartは物理的にはsda1の後になるのでsda1のstartとsizeを加えた 17010688 + 33554432 = 50565120 となります。ESPの容量は最低でも100MB程度は欲しいので余裕を見て256MBを設定しています。sda13のパーティションのtypeにはWindowsの一般的なデータパーティションを示す`EBD0A0A2-B9E5-4433-87C0-68B6B72699C7`を指定し、パーティションUUIDには先ほど用意した`f2b1b3fc-81da-4ef8-9494-32dd9c0b20a0`を設定します。パーティションの名前は無くても構わないのですが、一応"DOS"としてあります。

新規にパーティションを用意する場合、startとsizeは必ず8の倍数になるよう設定します。古いストレージではセクタサイズが512バイトでしたが、現在のストレージではセクタサイズは4096バイトが一般的で、従来と互換性をもたせるため512バイト単位でアクセスできるようになっています。これを区別するためにストレージ上のセクタを物理セクタ、アクセス可能なセクタを論理セクタと呼んでいて、物理セクタサイズを論理セクタサイズで割った値(つまり 4096 / 512 )が8というわけです。8の倍数にそろっていないと物理セクタとアクセスするセクタの境界が合わないことでパフォーマンスに悪影響があります。ちなみに本記事の例で使用しているSSDは少々古いものであるため物理セクタサイズも論理セクタと同じ512バイトです。

p2-sda-dumpの変更点を確認できたら、sfdiskコマンドを使ってストレージに書き込みます。

```commaned
root@debian:/mnt# sfdisk /dev/sda < p2-sda-dump
```

変更後のパーティションテーブルは次のようになります。

```commaned
root@debian:/mnt# sfdisk --list /dev/sda | tee p2-sda-list
Disk /dev/sda: 111.79 GiB, 120034123776 bytes, 234441648 sectors
Disk model: INTEL SSDSC2BW12
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: AC161E76-BF4B-924D-9C72-06CE3C6EABCF

Device        Start      End  Sectors  Size Type
/dev/sda1  17010688 50565119 33554432   16G Linux filesystem
/dev/sda2        69    32836    32768   16M ChromeOS kernel
/dev/sda3   8622080 17010687  8388608    4G ChromeOS root fs
/dev/sda4     32837    65604    32768   16M ChromeOS kernel
/dev/sda5    233472  8622079  8388608    4G ChromeOS root fs
/dev/sda6        65       65        1  512B ChromeOS kernel
/dev/sda7        66       66        1  512B ChromeOS root fs
/dev/sda8     69632   102399    32768   16M Linux filesystem
/dev/sda9        67       67        1  512B ChromeOS reserved
/dev/sda10       68       68        1  512B ChromeOS reserved
/dev/sda11       64       64        1  512B unknown
/dev/sda12   102400   233471   131072   64M EFI System
/dev/sda13 50565120 51089407   524288  256M Microsoft basic data

Partition table entries are not in disk order.
root@debian:/mnt# 
```

次にsda1とsda13にファイルシステムを作成します。sda1はEXT4ですからmkfs.ext4コマンドを使います。

```commaned
root@debian:/mnt# mkfs.ext4 -L H-STAGE /dev/sda1
mke2fs 1.46.2 (28-Feb-2021)
/dev/sda1 contains a ext4 file system labelled 'H-STATE'
        last mounted on /mnt/stateful_partition on Fri Mar 17 04:05:56 2023
Proceed anyway? (y,N) y
Discarding device blocks: done
Creating filesystem with 4194304 4k blocks and 1048576 inodes
Filesystem UUID: 5e407efa-1a3a-41f7-9d04-b036bb893ff6
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
        4096000

Allocating group tables: done
Writing inode tables: done
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done

root@debian:/mnt#
```

sda13はESP用ですからFAT32で作成します。

```commaned
root@debian:/mnt# mkdosfs -F 32 -n EFI-SYSTEM /dev/sda13
mkfs.fat 4.2 (2021-01-31)
root@debian:/mnt# 
```

## 既存ESPの置き換え

新しいESPの準備ができたので、sda12の内容をsda13にコピーします。ここでは双方のパーティションをマウントしてからtarを使ってコピーしていますが、cp -rなど他の方法でもかまいません。コピーが完了したら、sda12とsda13をアンマウントします。

```commaned
root@debian:/mnt# mkdir /mnt/efi /mnt/dos
root@debian:/mnt# mount /dev/sda12 /mnt/efi
root@debian:/mnt# mount /dev/sda13 /mnt/dos
root@debian:/mnt# (cd /mnt/efi ; tar cf - * ) | (cd /mnt/dos ; tar xvf -)
efi/
efi/boot/
efi/boot/bootx64.efi
.....(省略).....
syslinux/ldlinux.c32
root@debian:/mnt# umount /mnt/dos
root@debian:/mnt# umount /mnt/efi
```

sda13に新たなESPが用意できたので、既存のsda12を削除してsda13の領域をsda12のESPに変更します。しかしそれだけでは元のsda12の領域(sda8とsda5の間、IDで9の領域)に64MBの空きができてしまいます。Windowsの場合インストール時に16MBの予約パーティションを作るようで、ここが空いたままでは予約パーティションがsda8の次に作成されてしまい、せっかくパーティションインデックスと物理的順番を12で一致させたESPが物理的に13番目になってしまいます。そこで元sda12の空き領域を無くすため、sda8の領域をsda5の手前まで拡張します。

今度はp2-sda.dumpをp3-sda-dumpにコピーして、p3-sda-dumpを編集します。

```commaned
root@debian:/mnt# cp p2-sda-dump p3-sda-dump
root@debian:/mnt# vi p3-sda-dump
```

編集内容は次のようになります。

```commaned
root@debian:/mnt# diff -U0 p2-sda-dump p3-sda-dump
--- p2-sda-dump 2023-04-08 22:51:03.042949000 +0900
+++ p3-sda-dump 2023-04-23 21:43:45.434028000 +0900
@@ -16 +16 @@
-/dev/sda8 : start=       69632, size=       32768, type=0FC63DAF-8483-4772-8E79-3D69D8477DE4, uuid=9E8B7A51-B372-5B44-B4D4-EB4ABE5F77CF, name="OEM"
+/dev/sda8 : start=       69632, size=      163840, type=0FC63DAF-8483-4772-8E79-3D69D8477DE4, uuid=9E8B7A51-B372-5B44-B4D4-EB4ABE5F77CF, name="OEM"
@@ -20,2 +20 @@
-/dev/sda12 : start=      102400, size=      131072, type=C12A7328-F81F-11D2-BA4B-00A0C93EC93B, uuid=FF81FED5-A756-3C44-9693-3E41EE823552, name="EFI-SYSTEM", attrs="LegacyBIOSBootable"
-/dev/sda13 : start=    50565120, size=      524288, type=EBD0A0A2-B9E5-4433-87C0-68B6B72699C7, uuid=f2b1b3fc-81da-4ef8-9494-32dd9c0b20a0, name="DOS"
+/dev/sda12 : start=    50565120, size=      524288, type=C12A7328-F81F-11D2-BA4B-00A0C93EC93B, uuid=FF81FED5-A756-3C44-9693-3E41EE823552, name="EFI-SYSTEM", attrs="LegacyBIOSBootable"
root@debian:/mnt#
```

sda8は、元の16MBにsda12の64MBのサイズを加えた80MB(163840セクタ)に変更しています。sda12はtypeとuuidはそのままで、sda13のstartとsizeの値に変更します。そしてsda13は不要なので単純に削除します。

Macの場合はsda13を削除せず、typeをAPFSのGUIDである`7C3457EF-0000-11AA-AA11-00306543ECAC`、nameを`"Customer"`に変更します(変更は必須では無いと思いますが試していません)。さらにsda13がsda12の直後になるように、startにsda12のsizeとstartを加えた値を設定しsizeには残り容量に収まる範囲で適当な値(例えば50GB程度)を割り当てます。macOSのインストール時にsda13をAPFSでフォーマットする必要がありますが、フォーマットとともにsda13は残りの領域全てを含むサイズに拡張されるので厳密なサイズを設定する必要はありません。

正しく変更できているのを確認したら、再びsfdiskコマンドでパーティションテーブルを書き換えます。

```commaned
root@debian:/mnt# sfdisk /dev/sda < p3-sda-dump
.....(省略).....
root@debian:/mnt#
```

変更後のパーティションテーブルは次のようになります。

```commaned
root@debian:/mnt# sfdisk --list /dev/sda | tee p3-sda-list
Disk /dev/sda: 111.79 GiB, 120034123776 bytes, 234441648 sectors
Disk model: INTEL SSDSC2BW12
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: AC161E76-BF4B-924D-9C72-06CE3C6EABCF

Device        Start      End  Sectors  Size Type
/dev/sda1  17010688 50565119 33554432   16G Linux filesystem
/dev/sda2        69    32836    32768   16M ChromeOS kernel
/dev/sda3   8622080 17010687  8388608    4G ChromeOS root fs
/dev/sda4     32837    65604    32768   16M ChromeOS kernel
/dev/sda5    233472  8622079  8388608    4G ChromeOS root fs
/dev/sda6        65       65        1  512B ChromeOS kernel
/dev/sda7        66       66        1  512B ChromeOS root fs
/dev/sda8     69632   233471   163840   80M Linux filesystem
/dev/sda9        67       67        1  512B ChromeOS reserved
/dev/sda10       68       68        1  512B ChromeOS reserved
/dev/sda11       64       64        1  512B unknown
/dev/sda12 50565120 51089407   524288  256M EFI System

Partition table entries are not in disk order.
root@debian:/mnt# 
```

この状態でsda8は、実際に使われている領域よりパーティションサイズが大きいことになりますが問題ありません。

最初に保存したp1-sda-listと比較すると次のようになります。

```commaned
root@debian:/mnt# diff -U0 --ignore-space-change p1-sda-list p3-sda-list
--- p1-sda-list 2023-04-08 22:51:03.042130000 +0900
+++ p3-sda-list 2023-04-08 22:51:03.044934000 +0900
@@ -10 +10 @@
-/dev/sda1  17010688 234441599 217430912 103.7G Linux filesystem
+/dev/sda1  17010688 50565119 33554432   16G Linux filesystem
@@ -17 +17 @@
-/dev/sda8     69632    102399     32768    16M Linux filesystem
+/dev/sda8     69632   233471   163840   80M Linux filesystem
@@ -21 +21 @@
-/dev/sda12   102400    233471    131072    64M EFI System
+/dev/sda12 50565120 51089407   524288  256M EFI System
```

これでWindowsやmacOSなどの他のOSをインストールする準備ができたので、PCをシャットダウンします。Debian LiveのUSBメモリとパーティションテーブルのメモをとった記録用のUSBメモリは後の作業で使用するのでそのまま保管してください。

```commaned
root@debian:/mnt# poweroff
```

この後PCの電源を入れるとChromeOS Flexの起動を確認できます。もし起動しない場合は、今までの手順を見直します。Debian Liveを使って修復するか、またはChromeOS Flexのインストールからやり直すことになるかもしれません。

## WindowsまたはmacOSのインストール

今度はWindowsやmacOSのインストール用USBメモリを使ってPCを起動し、対象OSをインストールします。インストール時に注意するのは、今までの作業で用意したChromeOS Flexのパーティションを壊さないよう、ストレージの適正な領域(Windowsの場合なら空き領域、Macであれば用意したsda13に該当するパーティション)にインストールすることです。あとは通常のmacOS、Windowsのインストールとなんら変わりありません。

インストールが終了したらOSが動くことを確認します。この時点では**WindowsやmacOS等インストールしたOSは起動しますが、ChromeOS Flexは起動できなくなっています**。理由は最初のところでも書いたように、WindowsやmacOSがインストールのためにパーティションテーブルの変更を行うと、ChromeOS Flexのパーティションテーブルを書き換えてしまうためです。

## ChromeOS FLex用パーティションテーブルの修復

再びDebian LiveのUSBメモリで起動し、起動しなくなっているChromeOS Flexのパーティションテーブルの修復を行います。Debian Liveが起動したら、前の作業で記録をとったUSBメモリを/mntにマウントします。なおaptコマンドを使うことはありません。

```commaned
user@debian:~$ sudo -i
root@debian:~# mount /dev/sdc1 /mnt
root@debian:~# cd /mnt
root@debian:/mnt# 
```

起動しなくなったパーティションテーブルの状況を`sfdisk --list`で確認します。

```commaned
root@debian:/mnt# sfdisk --list /dev/sda | tee p4-sda-list
Disk /dev/sda: 111.79 GiB, 120034123776 bytes, 234441648 sectors
Disk model: INTEL SSDSC2BW12
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: AC161E76-BF4B-924D-9C72-06CE3C6EABCF

Device        Start       End   Sectors  Size Type
/dev/sda1        64        64         1  512B unknown
/dev/sda2        65        65         1  512B ChromeOS kernel
/dev/sda3        66        66         1  512B ChromeOS root fs
/dev/sda4        67        67         1  512B ChromeOS reserved
/dev/sda5        68        68         1  512B ChromeOS reserved
/dev/sda6        69     32836     32768   16M ChromeOS kernel
/dev/sda7     32837     65604     32768   16M ChromeOS kernel
/dev/sda8     69632    233471    163840   80M Linux filesystem
/dev/sda9    233472   8622079   8388608    4G ChromeOS root fs
/dev/sda10  8622080  17010687   8388608    4G ChromeOS root fs
/dev/sda11 17010688  50565119  33554432   16G Linux filesystem
/dev/sda12 50565120  51089407    524288  256M EFI System
/dev/sda13 51089408  51122175     32768   16M Microsoft reserved
/dev/sda14 51122176 234440703 183318528 87.4G Microsoft basic data
```

sda13とsda14がWindowsのインストールによって追加されたパーティションですが、さらにsda15も増えている場合があります。Macの場合は事前にsda13を用意しているのでパーティション数に変化は無いですが、sda13に関してはsize(Sectors)が変わっているはずです。

ChromeOS Flexインストール時点の1から12を見ると、sda12は事前にパーティションインデックスと物理的順番を一致するように対処してあったので変化はありません。しかしsda1からsda11までは、ストレージ上の物理的順番にパーティションテーブルが並んでしまっています。どうやらWindowsやmacOSでは、パーティションの追加等でパーティションテーブルを書き換えると物理的順番に並べ替えてしまうようです。このままではChromeOS Flexが起動できないので、パーティションテーブルの並び順を元の順番に修正します。

まず`sfdisk --dump`で現状のパーティションテーブルをp4-sda-dumpに保存します。

```commaned
root@debian:/mnt# sfdisk --dump /dev/sda > p4-sda-dump
```

p4-sda-dumpを前に保存したp3-sda-dumpと比べると、順番は変わっているもののパーティションの大きさやtype, uuidの情報は変わっていないことがわかります。そこで今回追加されたsda13, sda14(あるいはsda15まで)のパーティション情報をp3-sda-dumpに追加してストレージに反映すれば、ChromeOS Flexのパーティションテーブルを修復できます。

修復するためのパーティションテーブルは、エディタを使うこともなく次のコマンドラインで作成できます。

```commaned
root@debian:/mnt# cp p3-sda-dump p5-sda-dump
root@debian:/mnt# tail -n 2 p4-sda-dump >> p5-sda-dump
```

2つめの`tail -n`のパラメータは、増えたパーティション数に合わせて1, 2, 3等を指定します。Macの場合はsda13を事前に用意しているため、sda13の行をmacOSインストール後のものと置き換えます。p5-sda-dumpの修正ができたら元になったp3-sda-dumpとdiffコマンドで比較して、正しく変更できたかどうかを確認します。

```commaned
root@debian:/mnt# diff -U1  p3-sda-dump p5-sda-dump
--- p3-sda-dump 2023-04-26 12:30:22.078533000 +0900
+++ p5-sda-dump 2023-04-26 12:30:22.078899000 +0900
@@ -20 +20,3 @@
 /dev/sda12 : start=    50565120, size=      524288, type=C12A7328-F81F-11D2-BA4B-00A0C93EC93B, uuid=FF81FED5-A756-3C44-9693-3E41EE823552, name="EFI-SYSTEM", attrs="LegacyBIOSBootable"
+/dev/sda13 : start=    51089408, size=       32768, type=E3C9E316-0B5C-4DB8-817D-F92DF00215AE, uuid=46FFCC48-8858-4FF6-BE06-E352C55E9FD1, name="Microsoft reserved partition", attrs="GUID:63"
+/dev/sda14 : start=    51122176, size=   183318528, type=EBD0A0A2-B9E5-4433-87C0-68B6B72699C7, uuid=2C6B3DFE-8DF4-4BF1-8C1F-01173A1042E1, name="Basic data partition"
root@debian:/mnt# 
```

問題なければ修正したパーティションテーブルをストレージに反映します。

```commaned
root@debian:/mnt# sfdisk /dev/sda < p5-sda-dump
```

これでChromeOS Flexと他OSのマルチブート環境の設定は完了となりますが、将来WindowsやmacOSでのパーティションの変更でChromeOS Flexのパーティションが壊れた時に備えて途中で用意したp3-sda-dumpを保管しておいてください。

## 起動画面のイメージ

実際にPCを起動してみましょう。

### Windows PCの場合

Windows PCの場合は起動時にUEFI BIOSのブートセレクタを選ぶと、こんな感じで「ChromeOF Flex」と「Windows Boot Manager」が選べると思います。

![boot-Windows.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/130638/8a6b8c32-21ca-5632-deab-57c1433a51dd.jpeg)

このPCではFreeBSDやDebian GNU/Linuxもインストールしてあり4種類のOSを切り替えて利用できるようになっています。

### Macの場合

Macの場合はOptionキーを押しながら起動すれば次のドライブの選択画面が表示されますが、このうち「EFI Boot」がChromeOS Flexとなります。

![boot-macOS.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/130638/d0f93100-bf58-ed64-5637-f0d3da7005e8.jpeg)

Macでは起動OSを選ぶ時にControlキーを押しながらクリックすれば、次回からのデフォルトの起動OSを変更できます。

## RTCの時刻の扱いの違い

macOSでは問題無いのですが、WindowsとChromeOS Flexを組み合わせた場合はRTC(Real Time Clock, PCが内蔵しているBIOSで確認できる時計)の時刻の扱いの違いが問題になります。ChromeOS FlexではRTCの時刻をUTCとして扱うのに対して、Windowsではローカルの時刻つまりJSTとして扱います。そのため何もしないとChromeOS Flexを使った後にWindowsを起動すると、時刻が9時間ずれてしまいます。この問題は次のレジストリを設定すると、WindowsがRTCをUTCで扱うようになって回避できます。

```commaned
Windows Registry Editor Version 5.00

[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\TimeZoneInformation]
"RealTimeIsUniversal"=dword:00000001
```

レジストリの修正はリスクを伴いますので、くれぐれも慎重に操作してください。

## マルチブート環境構築後のパーティションテーブルの管理と修復

本文中でインストールしたWindowsはWindows 10で、インストール後にsda13とsda14の2つのパーティションが増えました。後日Windows 11にアップグレードしたところsda14のサイズが縮小されて新たにsda15として回復パーティション追加になったとともにパーティションテーブルが書き換えられてしまい、ChromeOS Flexが起動できなくなりました。その修復のため、改めてDebian Liveを使って作業中に保存したp3-sda-dumpを元にしてパーティションの修復を行う羽目になりました。Windowsの回復パーティションについては必ずインストール時点で作られると思っていたのですが、そうでは無い場合もあるようです。

このようにWindowsやmacOSによってパーティションに変更が行われると、ほぼ間違い無くChromeOS Flexのパーティションテーブルは破壊されるので十分注意する必要があります。

またChromeOS FlexではABアップデートを行うため、アップデート毎にパーティションテーブルのアトリビュートが書き換えられます。アトリビュートの変更と言っても入れ替えだけなので、最新のバックアップで無いパーティションテーブルを戻すと一つ古いOSになるだけだと思いますが、できれば定期的に最新のパーティションテーブルを保存するようにするのが望ましいです。同じPCに3つ目のOSとしてDebian等のLinux環境をインストールしておくと、最新のパーティションテーブルのバックアップといざという時の修復が簡単になります。

## おわりに

筆者の場合、以前にChromeOS Flexを利用したのはかなり古いPCであったためパフォーマンス面ではあまり快適な環境ではありませんでした。今回はWindowsで使っても実用的なパフォーマンスがあるPCなのでChromeOS Flexも快適に利用でき、余計なアプリが動かないこともあってか動作は軽く、起動や終了の速さも特筆ものでした。

WindowsやmacOSが快適に使えるPCでわざわざChromeOS Flexを使うメリットはあまり無さそうですが、CPUはそこそこなのにメモリが少ない場合(4GB程度)にChromeOS Flexを選ぶのは最適な選択肢かもしれません。
