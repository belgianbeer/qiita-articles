# 最初にまとめ

長くなってしまったのでまとめを最初に。

- sudoを使うのをやめdoasに乗り替えた
- doasには必要十分な機能があり、sudoに比べ設定がシンプル
- doasは元々がOpenBSDプロジェクトのものであり、セキュリティ面でも信頼できる
- FreeBSDではpkgでインストール
- Linuxではソースからインストール (記事ではDebian/Ubuntu, CentOSでの例を紹介)
- doasのソース規模はsudoのそれに比べかなり小さい

最近manやmakeを知らない人が居るという事実を見かけるので、その辺りの利用例も含めて少々回りくどく記述しています。

> <small>本記事はSoftware Design誌 2020年1月号 に掲載された「sudoからdoasへ」の内容とほぼ同じもの、というより元の原稿そのものです。ネタを明かすと、元々Qiitaで公開するつもりで書いたのですが、書きすぎてボリュームが雑誌の記事になるぐらいになってしまったのをSoftware Designに拾っていただいたというのが本当の話です。</small>

# sudoコマンド
sudoコマンドはUNIX系オペレーティングシステムでは、システム管理に必須と言えるほど広く普及しているコマンドです。

元々UNIXにはsuコマンドが標準で用意されていて、各種BSDやLinuxの各ディストリビューションにもあります。suはユーザーが一時的にroot権限を得るために必要なコマンドで、実行するとroot環境のシェルが起動しますが、特権を得るためにはrootのパスワードが必要です。rootではない任意のユーザー権限を得ることもできますが、その権限を得ようとするユーザーのパスワードが必要になります。つまりsuではこれから利用する権限のユーザーのパスワードが必要になるわけです。

サーバーが1台で管理者がそのままユーザーであるような場合、特権を利用するときはsuでも問題はありません。ここでサーバーが複数あってそれぞれのrootのパスワードが共通の場合に、特定のユーザーに特定のサーバーだけの特権を与える必要がある場合を考えてみましょう。suを使うためには、rootのパスワードが全てのサーバーで共通のままではそのユーザーが別のサーバーでも特権を得られてしまいますから、当該サーバーのみrootのパスワードを別にする等の対応が必要になります。このようなことを複数のサーバーで設定すると、結局rootのパスワードをそれぞれ変えるとか、root以外の特権ユーザーを用意するなどの対応が必要となってしまいます。

sudoでは、特権を得るために必要なパスワードにユーザー自身のパスワードを使うことで前述の問題を解決しています。もちろん誰もが自分のパスワードで特権が得られては問題ですから、設定ファイルでユーザーを限定し、さらに特権を使って利用できるコマンドの限定等もできるようになっています。また設定ファイルを編集するためのvisudoという専用のコマンドも付属しています。

筆者が個人的にsudoが便利だと思っているのは次の3点です。

- ユーザー自身のパスワードで特権を得られ、特権時に使えるコマンドを限定できる。必要に応じてパスワード無しで特権を得る設定にもできる。
- sudoに続いてコマンド名を指定することで、そのコマンドのみ特権で実行できるので1コマンドのみ特権が必要な場合等に便利。
- 一度sudoでパスワードを入力すると、ある一定時間内にsudoを使う場合はパスワードの入力を省略でき、連続して特権で作業する場合の利便性が高い。

2番めの項目について少し補足すると、suでも引数を指定することで特権を使った1コマンドだけの実行はできますが、パラメータの指定が煩雑で使い勝手はsudoに軍配が上がります。

# sudoの脆弱性問題
sudoのように特権を扱うコマンドではどうしても脆弱性問題がつきまといがちです。つい最近にも(2019年10月中旬) sudo の脆弱性が話題になりました。この脆弱性は、sudoが利用できる設定になっているユーザーであれば設定の制限を超えた特権を得られてしまうというものでした。

sudoはすでに30年ぐらい歴史のある古いコマンドで、筆者が利用するようになったのは90年代前半頃からですが、以来時折sudoの脆弱性の報告を見ています。sudoは管理機能が豊富で便利な設定もできますが、あれば確かに便利でも実際はほとんど使われないような機能が多数用意されています。設定マニュアルを読んで全機能を理解するのはとても大変で、個人的には「こんなに機能要らないんだが…」と思いつつ使っているのが現状です[^1]。

[^1]: 正直筆者もsudoの全機能は理解できていません。

今回のsudoの脆弱性の報告を見て「またsudoに脆弱性か」と思ったわけですが、そこで最近のOpenBSDには標準コマンドとしてdoasが用意されていることを思い出しました。筆者の使い方であればsudoまでの機能は全く不要なので、この機会にとdoasを導入してみました。

# doasコマンド
doasもユーザー自身のパスワードを使って特権を得る点は、sudoと同様です。違いはと言えばsudoに比べて非常に機能が限定されていて、このぐらいの機能は必要だと思われる最小限の機能しかないことです。例えばsudoにはコマンドやユーザーのエイリアス等を活用して、複数のユーザーやコマンドをグルーピングして管理できますが、doasにはそういった管理に便利な機能は全くありません。しかしざっくり見たところ、筆者の利用範囲ではdoasでも困ることは無さそうですし、多くの利用範囲はこれだけあれば十分だろうと思います。

前述の通りdoasはOpenBSDのプロジェクトから生まれたコマンドの一つです。OpenBSDはセキュリティ面に非常に注力したオペレーティングシステムで、今や当たり前のように広く使われているSSHの代表的な実装のOpenSSHもOpenBSDプロジェクトからの産物です。そのような点でOpenBSDから生まれたdoasは、セキュリティ面でも信頼できると言えます。[^2]。

[^2]: sudoを信頼していないわけでは無いので誤解なきよう

# doasのインストール

## FreeBSD編
筆者は普段メインの環境としてFreeBSDを利用しているので、doasならすでにportsにあるだろうと思い調べてみると、予想通りsecurity/doasにありpkgも存在していました。そこでdoasのpkgをインストールします。

```
$ sudo pkg install doas
Updating FreeBSD repository catalogue...
----- (中略) -----
New packages to be INSTALLED:
	doas: 6.2

Number of packages to be installed: 1

Proceed with this action? [y/N]: y
[1/1] Installing doas-6.2...
[1/1] Extracting doas-6.2: 100%
=====
Message from doas-6.2:

--
To use doas,

/usr/local/etc/doas.conf

must be created. Refer to doas.conf(5) for further details.
----- (以下略) -----
```

## doas.confの設定
インストールが終わったら、次はdoasの設定です。インストール時のメッセージからわかるように、設定は /usr/local/etc/doas.conf に記述します。パッケージではdoas.confの編集の元になるファイル等は用意されないためゼロから設定する必要がありますが、難しくはありません。実は筆者がdoasを設定するのはこれが初めてでは無く、過去にOpenBSDで設定して利用した経験があります。しかしその時は、doasの最低限の設定を1行書いただけでした。

何はともあれ新規のコマンドを設定する場合に重要なのは、マニュアルの確認です。manコマンドを使ってdoas.confの設定方法を確認します。

```
$ man doas.conf
DOAS.CONF(5)              FreeBSD File Formats Manual             DOAS.CONF(5)

NAME
     doas.conf - doas configuration file

SYNOPSIS
     /usr/local/etc/doas.conf

DESCRIPTION
     The doas(1) utility executes commands as other users according to the
     rules in the doas.conf configuration file.

     The rules have the following format:

           permit|deny [options] identity [as target] [cmd command [args ...]]

     Rules consist of the following parts:

     permit|deny  The action to be taken if this rule matches.
----- (以下略) -----
```

この通りdoas.confでは、1行に1個のルールを記載し、後半に記載がありますが '\' はエスケープとして働くという、比較的よく見かける馴染みのある仕様です。manの後半には設定例がありますがそれを含めても拍子抜けするほど全体が短く、sudoの設定ファイルであるsudoersに比較して設定項目がとても少ないことがわかります。例えばoptionsのキーワードは nopass, keepenv, sentenv, persist の4種類しかありません。

通常各行は permit に続き必要なオプションを記入し、identityにはユーザー名やグループ名を設定します。またグループ名を指定する場合は :wheel のようにグループ名の先頭に : をつけます。前述の通りマニュアルには設定例も記載されているので、参考にすると良いでしょう。

doasではsudoにあるvisudoのような設定ファイル編集のための専用コマンドは用意されていないので、vi, vim, nano等使い慣れたエディタで/usr/local/etc/doas.confを用意します。以下の例では1行のdoas.confを用意しています。

```
$ sudo vi /usr/local/etc/doas.conf
----- (doas.confの編集) -----
$ ls -l /usr/local/etc/doas.conf
-rw-r--r--  1 root  wheel  64 Oct 21 18:11 /usr/local/etc/doas.conf
$ cat /usr/local/etc/doas.conf 
permit nopass minmin
$
```

この例のようにdoas.confのパーミッションは、rootがオーナーで644あるいは必要に応じて640や600に設定します。筆者は普段ユーザー名にminminを使っているのでここではそのまま例として示しています。

設定ではoptionsにnopassオプションを加えることでパスワード無しで任意のコマンドを特権で実行できるようにしています。 **nopassオプションによるパスワード無しの設定はリスクもありますので筆者がこれを推薦しているわけではありません**。あくまでもこれは筆者の個人サーバーの設定であり、他のサーバーではnopassオプションの有無を使い分けています。

最初のところで書いたように、sudoには一度sudoを実行すると繰り返し利用する場合にしばらくの間(デフォルトでは15分間)パスワード入力を省く機能があり、doasにも同様の機能を提供するpersistオプションが用意されているようですが、OpenBSDでしか利用できないと記載されてます。

```
    persist  After the user successfully authenticates, do not
             ask for a password again for some time. Works on
             OpenBSD only, persist is not available on Linux or
             FreeBSD.
```

sudoでは visudoが編集後にsudoresのシンタックスチェックを行い、誤りがあると警告して終了せずに再度編集できますが、doas.confでは普通のエディタを利用する事もありそのような編集時に設定ミスを確認することはできません。その代わりにdoas自身に-Cオプションで設定ファイルをチェックする機能があります。

```
$ doas -C /usr/local/etc/doas.conf
$
```

エラーがあればエラーメッセージを表示しますので、ここでdoas.confが問題無く記述できていることがわかります。

万が一doas.confに設定ミスがあるとdoasが実行できなくなるリスクもあるので、既存のdoas.confを編集するのであれば、次のような手順を行うほうが安全と言えます。

```
$ cp /usr/local/etc/doas.conf /tmp  ← 一旦 /tmp に doas.conf をコピーする
$ vi /tmp/doas.conf                 ← doas.conf の編集
<必要な編集を行う>
$ doas -C /tmp/doas.conf            ← 編集後の設定確認
$ doas install -m 644 /tmp/doas.conf /usr/local/etc  ← doas.confを更新
$
```

doasのオプションは `man doas` で調べられますが、簡単にリストアップしておきます。

-  -a： /etc/login.confに記載したdoasの認証方式を指定する
- -C ファイル名： 指定ファイルをdoasの設定ファイルとしてシンタックスをチェックする 
- -n：ノンインタラクティブな利用時に指定する(ssh 経由リモートサーバーでdoasを利用するときなど)。
- -s：ユーザーのシェルを起動する
- -u ユーザー名：指定したユーザーの権限を得る(デフォルトはroot)
- --：以降に指定したものを実行コマンドと引数として解釈する

doasが動作するかどうかを確認するには、idコマンドが最適です。

```
$ doas id
uid=0(root) gid=0(wheel) groups=0(wheel),5(operator)
$ 
```

この通りuidが0、つまりroot権限でidコマンドを実行でき特権が得られていることがわかります。

## doas利用時の環境変数

個人的に一番気になるのは、doasを利用した場合の環境変数PATHの扱いです。というのも例えばホームディレクトリの~/bin には個人的に利用するコマンドがあってシステム管理作業でも同じものを利用したい場合、PATHが変わってしまうとそれらのコマンドが単純には利用できなくなります。doasで特権に切り替わった時でも、普段設定してあるPATHを引き継ぎげれば都合が良いわけです。

まずはPATHがどうなるかをenvコマンドで確認してみます。

```
$ doas env | fgrep PATH
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
$
```

doasの環境下のデフォルトではこのようなPATHになり自分のPATHは引き継がれていないことがわかります。個人のPATHを引き継ぐ目的には、ぱっと見keepenvオプションを使えばよさそうだったので、設定してためしてみました。

```
$ cat /usr/local/etc/doas.conf 
permit nopass keepenv minmin
$ doas env | fgrep PATH
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
$ 
```

PATHに変化はありません。そこで改めて doas.confの keepenvのところをよく眺めてみます。

```
    keepenv  The user's environment is maintained.  The default
             is to reset the environment, except for the
             variables DISPLAY and TERM.
             ----- (中略) -----
             Note: The target user's PATH variable can be set at
             compile time by adjusting the GLOBAL_PATH variable
             in doas's Makefile. By default, the target user's
             path will be set to
             "/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:"
```

最後の Note: の節からわかるように、doasでの特権ユーザー時のPATHは、コンパイル時に指定したものが設定されるようになっています。

**一般論として、環境変数PATHをそのまま特権に引き継ぐのはセキュリティ面でリスクがあります**。ですからdoasを実行したときのデフォルトのPATHはこのようになっているわけですが、管理者が必要に応じて自分のPATHを引き継ぐのは問題ありません。確認したところ、ユーザーのPATHを引き継ぐためには明示的にsetenvオプションで指定すればよいことがわかりました。

```
$ cat /usr/local/etc/doas.conf 
permit nopass keepenv setenv { PATH } minmin
$ doas env | fgrep PATH
PATH=/home/minmin/bin:/usr/local/bin:/bin:/usr/bin:/usr/local/sbin:/sbin:/usr/sbin
$ 
```

結局筆者の個人サーバーのdoas.confは、次のように2行設定しました。

```
$ cat /usr/local/etc/doas.conf
permit nopass setenv { PATH } minmin
permit nopass keepenv root
$
```

rootユーザーであっても doasを利用する場合に何も設定が無いとdoasの実行を拒否されるため、2行目はそれを回避するための設定です。

なおmanの記述ではユーザーの HOME変数はkeepenvで引き継げるように読めるのですが、試したところどうもそうではなさそうで、HOME変数を引き継ぐのであれば setenvで明示的に指定する必要があります。筆者の場合はFreeBSDでデスクトップ環境を使うことは無いので問題になりませんが、デスクトップ環境を使う場合は、環境変数を適正に引き継がないとアプリケーションのクラッシュを招くことがありそうです。

## doasの実行記録
doasを実行すると、FreeBSDの場合は /var/log/auth.log に記録されます(sudoも同じ)。auth.log を見るのにも特権が必要となりますので、doasを使って確認します。

```
$ doas id
uid=0(root) gid=0(wheel) groups=0(wheel),5(operator)
$ doas tail -2 /var/log/auth.log
Oct 28 23:11:17 fbsdsvr doas: minmin ran command id as root from /home/minmin
Oct 28 23:11:37 fbsdsvr doas: minmin ran command tail -2 /var/log/auth.log as root from /home/minmin
$ 
```

なお tail に -2 と2行分だけに限定しているのは、本サンプルを作るために余計なものを表示させないようにしているだけであり、それ以上の意味はありません。

## Linux編

前項で引用したマニュアルのpersistのオプションのところにLinuxの文字列があることで、Linuxでもdoasが使えることに気づき、実際にLinuxにも導入してみました。一口にLinuxと言っても様々なディストリビューションがありますが、ここではDebian/Ubuntu、CentOSで試しています。

### doasのソースの入手

doasはsudoのように現時点で誰にでも知られているコマンドでは無いので、調べた範囲ではLinux用のパッケージはまったくありません。doasをインストールするためには、ソースを用意してコンパイルとインストールの作業を行う必要があります。

doas自体はOpenBSDの一部であるため、ソースはOpenBSDのソースツリーに含まれています。しかしOpenBSDのソースからdoasの部分だけ持ってきてもそのままでは不足するものがあり、他のOSでのコンパイルはすんなりとはいきません。そこでLinux等の他OSでもコンパイルできるようにソースレベルでパッケージ化したものがGithubで公開されています。

https://github.com/slicer69/doas/

まずはこのソースをgit cloneでローカルに展開します。

```
$ git clone https://github.com/slicer69/doas.git
----- (中略) -----
$ cd doas
$ ls -F
LICENSE   README.md  doas.1  doas.conf.5  env.c
Makefile  compat/    doas.c  doas.h       parse.y
$
```

ファイルの一覧からわかるようにdoasはCで記述されています。

### doasのコンパイルとインストール (Debian/Ubuntu編)

README.mdによると、Linuxでは `make` でコンパイル、`make install` でインストールできると書いてあり、難しいところは何も無さそうです。そこで早速コンパイルしてみましょう。以下はDebianで試していますが、Ubuntuでもまったく同様にできることを確認しています。

DebianでCのプログラムをコンパイルするためには、開発環境であるbuild-essentialのパッケージが必要なので、まずbuild-essentialをインストールします。

```
$ sudo apt update
----- (中略) -----
$ sudo apt -y install build-essential
----- (中略) -----
$ 
```

もちろんすでに開発環境を準備済であればこの作業は省略できます。

では実際にコンパイルしてみましょう。

```
$ make
cc -Wall -O2 -DUSE_PAM -DDOAS_CONF=\"/usr/local/etc/doas.conf\"  -D_GNU_SOURCE -include compat/compat.h -Icompat  -c -o doas.o doas.c
doas.c:48:31: fatal error: security/pam_appl.h: そのようなファイルやディレクトリはありません
 #include <security/pam_appl.h>
                               ^
compilation terminated.
<ビルトイン>: ターゲット 'doas.o' のレシピで失敗しました
make: *** [doas.o] エラー 1
$
```

ヘッダーファイルであるpam_appl.hが見つからないとエラーになりコンパイルが停止します。調べてみるとpam_appl.hは、PAMを利用するプログラムを作る際に必要なものです。実際README.mdには、libpam0g-devが必要であるという記載がありました(README.mdはよく読まないといけないですね ^^;)。

```
$ sudo apt -y install libpam0g-dev
```

気を取り直してあらためてコンパイルしてみます。

```
$ make
cc -Wall -O2 -DUSE_PAM -DDOAS_CONF=\"/usr/local/etc/doas.conf\"  -D_GNU_SOURCE -include compat/compat.h -Icompat  -c -o doas.o doas.c
cc -Wall -O2 -DUSE_PAM -DDOAS_CONF=\"/usr/local/etc/doas.conf\"  -D_GNU_SOURCE -include compat/compat.h -Icompat  -c -o env.o env.c
cc -Wall -O2 -DUSE_PAM -DDOAS_CONF=\"/usr/local/etc/doas.conf\"  -D_GNU_SOURCE -include compat/compat.h -Icompat  -c -o compat/execvpe.o compat/execvpe.c
compat/execvpe.c: In function ‘execvpe’:
compat/execvpe.c:61:5: warning: nonnull argument ‘name’ compared to NULL [-Wnonnull-compare]
  if ( (! name) || (*name == '\0') ){
     ^
cc -Wall -O2 -DUSE_PAM -DDOAS_CONF=\"/usr/local/etc/doas.conf\"  -D_GNU_SOURCE -include compat/compat.h -Icompat  -c -o compat/reallocarray.o compat/reallocarray.c
yacc parse.y
make: yacc: コマンドが見つかりませんでした
Makefile:54: ターゲット 'y.tab.o' のレシピで失敗しました
make: *** [y.tab.o] エラー 127
$
```

yaccコマンドが無いとエラーになってコンパイルが終了し、build-essentialにはyaccが含まれていないことに気づきました(そりゃそうか)。Debianでは数種類のyaccの実装からパッケージを選択できるようになっていますが、ここではBerkeley版のyaccであるbyaccをインストールして改めてコンパイルします。byaccの替わりにbisonをインストールしてもDebian/Ubuntuではそのままコンパイルできます。

```
$ sudo apt -y install byacc
----- (中略) -----
$ make
yacc parse.y
cc -include compat/compat.h -Icompat -Wall -O2 -DUSE_PAM -DDOAS_CONF=\"/usr/local/etc/doas.conf\"  -D_GNU_SOURCE -c y.tab.c
----- (中略) -----
cc -o doas doas.o env.o compat/execvpe.o compat/reallocarray.o y.tab.o compat/closefrom.o compat/errc.o compat/getprogname.o compat/setprogname.o compat/strlcat.o compat/strlcpy.o compat/strtonum.o compat/verrc.o -lpam -lpam_misc
$ 
```

コンパイルが無事終了したので、あとはインストールです。

```
$ sudo make install
mkdir -p /usr/local/bin
cp doas /usr/local/bin/
chmod 4755 /usr/local/bin/doas
mkdir -p /usr/local/man/man1
cp doas.1 /usr/local/man/man1/
mkdir -p /usr/local/man/man5
cp doas.conf.5 /usr/local/man/man5/
$
```

実行されるコマンドからわかるように、 `make install` で実際にインストールされるファイルは次の表の通りです。

| ファイル  | ファイルの内容 |
|:---------------------------------|:---------------------|
| /usr/local/bin/doas              | doasのプログラム本体 |
| /usr/local/man/man1/doas.1       | doasコマンドそのものmanファイル |
| /usr/local/man/man5/doas.conf.5  | doas.confのmanファイル |

この程度のファイル数であれば、パッケージになっていないプログラムでも管理は難しくありません。もし将来 doasをアンインストールする必要ができた場合は、これらの3つのファイルと、doasの設定ファイルである/usr/local/etc/doas.confを削除するだけです。

doasがインストールできたら次はdoas.confを用意しますが、ここではFreeBSDで設定したものと同じものを/usr/local/etcに用意しました。

Debianの場合もdoasの実行はFreeBSDと同様に/var/log/auth.logに記録されますが、実行するコマンドにオプションを指定する場合は、オプション解析ライブラリの関係で明示的にコマンドとの区切りを示す”--”を指定する必要があるようです。

```
$ doas id
uid=0(root) gid=0(root) groups=0(root)
$ doas tail -2 /var/log/auth.log
doas: invalid option -- '2'       ← ‘2’ が doas 自身のオプションと扱われてエラー
usage: doas [-ns] [-a style] [-C config] [-u user] command [args]
$ doas -- tail -2 /var/log/auth.log
Oct 30 23:50:25 idmdemo0 doas: minmin ran command id as root from /home/minmin
Oct 30 23:50:35 idmdemo0 doas: minmin ran command tail -2 /var/log/auth.log as root from /home/minmin
```

### doasのコンパイルとインストール (CentOS編)

CentOSでもdoasのコンパイルとインストールを試してみました。

PAMのヘッダーファイルやyaccについては準備しないとDebianと同じ問題が起きますので、Cのコンパイルに必要なgcc, makeと共にインストールしています。PAMの開発環境はpam-develのパッケージ、yaccについてはbyaccのパッケージを利用しています。

```
$ sudo yum install gcc make pam-devel byacc
----- (中略) -----
$ make
cc -Wall -O2 -DUSE_PAM -DDOAS_CONF=\"/usr/local/etc/doas.conf\"  -D_GNU_SOURCE -include compat/compat.h -Icompat  -c -o doas.o doas.c
----- (中略) -----
cc -o doas doas.o env.o compat/execvpe.o compat/reallocarray.o y.tab.o compat/closefrom.o compat/errc.o compat/getprogname.o compat/setprogname.o compat/strlcat.o compat/strlcpy.o compat/strtonum.o compat/verrc.o -lpam -lpam_misc
$ sudo make install
----- (中略) -----
$ 
```

インストールされるファイルについては、Debianの場合となんら差異はありません。

CentOSの場合、既にbisonがインストール済であればbyaccを新たにインストールする必要は無く、`make YACC=’bison -y’` としてmakeを起動すればコンパイルできます。

すでに紹介したOS同様doas.confを設定して、idコマンドでdoasの動作を確認します。またCentOSの場合doasの実行記録は、/var/log/secureになります。

```
$ doas id
uid=0(root) gid=0(root) groups=0(root) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
$ doas -- tail -2 /var/log/secure
Oct 30 23:56:56 centos0 doas: minmin ran command id as root from /home/minmin
Oct 30 23:57:15 centos0 doas: minmin ran command tail -2 /var/log/secure as root from /home/minmin
```

# doasとsudoでソースの規模を比較してみる
doasのソースをローカルに展開したので、この際と思いsudoとのソースの規模を単純に行数で比較してみました。

## sudo のソース規模
sudoはこの記事の作成時点で最新のsudo-1.8.28p1.tar.gzで数えてみます。

```
$ tar xf sudo-1.8.28p1.tar.gz
$ cd sudo-1.8.28p1
$ ls -F
ABOUT-NLS               config.h.in             ltmain.sh
ChangeLog               config.sub              m4/
INSTALL                 configure*              mkdep.pl*
INSTALL.configure       configure.ac            mkinstalldirs*
MANIFEST                doc/                    mkpkg*
Makefile.in             examples/               pathnames.h.in
NEWS                    include/                plugins/
README                  indent.pro              po/
README.LDAP             init.d/                 pp*
aclocal.m4              install-sh*             src/
autogen.sh*             lib/                    sudo.pp
config.guess            log2cl.pl*
$ 
```

sudoではsudoの本体の他に多数のライブラリのファイル等が含まれているので、srcディレクトリ下の、Cのソースファイル(*.c)とヘッダーファイル( *.h)を数えてみました。[^3]

[^3]: 実際にsudoのコア部分のソースがこの範囲だけなのかどうかまでは確認していません。

```
$ find src -type f -name '*.[ch]' | xargs wc -l
      76 src/preload.c
     735 src/exec_monitor.c
----- (中略) -----
     113 src/sudo_exec.h
     197 src/get_pty.c
   12612 total
$
```
12600行ほどあるのがわかります。ちなみにソースパッケージに含まれている *.[ch] に該当するファイルをすべて数えると10万行以上あり、正直とても驚きました。

## doas のソース規模
これに対してdoasは次の通り約1200行で、sudoに比較してとても小規模であることがわかります。

```
$ wc *.[chy]
     564    1716   13080 doas.c
      65     238    1673 doas.h
     230     693    4951 env.c
     343    1044    6831 parse.y
    1202    3691   26535 total
$
```

OpenBSDではないOSへ移植するために必要なライブラリ等が compat ディレクトリ下にありますが、これらをすべて数えてもdoasのソースパッケージは3000行程度です。

ソース規模が小さければ良いというわけでは無いですが、大きければそれだけ不具合を含むリスクが高まるのも事実です。

# doasのインストール後
筆者の場合sudoでは-uと-sオプションを使うことが多いのですが、doasでもこれらは同じオプションの指定で同じ機能を利用できます。つまり筆者の利用範囲ではdoasがあればsudoは不用なので、FreeBSD環境ではシステムからはsudoをアンインストールしました。

しかし自作のシェルスクリプト内部でsudoを利用しているものがあるため、書き換えるまでの当面の対応も含めてsudoはdoasへのシンボリックリンクに設定しました。

```
$ doas ln -s doas /usr/local/bin/sudo
$ ls -l /usr/local/bin/sudo
lrwxr-xr-x  1 root  wheel  4 Oct 22 10:46 /usr/local/bin/sudo -> doas
$ ls -lL /usr/local/bin/sudo
-rwsr-xr-x  1 root  wheel  28800 Oct  4 05:20 /usr/local/bin/sudo
$ sudo id             ← ここからのsudoの実体はdoas
uid=0(root) gid=0(wheel) groups=0(wheel),5(operator)
$ sudo -V             ← sudoは-Vでバージョンを表示するがdoasには無いオプション
doas: illegal option -- V
usage: doas [-ns] [-a style] [-C config] [-u user] command [args]
$ 
```

Linuxの場合、一部のLinuxのディストリビューションではsudoがあることを前提にシステムが構築されている場合もありますので、もしsudoをシステムから削除するのであれば、doasはsudoとは完全に同じでは無いことを良く理解した上で問題無いと判断できてからにしましょう。無理にsudoを削除する必要はありません。

# 最後に
sudoに脆弱性が発見されても、コマンドの性質上リモートから攻撃される危険性はありません。特に自分1人あるいは信頼できる人だけで利用しているサーバーであれば、慌てて対策を取る必要もありません。その意味でプライベートで利用しているサーバーでわざわざdoasに乗り替える必要性はあまり無いのかもしれませんが、個人的には同じ機能を利用するのであればシンプルなものを選ぶことにしていることもあり、今回doasを導入したのはは正解だったと考えています。
