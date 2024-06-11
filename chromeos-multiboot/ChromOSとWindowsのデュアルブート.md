<!-- https://qiita.com/belgianbeer/items/b638c12150dc86911922 -->

# ChromeOS FlexとWindowsやmacOS等のマルチブート環境を作る

> 【2024-06-11 追記】久しぶりにChromeOS Flexのマルチブート環境を作ってみたところ本記事のオリジナルの公開時点からChromeOS Flexのインストーラに変更があり、以前のままではWindowsでうまくいかないことがわかりました。また改めて作業したことで、Windowsでインストール後のパーティション修復を不要にできることにも気付きました。さらに筆者は普段英語キーボードなので気づいてなかったのですが、Debian Liveでは英語キーボードの配列となるため日本語キーボードのユーザーへ注意が抜けていました。それらの点を踏まえて記事を更新しました。

## はじめに

ChromeOS Flexは1台のPCで他のOSとのマルチブートをサポートしていません。というのもインストール時にストレージの使用量を調整する手段が無く、既存のOSの領域を削除した上で全領域をChromeOS Flex用として確保してしまいます。そのため他のOSをインストールするための空き領域を確保できないわけです。

ここでは裏技的な手法を使って、1台のPCにChromeOS Flexと他のOSとのマルチブート環境を構築する方法について説明します。あくまでも裏技なので将来にわたってこの方法が通用するかどうかは不明ですが、当面は問題無いでしょう。

しかしながら、**ChromeOS Flexと他のOSのマルチブート環境はあまりお勧めとは言えません**。というのもChromeOS Flexとのマルチブート環境のPCで**WindowsやmacOSがパーティションの変更を行うと、ChromeOS Flex用のパーティションテーブルが破壊される為です**。実際この記事を書きながら設定したPCでも、マルチブート環境構築後にパーティションテーブルが破壊されるという事態が発生しました(後で紹介します)。これはWindowsやmacOSが悪いというより、ChromeOS Flexのパーティションテーブルが少し変わった構成になっていることによります。もし破壊されてもパーティションテーブルのバックアップがあれば修復できるのですが、ここで紹介するマルチブート環境を使い続けるのであれば、トラブル時に速やかに対処を行うセンスが要求されます。

マルチブート環境は既存のWindowsやMacの環境を一旦削除して作り直すものです。ストーレジのパーティションテーブルの理解が不十分だと、途中で失敗する可能性は多いにあります。実行する場合は自己責任でお願いします。

## 用意するもの

ChromeOS Flexとのマルチブート環境を作るのに必要なものは次の通りです。

1. UEFIをサポートしたWindows PCまたはMac(Intel CPU)本体
1. ChromeOS Flexのインストール用USBメモリ(8GB以上)
1. Windows PCであればWindows 10か11のインストール用USBメモリ(8GB以上)、Macの場合はmacOSのインストール用USBメモリ(16GB以上)
1. Debian Liveの起動用USBメモリ(2GB以上)
1. 記録用のUSBメモリ(100KB程度の空きがあれば十分)

UEFIをサポートしていないPCでもChromeOS Flexだけなら問題なく動作します。しかし本記事では他OSとのブートの切り替えにUEFIで用意されているブートセレクタを使うことを前提にしているため、GPT(GUID Partition Table)を扱えるUEFI対応のPCが必須となります。Intel CPUのMacはUEFIをサポートしているので問題ありません。少し古いWindows PCの場合は、**UEFIをサポートしているのにデフォルトでは無効になっている**ことがあります。その場合BIOSの設定でUEFIの機能を有効にする必要があります。ただしUEFIを有効に切り替えると既存のWindowsが起動しなくなることがあるので、切り替えはChromOS Flexをインストールするときに行います。

Windows PC、Macのいずれを利用するにしても、**ChromeOS Flexをインストールすると既存データは削除されてしまいますので、事前のバックアップは必須**です。さらにWindows PCの場合は[回復ドライブ](https://support.microsoft.com/ja-jp/windows/%E5%9B%9E%E5%BE%A9%E3%83%89%E3%83%A9%E3%82%A4%E3%83%96%E3%82%92%E4%BD%9C%E6%88%90%E3%81%99%E3%82%8B-abb4691b-5324-6d4a-8766-73fab304c246)を作成し、最悪の場合に出荷時の設定に戻せるよう準備しておくことをお勧めします。

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

必須ではないですが、openssh-serverのパッケージをインストールすると別のPCからsshでログインできるようになります。`ip addr show`でIPアドレスを確認しsshで対象のPCにログインすれば、テキストのコピーやペーストができるようになって作業効率があがります。それに**Debian Liveでは英語キーボードの配列になる**ため、普段日本語キーボードしか利用していない人にとっては一部の記号の入力が困難になります。その場合もsshを利用して使い慣れているキーボードの機器からログインしたほうが作業がやりやすくなります。なおsshでログインするときのユーザー名は「user」、パスワードは「live」です。

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
Disk identifier: 375C8B32-FD8D-464E-91A6-FB83A244B8C4

Device        Start       End   Sectors   Size Type
/dev/sda1  17272832 234441599 217168768 103.6G Linux filesystem
/dev/sda2        69     32836     32768    16M ChromeOS kernel
/dev/sda3   8884224  17272831   8388608     4G ChromeOS root fs
/dev/sda4     32837     65604     32768    16M ChromeOS kernel
/dev/sda5    495616   8884223   8388608     4G ChromeOS root fs
/dev/sda6        65        65         1   512B ChromeOS kernel
/dev/sda7        66        66         1   512B ChromeOS root fs
/dev/sda8    331776    364543     32768    16M Linux filesystem
/dev/sda9        67        67         1   512B ChromeOS reserved
/dev/sda10       68        68         1   512B ChromeOS reserved
/dev/sda11       64        64         1   512B unknown
/dev/sda12   364544    495615    131072    64M EFI System

Partition table entries are not in disk order.
root@debian:/mnt#
```

ここで、Start、Endの各数字はストレージのLBA(Logical Block Addressing)、Sectorsはブロック数(セクタ数)を示していて、1ブロックは512バイトです。/dev/sda2を例にすると、LBAの始まりが69、最後がLBAで32836、ブロック数は32768(=32836-69+1)となります。ブロック数が32768ですから 32768 \* 512 / 1024 / 1024 = 16 となり容量は16MBとなります。

パーティションテーブルを見たことのある人なら違和感を覚えると思いますが、ChromeOS Flexではパーティションが12個もあり、更にパーティションテーブルのインデックス(/dev/sdaXのXで示す数字がパーティションテーブルのインデックス)とストレージ上の物理的順番が一致していません。通常パーティションを作成する場合先頭から順に割り当てるので、`sfdisk --list`等で見ればスタートセクタの値は小さいものから順に並びます。しかしどういうわけかChromeOS Flexではこのようにバラバラの順番でパーティションが並んでいます。そのため`Partition table entries are not in disk order.`というメッセージも表示されています。

### ChromeOS Flexのパーティションテーブルのバックアップ

パーティションのインデックスと物理的順番が一致していないことが、この後WindowsやmacOSをインストールしたときにChromeOS Flexのパーティションテーブルが壊れる原因です。この後パーティションテーブルを編集するため、ChromeOS Flexインストール直後のパーティションテーブルを保存します。次の例は`sfdisk --dump`でパーティションテーブルを`p1-sda-dump`というファイルに保存しています。

```command
root@debian:/mnt# sfdisk --dump /dev/sda | tee p1-sda-dump
label: gpt
label-id: 375C8B32-FD8D-464E-91A6-FB83A244B8C4
device: /dev/sda
unit: sectors
first-lba: 34
last-lba: 234441614
sector-size: 512

/dev/sda1 : start=    17272832, size=   217168768, type=0FC63DAF-8483-4772-8E79-3D69D8477DE4, uuid=ABB70E5B-29DC-E64D-ADFC-4C85AF8FF5AB, name="STATE"
/dev/sda2 : start=          69, size=       32768, type=FE3A2A5D-4F32-41A7-B725-ACCC3285A309, uuid=53F298F1-1E89-A940-A8E0-8D82BCC9701F, name="KERN-A", attrs="GUID:48,53,54,56"
/dev/sda3 : start=     8884224, size=     8388608, type=3CB8E202-3B7E-47DD-8A3C-7FF2A13CFCEC, uuid=9FD0B508-505C-5547-96F6-F4331FD31A96, name="ROOT-A"
/dev/sda4 : start=       32837, size=       32768, type=FE3A2A5D-4F32-41A7-B725-ACCC3285A309, uuid=D9537851-4016-DF43-8382-52E0ECEE0401, name="KERN-B", attrs="GUID:52,53,54,55"
/dev/sda5 : start=      495616, size=     8388608, type=3CB8E202-3B7E-47DD-8A3C-7FF2A13CFCEC, uuid=DDACFDD2-0D0E-474B-84EE-4039BD636DBD, name="ROOT-B"
/dev/sda6 : start=          65, size=           1, type=FE3A2A5D-4F32-41A7-B725-ACCC3285A309, uuid=70CDA11D-CC98-7C40-ADA0-82C445DA3C3D, name="KERN-C", attrs="GUID:52,53,54,55"
/dev/sda7 : start=          66, size=           1, type=3CB8E202-3B7E-47DD-8A3C-7FF2A13CFCEC, uuid=DD34BE00-E29C-0046-96E7-4A1A5560ED1C, name="ROOT-C"
/dev/sda8 : start=      331776, size=       32768, type=0FC63DAF-8483-4772-8E79-3D69D8477DE4, uuid=0C70F5EF-DE31-C945-8F15-B7A54834ADB6, name="OEM"
/dev/sda9 : start=          67, size=           1, type=2E0A753D-9E48-43B0-8337-B15192CB1B5E, uuid=083B9722-78BC-D243-AE53-381646ABBA34, name="reserved"
/dev/sda10 : start=          68, size=           1, type=2E0A753D-9E48-43B0-8337-B15192CB1B5E, uuid=4C78ED32-C8AA-F34E-A0E0-FD70762E6DB5, name="reserved"
/dev/sda11 : start=          64, size=           1, type=CAB6E88E-ABF3-4102-A07A-D4BB9BE3C1D3, uuid=C108F1BC-E56E-5F40-B929-B3F1679BBAD6, name="RWFW"
/dev/sda12 : start=      364544, size=      131072, type=C12A7328-F81F-11D2-BA4B-00A0C93EC93B, uuid=F4CD81A7-771B-B54D-89B9-DCA60C886D4F, name="EFI-SYSTEM", attrs="LegacyBIOSBootable"
root@debian:/mnt# 
```

このリストからわかるようにGPTの各パーティションエントリには、先頭のLBA、サイズ(記録されているのはパーティションの最後のLBA)、パーティションのタイプを示すUUID(GUID)、パーティション固有のUUID、パーティションの名前、アトリビュートから構成されています。なおGPTでは最大で128個のパーティションを扱えます。

### WindowsやmacOS用の空き領域の確保

次のリストは最初に確認した`sfdisk --list /dev/sda`のパーティションテーブルをスタートセクタの小さい順つまり物理的順番に並び変え、説明のためパーティションの順番をIDとして追加してあります。

```command
ID : Device        Start       End   Sectors   Size Type
 1 : /dev/sda11       64        64         1   512B unknown
 2 : /dev/sda6        65        65         1   512B ChromeOS kernel
 3 : /dev/sda7        66        66         1   512B ChromeOS root fs
 4 : /dev/sda9        67        67         1   512B ChromeOS reserved
 5 : /dev/sda10       68        68         1   512B ChromeOS reserved
 6 : /dev/sda2        69     32836     32768    16M ChromeOS kernel
 7 : /dev/sda4     32837     65604     32768    16M ChromeOS kernel
 8 : /dev/sda8    331776    364543     32768    16M Linux filesystem
 9 : /dev/sda12   364544    495615    131072    64M EFI System
10 : /dev/sda5    495616   8884223   8388608     4G ChromeOS root fs
11 : /dev/sda3   8884224  17272831   8388608     4G ChromeOS root fs
12 : /dev/sda1  17272832 234441599 217168768 103.6G Linux filesystem
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

まず、先ほど保存したp1-sda-dumpを別のファイルにコピーします。そしてFAT用パーティションを作るためにUUIDを1個用意します(本記事からコピペできないようUUIDの一部をマスクしています)。

```command
root@debian:/mnt# cp p1-sda-dump p2-sda-dump
root@debian:/mnt# uuidgen
91b804e1-8930-48ee-817d-c42exx9b95b1
root@debian:/mnt#
```

次にテキストエディタを使って、コピーしたp2-sda-dumpを編集します。元のp1-sda-dumpと編集後のp2-sda-dumpの差分は次のようになります。

```
root@debian:/mnt# vi p2-sda-dump
.....(省略).....
root@debian:/mnt# diff -U0 p1-sda-dump p2-sda-dump
--- p1-sda-dump 2024-05-31 04:58:38.000000000 +0000
+++ p2-sda-dump 2024-05-31 05:04:58.000000000 +0000
@@ -9 +9 @@
-/dev/sda1 : start=    17272832, size=   217168768, type=0FC63DAF-8483-4772-8E79-3D69D8477DE4, uuid=ABB70E5B-29DC-E64D-ADFC-4C85AF8FF5AB, name="STATE"
+/dev/sda1 : start=    17272832, size=    33554432, type=0FC63DAF-8483-4772-8E79-3D69D8477DE4, uuid=ABB70E5B-29DC-E64D-ADFC-4C85AF8FF5AB, name="STATE"
@@ -20,0 +21 @@
+/dev/sda13 : start=    50827264, size=      524288, type=EBD0A0A2-B9E5-4433-87C0-68B6B72699C7, uuid=91b804e1-8930-48ee-817d-c42exx9b95b1, name="DOS"
root@debian:/mnt# 
```

sda1の変更は単純にパーティションサイズを小さくするだけで、元は100GB以上あるので、16GBに変更するため 16 \* 1024 \* 1024 \* 1024 / 512 = **33554432** に設定しています。ChromeOS Flexにどのぐらいの容量を割り当てるべきなのかは使い方によって変わりますが、筆者の場合はChromeOS Flexのローカールストレージはほとんど使用しないので16GBで十分という判断です。sda1のサイズの縮小によって他のOSで87GB程利用できるようになります。

次にESPを置き換えるためのFATパーティション(ESPのファイルシステムはFAT16やFAT32)を一旦sda13に用意します。sda13のstartは物理的にはsda1の後になるのでsda1のstartとsizeを加えた 17272832 + 33554432 = **50827264** となります。ESPの容量は最低でも100MB程度は欲しいので余裕を見て256MBを設定しています。sda13のパーティションのtypeにはWindowsの一般的なデータパーティションを示す`EBD0A0A2-B9E5-4433-87C0-68B6B72699C7`を指定し、パーティションUUIDには先ほど用意した`91b804e1-8930-48ee-817d-c42exx9b95b1`を設定します。パーティションの名前は無くても構わないのですが、一応"DOS"としてあります。

新規にパーティションを用意する場合、startとsizeは必ず8の倍数になるよう設定します。古いストレージではセクタサイズが512バイトでしたが、現在のストレージではセクタサイズは4096バイトが一般的で、従来と互換性をもたせるため512バイト単位でアクセスできるようになっています。これを区別するためにストレージ上のセクタを物理セクタ、アクセス可能なセクタを論理セクタと呼んでいて、物理セクタサイズを論理セクタサイズで割った値(つまり 4096 / 512 )が8というわけです。8の倍数にそろっていないと物理セクタとアクセスするセクタの境界が合わないことでパフォーマンスに悪影響があります。ちなみに本記事の例で使用しているSSDは少々古いものであるため物理セクタサイズも論理セクタと同じ512バイトです。

p2-sda-dumpの変更点を確認できたら、sfdiskコマンドを使ってストレージに書き込みます。

```command
root@debian:/mnt# sfdisk /dev/sda < p2-sda-dump
```

変更後のパーティションテーブルは次のようになります。

```command
root@debian:/mnt# sfdisk --list /dev/sda | tee p2-sda-list
Disk /dev/sda: 111.79 GiB, 120034123776 bytes, 234441648 sectors
Disk model: INTEL SSDSC2BW12
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 375C8B32-FD8D-464E-91A6-FB83A244B8C4

Device        Start      End  Sectors  Size Type
/dev/sda1  17272832 50827263 33554432   16G Linux filesystem
/dev/sda2        69    32836    32768   16M ChromeOS kernel
/dev/sda3   8884224 17272831  8388608    4G ChromeOS root fs
/dev/sda4     32837    65604    32768   16M ChromeOS kernel
/dev/sda5    495616  8884223  8388608    4G ChromeOS root fs
/dev/sda6        65       65        1  512B ChromeOS kernel
/dev/sda7        66       66        1  512B ChromeOS root fs
/dev/sda8    331776   364543    32768   16M Linux filesystem
/dev/sda9        67       67        1  512B ChromeOS reserved
/dev/sda10       68       68        1  512B ChromeOS reserved
/dev/sda11       64       64        1  512B unknown
/dev/sda12   364544   495615   131072   64M EFI System
/dev/sda13 50827264 51351551   524288  256M Microsoft basic data

Partition table entries are not in disk order.
root@debian:/mnt# 
```

次にsda1とsda13にファイルシステムを作成します。sda1はEXT4ですからmkfs.ext4コマンドを使います。

```command
root@debian:/mnt# mkfs.ext4 -L H-STAGE /dev/sda1
mke2fs 1.47.0 (5-Feb-2023)
/dev/sda1 contains a ext4 file system labelled 'H-STATE'
        last mounted on /tmp/install-mount-point on Fri May 31 04:46:18 2024
Proceed anyway? (y,N) y
Discarding device blocks: done
Creating filesystem with 4194304 4k blocks and 1048576 inodes
Filesystem UUID: 81141d32-e853-4dbe-baed-48f96c2f7bea
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

```command
root@debian:/mnt# mkdosfs -F 32 -n EFI-SYSTEM /dev/sda13
mkfs.fat 4.2 (2021-01-31)
root@debian:/mnt# 
```

## 既存ESPの置き換え

新しいESPの準備ができたので、sda12の内容をsda13にコピーします。ここでは双方のパーティションをマウントしてからtarを使ってコピーしていますが、cp -rなど他の方法でもかまいません。コピーが完了したら、sda12とsda13をアンマウントします。

```command
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

物理的な順番としてsda1の次にsda13として新たなESPが用意できたので、既存のsda12を削除してsda13の領域をsda12のESPに変更します。編集内容は、sda12のスタートとサイズにsda13に設定したものをコピーして、sda13の行を削除します。今度はp2-sda.dumpをp3-sda-dumpにコピーして、p3-sda-dumpを編集します。

```command
root@debian:/mnt# cp p2-sda-dump p3-sda-dump
root@debian:/mnt# vi p3-sda-dump
```

編集後の変更内容は次のようになります。

```command
root@debian:/mnt# diff -U0 p2-sda-dump p3-sda-dump
--- p2-sda-dump 2024-05-31 05:04:58.000000000 +0000
+++ p3-sda-dump 2024-05-31 05:26:10.000000000 +0000
@@ -20,2 +20 @@
-/dev/sda12 : start=      364544, size=      131072, type=C12A7328-F81F-11D2-BA4B-00A0C93EC93B, uuid=F4CD81A7-771B-B54D-89B9-DCA60C886D4F, name="EFI-SYSTEM", attrs="LegacyBIOSBootable"
-/dev/sda13 : start=    50827264, size=      524288, type=EBD0A0A2-B9E5-4433-87C0-68B6B72699C7, uuid=91b804e1-8930-48ee-817d-c42e759b95b1, name="DOS"
+/dev/sda12 : start=    50827264, size=      524288, type=C12A7328-F81F-11D2-BA4B-00A0C93EC93B, uuid=F4CD81A7-771B-B54D-89B9-DCA60C886D4F, name="EFI-SYSTEM", attrs="LegacyBIOSBootable"
root@debian:/mnt#
```

正しく変更できているのを確認したら、再びsfdiskコマンドでパーティションテーブルを書き換えます。

```command
root@debian:/mnt# sfdisk /dev/sda < p3-sda-dump
.....(省略).....
root@debian:/mnt#
```

変更後のパーティションテーブルは次のようになります。

```command
root@debian:/mnt# sfdisk --list /dev/sda | tee p3-sda-list
Disk /dev/sda: 111.79 GiB, 120034123776 bytes, 234441648 sectors
Disk model: INTEL SSDSC2BW12
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 375C8B32-FD8D-464E-91A6-FB83A244B8C4

Device        Start      End  Sectors  Size Type
/dev/sda1  17272832 50827263 33554432   16G Linux filesystem
/dev/sda2        69    32836    32768   16M ChromeOS kernel
/dev/sda3   8884224 17272831  8388608    4G ChromeOS root fs
/dev/sda4     32837    65604    32768   16M ChromeOS kernel
/dev/sda5    495616  8884223  8388608    4G ChromeOS root fs
/dev/sda6        65       65        1  512B ChromeOS kernel
/dev/sda7        66       66        1  512B ChromeOS root fs
/dev/sda8    331776   364543    32768   16M Linux filesystem
/dev/sda9        67       67        1  512B ChromeOS reserved
/dev/sda10       68       68        1  512B ChromeOS reserved
/dev/sda11       64       64        1  512B unknown
/dev/sda12 50827264 51351551   524288  256M EFI System

Partition table entries are not in disk order.
root@debian:/mnt# 
```
<!-- 
この状態でsda8は、実際に使われている領域よりパーティションサイズが大きいことになりますが問題ありません。
-->

最初に保存したp1-sda-listと比較すると次のようになります。

```command
root@debian:/mnt# diff -U0 --ignore-space-change p1-sda-list p3-sda-list
--- p1-sda-list 2024-05-31 04:58:24.000000000 +0000
+++ p3-sda-list 2024-05-31 05:33:18.000000000 +0000
@@ -10 +10 @@
-/dev/sda1  17272832 234441599 217168768 103.6G Linux filesystem
+/dev/sda1  17272832 50827263 33554432   16G Linux filesystem
@@ -21 +21 @@
-/dev/sda12   364544    495615    131072    64M EFI System
+/dev/sda12 50827264 51351551   524288  256M EFI System
root@debian:/mnt# 
```

sda1のサイズが103.6GBから16GBと小さくなり、sda12は物理的にsda1の次の場所になって64MBから256MBに大きくなっているのがわかります。

## OS用パーティションの作成とインストール

続いてWindowsやmacOSをインストールするためのパーティションを用意して、それぞれのOSをインストールします。

### Windowsの場合

以前の本記事では、WindowsのパーティションはWindowsのインストーラで作成していました。しかしその手順ではChromeOS Flex用のパーティションが破壊されるため、修復作業が必要でした。

最近になってWindowsのインストールに必要なパーティションを予め作成しておけばインストーラでパーティションを作ることが無くなり、結果としてパーティションの破壊を防ぎ修復作業も不要になることに気付きました。そこでWindows用のパーティションをインストーラに頼ることなく、この段階でsfdiskを使って作成します。

Windows 10や11をインストールするには、次の3つのパーティションが必要になります。

| 名前 | タイプを示すUUID | 内容 |
|--------|--------------------------------------|------------|
| 予約   | E3C9E316-0B5C-4DB8-817D-F92DF00215AE | マイクロソフトで予約されている。サイズは16MB   |
| データ | EBD0A0A2-B9E5-4433-87C0-68B6B72699C7 | Windowsのシステムとデータ用(Cドライブ)     |
| 回復   | DE94BBA4-06D1-4D40-A16A-BFD50179D6AC | Windowsの回復情報を保存する。サイズは1～2GB程度 |

パーティションの順番は任意ですが、ここでは予約パーティションをsda13、回復パーティションをsda14, データ用パーティションをsda15の順に割り当てて作成します。

まずuuidgenを使ってパーティション用のIDを3個用意します。

```
root@debian:/mnt# uuidgen > uuids
root@debian:/mnt# uuidgen >> uuids
root@debian:/mnt# uuidgen >> uuids
root@debian:/mnt# cat uuids
2ed439bd-96c9-48ee-a8f6-34e8xx4d3cf7
2614bf81-4be6-4aa8-adf5-4432xx13cb7a
c1ff36c2-0b82-4b33-9ee6-5bc5xxb2ffb2
root@debian:/mnt# 
```

次にp3-sda-dumpを元にp4-sda-dumpを作ります。

```command
root@debian:/mnt# cp p3-sda-dump p4-sda-dump
root@debian:/mnt# vi p4-sda-dump
```

p4-sda-dumpの修正はsda13、sda14、sda15の行の追加となりますが、ここで注意するのは各パーティションのスタートとサイズの値です。

sda13のスタートはsda12のstartにsizeを加えた値つまり 50827264 + 524288 = **51351552** となり、サイズは予約パーティションの場合16MBですから 16 \* 1024 \* 1024 / 512 = **32768** となります。これによって回復パーティションであるsda14のスタートが決まり 51351552 + 32768 = **51384320** で、サイズは1GBあれば通常は問題ないので 1024 \* 1024 \* 1024 / 512 = **2097152** を指定します。

残りのディスク領域をWindowsのシステムとデータ用のsda15に割り当てるわけで、スタートは 51384320 + 2097152 = **51351552** となりますが、サイズについては少し注意が必要です。というのは残りの領域がブロック数でいくつあるのかを確認する必要があるからです。いままでに何度か実行した`sfdisk --dump /dev/sda`の出力に次の行が含まれています。

```text
last-lba: 234441614
```

これはデータとして利用できる最終LBAを示しています。ですからsda15のサイズには、最大でlast-lbaの234441614からスタートの53481472を引いて1を加えた180960143ブロックを割り当てられます。ただ180960143は8の倍数ではないので、8の倍数で切り捨てて **180960136** を指定します。

なお、パーティションの名前やアトリビュートについては次の例と同じものにします(インストール済のWindowsから引用)。こうして各パーティションのstartとsizeが決まり、編集後のp4-sda-dumpの変更内容は次のようになります。

```command
root@debian:/mnt# diff -U1 p3-sda-dump p4-sda-dump
--- p3-sda-dump 2024-05-31 05:26:10.000000000 +0000
+++ p4-sda-dump 2024-05-31 08:31:26.000000000 +0000
@@ -20 +20,4 @@
 /dev/sda12 : start=    50827264, size=      524288, type=C12A7328-F81F-11D2-BA4B-00A0C93EC93B, uuid=F4CD81A7-771B-B54D-89B9-DCA60C886D4F, name="EFI-SYSTEM", attrs="LegacyBIOSBootable"
+/dev/sda13 : start=    51351552, size=       32768, type=E3C9E316-0B5C-4DB8-817D-F92DF00215AE, uuid=2ed439bd-96c9-48ee-a8f6-34e8xx4d3cf7, name="Microsoft reserved partition", attrs="GUID:63"
+/dev/sda14 : start=    51384320, size=     2097152, type=DE94BBA4-06D1-4D40-A16A-BFD50179D6AC, uuid=2614bf81-4be6-4aa8-adf5-4432xx13cb7a, attrs="RequiredPartition GUID:63"
+/dev/sda15 : start=    53481472, size=   180960136, type=EBD0A0A2-B9E5-4433-87C0-68B6B72699C7, uuid=c1ff36c2-0b82-4b33-9ee6-5bc5xxb2ffb2, name="Basic data partition"
root@debian:/mnt# 
```

p4-sda-dumpができたので、パーティションテーブルを更新します。

```command
root@debian:/mnt# sfdisk /dev/sda < p4-sda-dump
--- (中略) ---
root@debian:/mnt# sfdisk --list /dev/sda | tee p4-sda-list
Disk /dev/sda: 111.79 GiB, 120034123776 bytes, 234441648 sectors
Disk model: INTEL SSDSC2BW12
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 375C8B32-FD8D-464E-91A6-FB83A244B8C4

Device        Start       End   Sectors  Size Type
/dev/sda1  17272832  50827263  33554432   16G Linux filesystem
/dev/sda2        69     32836     32768   16M ChromeOS kernel
/dev/sda3   8884224  17272831   8388608    4G ChromeOS root fs
/dev/sda4     32837     65604     32768   16M ChromeOS kernel
/dev/sda5    495616   8884223   8388608    4G ChromeOS root fs
/dev/sda6        65        65         1  512B ChromeOS kernel
/dev/sda7        66        66         1  512B ChromeOS root fs
/dev/sda8    331776    364543     32768   16M Linux filesystem
/dev/sda9        67        67         1  512B ChromeOS reserved
/dev/sda10       68        68         1  512B ChromeOS reserved
/dev/sda11       64        64         1  512B unknown
/dev/sda12 50827264  51351551    524288  256M EFI System
/dev/sda13 51351552  51384319     32768   16M Microsoft reserved
/dev/sda14 51384320  53481471   2097152    1G Windows recovery environment
/dev/sda15 53481472 234441607 180960136 86.3G Microsoft basic data

Partition table entries are not in disk order.
root@debian:/mnt# 
```

p3-sda-listの結果と比べると次のように3つのパーティションが増えていることがわかります。

```command
root@debian:/mnt# diff -U0 --ignore-space-change p3-sda-list p4-sda-list
--- p3-sda-list 2024-05-31 05:33:18.000000000 +0000
+++ p4-sda-list 2024-05-31 08:33:12.000000000 +0000
@@ -21,0 +22,3 @@
+/dev/sda13 51351552  51384319     32768   16M Microsoft reserved
+/dev/sda14 51384320  53481471   2097152    1G Windows recovery environment
+/dev/sda15 53481472 234441607 180960136 86.3G Microsoft basic data
root@debian:/mnt#
```

ここまでの作業が終了したら、Debian Liveをシャットダウンします。

```command
root@debian:/mnt# poweroff
```

シャットダウン後にDebian LiveのUSBメモリを抜いて電源を入れると、ChromeOS Flexの起動を確認できます。もし起動しない場合は、今までの手順を見直します。Debian Liveを使って修復するか、またはChromeOS Flexのインストールからやり直すことになるかもしれません。

なお、途中で作成した**p3-sda-dumpは、ChromeOS Flexのパーティションが壊れた時の修復に必要となるので必ず保管してください**。

ChromeOS Flexの起動を確認できたら、次はWindowsのインストール作業になります。PCにWindowsのインストール用USBメモリを差してUSBメモリから起動しインストーラを実行します。インストール時の注意は、Windowsをインストールするパーティションを事前に用意したsda15を指定してインストールすることです。それさえ間違えなければ、あとは通常のWindowsのインストールとなんら変わりはなく、インストールが完了すればWindowsの初期設定画面が現れるはずです。

### macOSの場合

MacにmacOSをインストールする場合、インストール時にディスクユーティリティを使ってパーティションのフォーマットが必要になります。フォーマットを行うとChromeOS Flexのパーティション情報が破壊されるため、必ず**macOSインストール後にパーティションの修復が必要になります**。

まずmacOS用のパーティションをsda13に作成します。sda13はAPFSのGUIDである`7C3457EF-0000-11AA-AA11-00306543ECAC`を使います。sda13のスタートはsda12のstartにsizeを加えた値 50827264 + 524288 = **51351552** となります。

サイズはディスクユーティリティでフォーマットする際に利用可能な最大サイズに広げられるため空き容量をギリギリまで割り当てる必要は無く、適当な値(例えば50GB)を割り当てれば問題ありません。nameは`"Customer"`とします。

```command
root@debian:/mnt# uuidgen
cea1b7c3-961a-439b-83c4-42d4xxf092e3
root@debian:/mnt# cp p3-sda-dump p4-sda-dump
root@debian:/mnt# vi p4-sda-dump
```

p4-sda-dumpにはsda13の行を追加します。p3-sda-dumpからの変更点は次のようになります。

```command
root@debian:/mnt# diff -U1 p3-sda-dump p4-sda-dump
--- p3-sda-dump 2024-06-03 20:13:03.000000000 +0900
+++ p4-sda-dump 2024-06-04 06:07:02.000000000 +0900
@@ -20 +20,2 @@
 /dev/sda12 : start=    50827264, size=      524288, type=C12A7328-F81F-11D2-BA4B-00A0C93EC93B, uuid=4B628DE1-FE66-7840-8C8C-74B0ADA8FAB3, name="EFI-SYSTEM", attrs="LegacyBIOSBootable"
+/dev/sda13 : start=    51351552, size=   104857600, type=7C3457EF-0000-11AA-AA11-00306543ECAC, uuid=cea1b7c3-961a-439b-83c4-42d4xxf092e3, name="Customer"
root@debian:/mnt# 
```

p4-sda-dumpの修正が確認できたら、パーティションテーブルを更新します。

```command
root@debian:/mnt# sfdisk /dev/sda < p4-sda-dump
--- (中略) ---
root@debian:/mnt# sfdisk --list /dev/sda | tee p4-sda-list
Disk /dev/sda: 113 GiB, 121332826112 bytes, 236978176 sectors
Disk model: APPLE SSD TS128C
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 889D680A-FE77-D946-A383-8506969667B3

Device        Start       End   Sectors  Size Type
/dev/sda1  17272832  50827263  33554432   16G Linux filesystem
/dev/sda2  17141760  17272831    131072   64M ChromeOS kernel
/dev/sda3   8884224  17141759   8257536  3.9G ChromeOS root fs
/dev/sda4   8753152   8884223    131072   64M ChromeOS kernel
/dev/sda5    495616   8753151   8257536  3.9G ChromeOS root fs
/dev/sda6        65        65         1  512B ChromeOS kernel
/dev/sda7        66        66         1  512B ChromeOS root fs
/dev/sda8    331776    364543     32768   16M Linux filesystem
/dev/sda9        67        67         1  512B ChromeOS reserved
/dev/sda10       68        68         1  512B ChromeOS reserved
/dev/sda11       64        64         1  512B unknown
/dev/sda12 50827264  51351551    524288  256M EFI System
/dev/sda13 51351552 156209151 104857600   50G Apple APFS

Partition table entries are not in disk order.
root@debian:/mnt#
```

これでmacOSをインストールする準備ができたのでシャットダウンしてmacOSをインストールします。Debian LiveのUSBメモリとパーティションテーブルのメモをとった記録用のUSBメモリは後の作業で使用するのでそのまま保管してください。

```command
root@debian:/mnt# poweroff
```

USBメモリを抜いてMacの電源を入れると、ChromeOS Flexの起動を確認できます。

続いてmacOSのインストールを行います。macOSのインストーラのUSBメモリを使ってMacを起動します。macOSのインストーラが起動したら、以後の手順は次のようになります。

macOSのインストーラが起動したら、ディスクユーティリティを使ってmacOS用のパーティションをAPFSでフォーマットします。

1. 「APFS物理ストアdiskXXX」を選択して「パーティション作成」を選択
2. 50GB程度の「未使用のパーティション」を選択
3. パーティション情報の名前は「Macintosh HD」を指定
4. フォーマットは「APFS」を選択
5. 適用

これでmacOSインストール用のパーティションができるので、ディスクユーティリティを終了してmacOSのインストールを実行し、「Macintosh HD」のパーティションにインストールします。

インストールが終了したらmacOSが動くことを確認します。この時点では**macOSは起動するはずですが、ChromeOS Flexは起動できなくなっています**。

実際にChromeOS Flexを立ち上げると(起動方法は後述)次のように「Your system is repairing itself. Please wait.」と表示されバーインジケーターなども変化して、待っていれば修復できそうに見えますが実際にはいつまでたっても修復されません。

![chromeos-repairing.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/130638/1991b2ad-de73-b829-8031-ec5302d0d2a2.jpeg)

前述のように、macOSのディスクユーティリティがインストールのためにパーティションをフォーマットするとパーティションテーブルを書き換えてしまうためで、ChromeOS Flexでは修復できない状態になっています。

次にパーティションテーブルの修復方法を説明します。

## ChromeOS FLex用パーティションテーブルの修復

本修復作業はmacOSでは必須ですが、Windowsではインストール直後にパーティションテーブルの修復は必要ありません。しかしWindowsでも何かのきっかけでChromeOS Flex用のパーティションテーブルが書き換わり、修復作業が必要になることがあります。

修復もDebian Liveを使うのでDebian LiveのUSBメモリで起動し、前の作業で記録をとったUSBメモリを/mntにマウントします。なおaptコマンドを使うことはありません。

```command
user@debian:~$ sudo -i
root@debian:~# mount /dev/sdc1 /mnt
root@debian:~# cd /mnt
root@debian:/mnt# 
```

起動しなくなったパーティションテーブルの状況を`sfdisk --list`で確認します。

```command
root@debian:/mnt# sfdisk --list /dev/sda | tee p5-sda-list
Disk /dev/sda: 113 GiB, 121332826112 bytes, 236978176 sectors
Disk model: APPLE SSD TS128C
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 889D680A-FE77-D946-A383-8506969667B3

Device        Start       End   Sectors  Size Type
/dev/sda1        64        64         1  512B unknown
/dev/sda2        65        65         1  512B ChromeOS kernel
/dev/sda3        66        66         1  512B ChromeOS root fs
/dev/sda4        67        67         1  512B ChromeOS reserved
/dev/sda5        68        68         1  512B ChromeOS reserved
/dev/sda6    331776    364543     32768   16M Linux filesystem
/dev/sda7    495616   8753151   8257536  3.9G ChromeOS root fs
/dev/sda8   8753152   8884223    131072   64M ChromeOS kernel
/dev/sda9   8884224  17141759   8257536  3.9G ChromeOS root fs
/dev/sda10 17141760  17272831    131072   64M ChromeOS kernel
/dev/sda11 17272832  50827263  33554432   16G Linux filesystem
/dev/sda12 50827264  51351551    524288  256M EFI System
/dev/sda13 51351552 236715991 185364440 88.4G Apple APFS
```

ChromeOS Flexインストール時点の1から12を見ると(p3-sda-list参照)、sda12は事前にパーティションインデックスと物理的順番を一致するように対処してあったので変化はありません。しかしsda1からsda11までは、ストレージ上の物理的順番にパーティションテーブルが並んでしまっているのがわかります。macOSのディスクユーティリティが、パーティションの作成でフォーマットしたときに、物理的順番に並べ替えてしまうようです。このままではChromeOS Flexが起動できないので、パーティションテーブルの並び順を元の順番に修正します。

まず`sfdisk --dump`で現状のパーティションテーブルをp5-sda-dumpに保存します。

```command
root@debian:/mnt# sfdisk --dump /dev/sda > p5-sda-dump
```

p5-sda-dumpを前に保存したp3-sda-dumpと比べると、順番は変わっているもののパーティションの大きさやtype, uuidの情報は変わっていないことがわかります。sda13以降のパーティションの情報をp3-sda-dumpに追加してストレージに反映すれば、ChromeOS Flexのパーティションテーブルを修復できます。

修復するためのパーティションテーブルは、エディタを使うこともなく次のコマンドラインで作成できます。

```command
root@debian:/mnt# cp p3-sda-dump p6-sda-dump
root@debian:/mnt# tail -n 1 p5-sda-dump >> p6-sda-dump
```

2つめの`tail -n`のパラメータは、増えたパーティション数に合わせて1, 2, 3等を指定します。p6-sda-dumpの修正ができたら元になったp3-sda-dumpとdiffコマンドで比較して、正しく変更できたかどうかを確認します。

```command
root@debian:/mnt# diff -U1 p3-sda-dump.txt p6-sda-dump.txt
--- p3-sda-dump 2024-06-03 20:13:03.321219000 +0900
+++ p6-sda-dump 2024-06-04 06:32:12.000000000 +0900
@@ -20 +20,2 @@
 /dev/sda12 : start=    50827264, size=      524288, type=C12A7328-F81F-11D2-BA4B-00A0C93EC93B, uuid=4B628DE1-FE66-7840-8C8C-74B0ADA8FAB3, name="EFI-SYSTEM", attrs="LegacyBIOSBootable"
+/dev/sda13 : start=    51351552, size=   185364440, type=7C3457EF-0000-11AA-AA11-00306543ECAC, uuid=D6298900-687F-414B-8BF4-6FF6CFC39642, name="Customer"
root@debian:/mnt# 
```

問題なければ修正したパーティションテーブルをストレージに反映します。

```command
root@debian:/mnt# sfdisk /dev/sda < p6-sda-dump
```

これでChromeOS Flexと他OSのマルチブート環境の設定は完了となります。繰り返しになりますが将来WindowsやmacOSでのパーティションの変更でChromeOS Flexのパーティションが壊れた時に備えて途中で用意したp3-sda-dumpを必ず保管しておいてください。

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

```command
Windows Registry Editor Version 5.00

[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\TimeZoneInformation]
"RealTimeIsUniversal"=dword:00000001
```

レジストリの修正はリスクを伴いますので、くれぐれも慎重に操作してください。

## マルチブート環境構築後のパーティションテーブルの管理と修復

当初の記事執筆時(2023年3月頃)にインストールしたWindowsはWindows 10で、インストール後にsda13(予約)とsda14(Windowsデータ)の2つのパーティションが作られていました。後日Windows 11にアップグレードしたところsda14のサイズが縮小され、新たにsda15として回復パーティションが追加されました。この時ストレージ全体のパーティションテーブルが書き換えられてChromeOS Flexが起動できなくなり、修復のため改めてDebian Liveを使って作業中に保存してあったp3-sda-dumpを元にしてパーティションの修復を行う羽目になりました。

このようにWindowsやmacOSによってパーティションに変更が行われると、ほぼ間違い無くChromeOS Flexのパーティションテーブルは破壊されるので十分注意する必要があります。

また、ChromeOS FlexではABアップデート(更新時に2つのパーティションを切り替えて交互に利用する方式)を行うため、アップデート毎にパーティションテーブルのアトリビュートが書き換えられます。書き換えといってもアトリビュートの入れ替えだけなので、最新では無いバックアップのパーティションテーブルを戻すと一つ古いOSになるだけだと思いますが、できれば定期的に最新のパーティションテーブルを保存するようにするのが望ましいです。同じPCに3つ目のOSとしてDebian等のLinux環境をインストールしておくと、最新のパーティションテーブルのバックアップといざという時の修復が簡単になります。

## おわりに

実際にChromeOS Flexを使ってみると、同じPCでWindowsやmacOSに比べ余計なアプリが動かないこともあってか動作は極めて軽く、起動や終了の速さも特筆ものでした。アプリはWebブラウザ内で利用できるものだけになるのですが、今時はWordやExcel、PowerPoint、さらにはオンラインミーティング等もブラウザだけで扱えます。

WindowsやmacOSが快適に使えるPCでわざわざChromeOS Flexを使うメリットはあまり無さそうですが、CPUはそこそこなのにメモリが少ない場合(4GB程度)等にChromeOS Flexを選ぶのは最適な選択肢かもしれません。
