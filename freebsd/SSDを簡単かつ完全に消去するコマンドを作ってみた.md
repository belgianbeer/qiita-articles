<!-- https://qiita.com/belgianbeer/items/ecd070a649d859143e59 -->

# SSDを完全かつ簡単に消去するコマンドを作ってみた

## ストレージの消去コマンド

最近になってSSDやHDDのストレージには、書き込まれているデータを消去する制御コマンドがあることを知りました。FreeBSDやLinuxにはこの制御コマンドを発行するコマンド(FreeBSDではcamcontrol, Linuxではhdparm)が用意されています。書き込み済データを完全に消去できるのであれば、廃棄時に機密情報の漏洩を心配をする必要が無くなります。特にSSDでは、この制御コマンドによってセルの状態を工場出荷時にリセットし書き込み性能を回復できるメリットもあります。

消去そのものは制御コマンドを送るだけなのですが、そのためには事前に指定した手順でストレージ側を消せる状態にする必要がありOSのコマンドをそのまま使うだけでは微妙に手間です。そこでこれらの手順をまとめて、データ消去を簡単に実現するコマンド(シェルスクリプト)を作ってみました。FreeBSDとLinux(Debian)で動作確認しています。

プログラムは次のコマンド名のリンク(Github)にあります。

　 **[drive-secure-erase](https://github.com/belgianbeer/drive-secure-erase)**

ちなみにQiitaにも、Linuxでhdparmコマンドを使ってSSDを消去する解説記事は色々あります。

## 警告

本コマンドは大変危険なものです。SSDでの処理は数秒～数十秒で完了し、消去したデータを復活する手段はありません。誤って使用すると取り返しのつかない結果を招きます。

## Linuxでの事前準備

Linuxでhdparamがシステムにインストールされていない場合、まずhdparmをインストールする必要があります。DebianやUbuntuではaptコマンドを使います。

```code
$ apt install hdparm
```

FreeBSDでは特別な準備は必要ありません。

## 消去可能かどうかの判定

制御コマンドによる消去をセキュアイレースと呼びます。drive-secure-eraseでは実際に消去を行う前に、ドライブがセキュアイレースを実行できる状態かどうかを確認します。

対象のドライブはFreeBSDであれば"da1"、Linuxの場合は"/dev/sdb"のように指定します。絶対に対象ドライブを間違えないように注意してください。間違えて重要なドライブのデータを消去しても回復できません。そして、実行にはroot権限が必要です。

drive-secure-erase を実行するとドライブの状態を報告し、消去できるかどうかがわかります。

FreeBSDでの例

```code
$ drive-secure-erase da1
......
*** da1 is ready for secure erase.
$ 
```

Linuxでの例

```code
$ drive-secure-erase /dev/sdb
......
*** /dev/sdb is ready for secure erase.
$ 
```

ドライブがEnhanced Secure Eraseをサポートしている場合は、次のように表示します。

```code
$ drive-secure-erase da1
......
*** da1 is ready for enhanced secure erase.
$ 
```

## セキュアイレースの実行

消去できることがわかったら、引数にyesを追加して再び実行します。Enhanced Secure Erase可能ならそれを使い、そうでない場合は通常のSecure Eraseを実行します。数秒でドライブの消去が完了します。


```code
$ drive-secure-erase da1 yes
```

消去が完了しても、OS側は消去前のデータをキャッシュに残しているため、SSDをアクセスしても消去できていないように見えることがあるので注意が必要です。

## 注意

FreeBSDの場合次のようなエラーメッセージが表示されて終了することがありますが、正常に消去されます。

```
camcontrol: ATA SECURITY_ERASE_UNIT via pass_16 failed
```

drive-secure-eraseでは、NVMeストレージ[^nvme]とUSBメモリ[^usb]の消去はサポートしていません。

[^nvme]:NVMeにもSSD等と同様に制御コマンドがあるのですが、テスト環境を用意できないこともあって実装を見送っています。FreeBSDではnvmecontrolコマンドを、Linuxではnvme-cliに含まれているnvmeコマンドを直接利用すれば同様のことができます。検索してみたところ、Qiitaにもnvme-cliの使い方を書いた記事がいくつかあるようです。

[^usb]:USBメモリには消去するような機能は用意されていないようです。

くれぐれも指定するデバイス名を間違えないようにしてください。誤ってシステムドライブを指定した場合、多くは"fronzen"で消去できません。しかしシステム構成によっては消去できる場合があります。システムドライブの内容をいきなり消去した場合の被害は言うまでもありません。

## drive-secure-eraseをHDDで使用する

HDDでもdrive-secure-eraseは利用できます。しかしながらその処理にはドライブの容量と転送速度に応じた時間を必要とし、数時間から場合によっては1日以上かかることもあります。時間がかかるからといって途中で電源を切って中段しても、HDDの状態はロックされたままになり利用できなくなります。ロックされたドライブを解除するにはunlockコマンドを指定します。

```code
$ drive-secure-erase da1 unlock
```

## drive-secure-eraseは何をしているのか

drive-secure-eraseは内部で特別な処理を行っているわけではありません。FreeBSDであればcamcontrolのsecurityコマンド、Linuxではhdpermの--secure-eraseオプションを適切に使用することで本コマンドと同じことが行えます。最初に書いたようにこれらのコマンドを使うには、少しばかりの手続きが必要で単純には利用できません。コマンドを直接使う場合は次の手順となります。

1. セキュアイレース可能かどうかの確認
1. ユーザー名とパスワードを指定して、ドライブをセキュアモードに設定
1. 設定したユーザー名とパスワードを指定してドライブの消去

drive-secure-eraseでは、これらの手続きを自動で行うことで消去が簡単に実行できるようにしているだけです。

## ddコマンドで消去するのとの本質的な違い

ドライブを消去する場合ddコマンドを使って/dev/zeroを書く方法がよく使われますが、Secure Eraseとの違いは、前者がドライブに対してデータを送り続けなければならないのに対して、後者はドライブに対して制御コマンドを発行するだけ済む点です。またSSDのセルを初期化するようなことはddでデータを書き込む方法ではできず、その点でもSecure Eraseは大きな強味となります。

## Secure EraseとEnhanced Secure Eraseの違い

そのうち書くかもw
