# はじめに
MacユーザーであればTime Machineでのバックアップが必須であることは言うまでもない。FreeBSDをZFSでファイルサーバーにしている場合、Time Machineバックアップをファイル共有をこのサーバーで利用できれば余計なコスト避けることができる。しかし実際にこれらを実現するには、Sambaをどう設定すればよいのか。

ちなみにNetatalkを使ってAFPサーバーを構築してもTime Machieサーバーは実現できるし実際それでずっと利用してきている。ただSambaと両方動かすのも微妙な手間なので、シンプルにするためにどちらかをやめるとなると、NetatalkはSMBをサポートしないのでAFPをサポートしているSambaにまとめることになる。

ここで目的とする設定は次の通りである。

- Time Machineバックアップが正しく動作する
- ファイル共有でMacのリソースフォークをZFSの拡張属性に保存する(“._*” が作成されない)

実はSambaでのTime Machineサーバーは、以前から試行錯誤をしていてどうしてもうまくいってなかった。先日改めてテストしたところ無事動作を確認できたので、その設定を紹介する。
# SambaでMacのファイル共有
SambaではVFSモジュールの追加で特定のファイルシステムやプロトコルに対応できるようになっている。AFP用のサービスのモジュールは vfs_fruit で、Macとファイル共有を行うためにはこのモジュールの導入が必要となる。

```
	vfs objects = fruit
```

# 動作した設定
最初に動作した設定例を示す。なおMacには関係のない一般的な設定は省略してある。
## global セクション
Mac用の特別な設定は記載しない
## 個別の共有フォルダ
共有名を share とする例

```
[share]
	path = /data/share
	vfs objects = fruit streams_xattr
	fruit:resource = stream
	fruit:metadata = stream
```

## Time Machine用のバックアップフォルダ
共有名を TimeMachine とする例

```
[TimeMachine]
	path = /data/TimeMachine
	vfs objects = fruit
	fruit:resource = xattr
	fruit:time machine = yes
```

この設定では、 /data/TimeMachine 内に Mac側がバックアップフォルダを作成する際、一緒に ._ で始まるファイルができてしまうが、Time Machine 専用フォルダでファイル共有には利用しないので許容する。
## ZFSの構成
上記の設定で、/data/share と /backup/TimeMachine は ZFSでは別々のファイルシステムとして設定している。つまり zfs list では次のように見える

```
$ zfs list -o name -r zroot/data
NAME
zroot/data
zroot/data/share
zroot/data/TimeMachine
```

このように独立したファイルシステムにすると、ZFSのプロパティでTimeMachineのディスク容量の制限が簡単にできる。もちろん制限はSamba側でも設定できるが、個人的にはZFSプロパティのほうが管理しやすい。

もし、/data/shre と /data/TimeMachine が ZFSとして同じファイルシステムでのディレクトリの場合は、本設定が動作するのかどうかについては確認していない。
# 動作しなかった設定
## ダメだった例1
```
[global]
	vfs objects = fruit streams_xattr
	fruit:resource = stream
	fruit:metadata = stream

[share]
	path = /data/share
	# Mac用の特別な設定は設定しない

[TimeMachine]
	path = /data/TimeMachine
	fruit:time machine = yes
```

共有は動作するが、TimeMachineのバックアップが動作しない
## ダメだった例2

```
[global]
	vfs objects = fruit 
	fruit:resource = xattr

[share]
	path = /data/share
	# Mac用の特別な設定は設定しない

[TimeMachine]
	path = /data/TimeMachine
	fruit:time machine = yes
```

TimeMacineバックアップは動作する。しかし共有フォルダをファインダーでアクセスすると挙動不審で、例えばjpegファイルをダブルクリックすると、ファインダーからはオブジェクトが消える。しかし実際にはファイルは残っている

## ダメだった例3

```
[global]
	vfs objects = fruit streams_xattr
	fruit:resource = stream

[share]
	path = /data/share
	# Mac用の特別な設定は設定しない

[TimeMachine]
	path = /data/TimeMachine
	fruit:time machine = yes
```

共有は ._* ファイルができる。バックアップは動作しない。

# まとめ

テストした範囲では、このように共有設定とTime Machine用設定では vfs_fruit のオプションを変えないと動作しなかった。過去にいろいろ試したのは、globalセクションにvfs_fruitのオプションを設定して、Time Machine用のみに「fruit:time machine = yes」を追加したのがいけなかったようである。

ここでの結論は、FreeBSDでZFSを使いSambaでMac用の設定を行う場合は、**Time Machine用と共有用では別々の設定を行う** 、ということになる。なおUSFやLinuxでのEXT4では試していない。

動作する設定は見つけられたものの、どうもSambaのドキュメントの説明と噛み合わず自分的には少々気持ち悪い。もしZFSとSambaの組み合わせで、この設定で動いているぜというのがあれば、ぜひコメントして欲しい。
