# 最初にまとめ

長くなってしまったのでまとめを最初に。

- sudoを使うのをやめdoasに乗り替えた
- doasには必要十分な機能があり、sudoに比べ設定がシンプル
- doasは元々がOpenBSDプロジェクトのものであり、セキュリティ面でも信頼できる
- FreeBSDではpkgでインストール
- Linuxではソースからインストール (記事中にDebian/Ubuntu, CentOSの実例あり)
- doasのソース規模はsudoのそれの1/10程度と、とても小さい

# sudoコマンド

sudoコマンドはUNIX系オペレーティングシステムでは、システム管理に必須と言えるほど広く普及しているコマンドである。

元々UNIXには suコマンドが標準で用意されている(もちろん各種BSDやLinuxの各ディストリビューションにもある)。suはユーザーが一時的にroot権限を得るために必要なコマンドであり、実行するとroot環境のシェルが起動する。

しかしsuで特権を得るためには root自身のパスワードが必要であり、複数のシステムでrootのパスワードが共通な場合等に、特定のマシンだけで特定のユーザーに特権を与えるような場合には都合が悪い。

sudoはその点を改良し、ユーザー自身のパスワードで特権を得られ、さらに利用できるコマンドの種類等を制限できる管理面でとても良くできたコマンドと言える。

# sudoの脆弱性

sudoのようなコマンドではどうしても脆弱性問題がつきまといがちである。つい最近の話であるが、次の脆弱性が話題になった。

> [Linuxの「sudo」コマンドにroot権限奪取の脆弱性。ユーザーID処理のバグで制限無効化](https://japanese.engadget.com/2019/10/14/linux-sudo-root-id/) [^1]


[^1]: それにしてもこの見出しは意味不明。そもそもsudoはroot権限を得るものだろうとツッコミたい(笑)

sudoはすでに30年ぐらい歴史のある古いコマンドであり、今回に限らず過去にも脆弱性が複数報告されている。またsudoは管理機能が豊富で便利な設定もできるが、あれば確かに便利でも実際はほとんど使われないような機能が多数用意されていて、全機能を理解するのは大変である。個人的には「こんなに機能要らないんだが…」と思いつつ使っているのが現状である。

今回の報告を見て「またsudoの脆弱性か」と思ったわけだが、最近のOpenBSDには標準コマンドとしてdoasが用意されていることを思い出し、sudoの脆弱性と複雑な機能の問題から逃れるため doasを導入してみた。

# doasコマンド

doasもユーザー自身のパスワードを使って特権を得る点は、sudoと同様である。違いはと言えばsudoに比べて非常に機能が限定されていて、ほとんど必要最小限の機能しかないことである。

例えばsudoにあるコマンドやユーザーのエイリアス等の様々な機能を活用しているなら、そのままdoasに置き替えるのは容易では無いかもしれない。自分の場合はほぼパスワードレスで特権を利用するためだけにsudoを使っているので、doasに切り替えるのもなんら問題は無かった。

前述の通りdoasはOpenBSDのプロジェクトから生まれたコマンドの一つである。OpenBSDはセキュリティ面に非常に注力したオペレーティングシステムであり、今や当たり前のように使われているSSHの代表的な実装であるOpenSSHもOpenBSDプロジェクトからの産物である。そのような点でOpenBSDから生まれたdoasは、セキュリティ面でも信頼性は高い[^2]。

[^2]: sudoを信頼していないわけでは無いので誤解なきようお願いする

# doasのインストール

## FreeBSD編

普段FreeBSDをメインに利用しているので、doasならportsにあるだろうと思い調べると、予想通りsecurity/doasにありpkgも存在していた。そこでdoasのpkgをインストールする。

```
$ sudo pkg install doas
Updating FreeBSD repository catalogue...
<中略>
Checking integrity... done (0 conflicting)
The following 1 package(s) will be affected (of 0 checked):

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

Note: In order to be able to run most desktop (GUI) applications, the user
needs to have the keepenv keyword specified. If keepenv is not specified then
key elements, like the user's $HOME variable, will be reset and cause the GUI
application to crash.

Users who only need to run command line applications can usually get away
without keepenv.

When in doubt, try to avoid using keepenv as it is less secure to have
environment variables passed to privileged users.
```

## doas.confの設定

doasのインストールが終わったら、次は設定である。インストール時のメッセージからわかるように、doasの設定は /usr/local/etc/doas.conf で記述するが、doas.confのサンプル等はインストールされないためゼロから設定する必要がある。とは言えそれほど難しいものでは無い。実はdoasを設定するのはこれが初めてでは無く、過去にOpenBSDでは設定して利用した経験がある。

何はともあれ manコマンドで doas.conf の設定方法を確認する。

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
     <以下略>
```

この通り、1行1個のルールを記載し、後半に記載があるが '\' はエスケープとして働くというよくある仕様である。manで最後まで見ても拍子抜けするほど全体が短く、sudoの設定ファイルであるsudoersに比較してとても設定項目が少ないことがわかる。例えばoptionsのキーワードは nopass, keepenv, sentenv, persist の4種類しか無い。

通常は permit に続き必要なオプションを記入し、identityにはユーザーやグループを記述する。またグループは :wheel のようにグループ名の先頭に : をつけて指定する。

doasではsudoにあるvisudoのような設定ファイル編集のための専用コマンドは用意されていないので、通常のエディタで編集する。とりあえずは、次の行を設定し、挙動を確認してみる。

```
$ ls -l /usr/local/etc/doas.conf
-rw-r--r--  1 root  wheel  64 Oct 21 18:11 /usr/local/etc/doas.conf
$ cat /usr/local/etc/doas.conf 
permit nopass minmin
$
```

この例のようにdoas.confのパーミッションは、rootがオーナーで644あるいは必要に応じて640や600に設定する。設定ではoptionsにnopassを加えることでパスワード無しで任意のコマンドを特権で実行できるようにした。なお自分は普段アカウント名にminminを使っている。

sudoでは visudoが編集後にsudoresのシンタックスチェックを行ってくれるが、doasの場合は、doas自身に-Cオプションで設定ファイルをチェックできる。

```
$ doas -C /usr/local/etc/doas.conf
$
```

エラーがあれば表示するスタイルなので、doas.confが問題無く記述できていることがわかる。

doasが動作するかどうかを確認するには、idコマンドが最適である。

```
$ doas id
uid=0(root) gid=0(wheel) groups=0(wheel),5(operator)
$ 
```

この通りuidが0、つまりroot権限でidコマンドを実行でき特権が得られていることがわかる。

doasが問題無く動作することが分かったので、個人的に一番気になるのがdoasを利用した場合の環境変数のPATHの扱いの確認である。というのも ~/bin には個人的に利用するコマンドがあり、システム管理作業でも利用したいものがいくつかあるので、自分が普段利用しているPATHを引き継ぎたい[^3]。

[^3]: OpenBSDで試したときはPATHの扱いは確認していなかった

```
$ doas env | fgrep PATH
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
$
```

デフォルトではこのようなPATHになり自分のPATHは引き継がれていない。個人のPATHを引き継ぐ目的には、ぱっと見keepenvというオプションを使えばよさそうだったので、設定してためしてみた。

```
$ cat /usr/local/etc/doas.conf 
permit nopass keepenv minmin
$ doas env | fgrep PATH
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
$ 
```

PATHに変化はない。そこで改めて doas.confの keepenvのところをよく眺めてみる。

```
    keepenv  The user's environment is maintained.  The default
             is to reset the environment, except for the
             variables DISPLAY and TERM.

             Note: In order to be able to run most desktop (GUI)
             applications, the user needs to have the keepenv
             keyword specified. If keepenv is not specified then
             key elements, like the user's $HOME variable, will
             be reset and cause the GUI application to crash.
             Users who only need to run command line
             applications can usually get away without keepenv.
             When in doubt, try to avoid using keepenv as it is
             less secure to have environment variables passed to
             privileged users.

             Note: The target user's PATH variable can be set at
             compile time by adjusting the GLOBAL_PATH variable
             in doas's Makefile. By default, the target user's
             path will be set to
             "/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:"
```

最後の Note: のところにからわかるように、doasでの特権ユーザー時のPATHは、コンパイル時に指定したものが設定されるようになっている。一般論として、ユーザーが信用できないような場合には環境変数(特にPATH)をそのまま特権に引き継ぐのはセキュリティ面では好ましくない。とは言えシステム管理者が必要に応じて自分のPATHを引き継ぐのは問題ない。確認したところ、ユーザーのPATHを引き継ぐためには明示的にsetenvオプションで指定すればよい。

```
$ cat /usr/local/etc/doas.conf 
permit nopass keepenv setenv { PATH } minmin
$ doas env | fgrep PATH
PATH=/home/minmin/bin:/usr/local/bin:/bin:/usr/bin:/usr/local/sbin:/sbin:/usr/sbin
$ 
```

結局自分のサーバーのdoas.confは、keepenvを除いて次のように設定した。

```
$ cat /usr/local/etc/doas.conf
permit nopass setenv { PATH } minmin
permit nopass keepenv root
$
```

2行目の設定は、rootユーザーであっても doasを利用する場合に何も設定が無いと利用を拒否されるため、それを回避するための設定である。

なおmanの記述ではユーザーの HOME変数はkeepenvで引き継げるように読めるが、どうもそうではなさそうである。HOMEを引き継ぐのであれば、setenvで明示的に指定すればよい。

sudoには一度sudoを実行すると再度利用する場合にしばらくの間(デフォルトでは15分間)パスワード入力を省く機能があり、doasにも同様の機能を提供するpersistオプションが用意されているようであるが、OpenBSDでしか利用できないと記載されている。

```
    persist  After the user successfully authenticates, do not
             ask for a password again for some time. Works on
             OpenBSD only, persist is not available on Linux or
             FreeBSD.
```

## Linux編

上で引用した persist のオプションのところに Linux の文字列があることで、Linuxでもdoasが使えることに気づき、実際にLinuxにも導入してみた。Linuxと言ってもいろんなディストリビューションがあるので、ここでは Debian/Ubuntu、CentOS で試してみた。


### doasのソースの入手

doas自体はOpenBSDの一部であるため、ソースはOpenBSDのソースツリーに含まれている。しかし単純にdoasのソースだけ持ってきてもそのままでは他のOSでインストールするのは難しい。そこでLinux等の他OSでもコンパイルできるようにパッケージ化したものがGithubで公開されている。

https://github.com/slicer69/doas/

まずはこのソースをgit clone でローカルに展開する。

```
$ git clone https://github.com/slicer69/doas.git
<中略>
$ cd doas
```

### doasのコンパイルとインストール (Debian/Ubuntu編)

README.md によると、Linux では `make` でコンパイル、`make install` でインストールできると書いてあり、難しいところは何も無さそうである。そこで早速コンパイルしてみる。以下はDebian環境下での動作確認になるが、Ubuntuでも試したところまったく同様であった。

Debianでコンパイル環境を準備するためには、build-essential のパッケージをインストールする必要があるので、まず build-essentialのインストールから作業を始めている。

```
$ sudo apt update
<中略>
$ sudo apt install build-essential
<中略>
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

pam_appl.h が見つからないと言われてしまった。調べてみると pam_appl.hは、PAMを利用するプログラムを作る際に必要なヘッダファイルである。実際 README.md には、libpam0g-dev が必要であるという記載があった(README.md はよく読まないと ^^;)

```
$ sudo apt install -y libpam0g-dev
```

気を取り直してあらためてコンパイルしてみる。

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

yaccコマンドが無いと言われてコンパイルが終了し、 build-essential には yaccが含まれて居ないことに気づく(そりゃそうか)。Debianでは 数種類のyacc の実装からパッケージを選択できるようになっているようだが、ここでは bison をインストールし改めてコンパイルする。

```
$ sudo apt install -y bison
<中略>
$ make
yacc parse.y
cc -include compat/compat.h -Icompat -Wall -O2 -DUSE_PAM -DDOAS_CONF=\"/usr/local/etc/doas.conf\"  -D_GNU_SOURCE -c y.tab.c
<中略>
cc -o doas doas.o env.o compat/execvpe.o compat/reallocarray.o y.tab.o compat/closefrom.o compat/errc.o compat/getprogname.o compat/setprogname.o compat/strlcat.o compat/strlcpy.o compat/strtonum.o compat/verrc.o -lpam -lpam_misc
$ 
```

無事コンパイルできたので、あとはインストールする。

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

doasがインストールできたら、次はdoas.confを用意するわけだが、FreeBSDで設定したものと同じものを用意した。

```
$ doas id
uid=0(root) gid=0(root) groups=0(root)
$
```

この通り Debian でも doasの動作が確認できた。

### doasのコンパイルとインストール (CentOS編)

せっかくなので、CentOSでもコンパイルとインストールを試してみた。

PAMのヘッダーや yacc については Debian/Ubuntu と同じ問題が起きるので、あらかじめコンパイル環境の準備と共にインストールしておく。PAMの開発環境は pam-develのパッケージ、yacc については byacc のパッケージを利用した。

```
$ sudo yum install gcc make pam-devel byacc
<中略>
$ make
cc -Wall -O2 -DUSE_PAM -DDOAS_CONF=\"/usr/local/etc/doas.conf\"  -D_GNU_SOURCE -include compat/compat.h -Icompat  -c -o doas.o doas.c
<中略>
cc -o doas doas.o env.o compat/execvpe.o compat/reallocarray.o y.tab.o compat/closefrom.o compat/errc.o compat/getprogname.o compat/setprogname.o compat/strlcat.o compat/strlcpy.o compat/strtonum.o compat/verrc.o -lpam -lpam_misc
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

すでに紹介したOS同様 doas.conf を設定して、id コマンドで doas の動作を確認する。

```
$ doas id
uid=0(root) gid=0(root) groups=0(root) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
$ 
```

CentOSでもdoasは問題無く動作することが確認できた。

# doasとsudoでソースの規模を比較してみる

doasのソースをローカルに展開したので、この際と思い sudoとのソースの規模を単純に行数で比較してみた。

## sudo のソース規模
sudoはこの記事の作成時点で最新の sudo-1.8.28p1.tar.gz を展開したものである。

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

この通り、sudoでは多数のライブラリのファイル等が含まれているので、手っ取り早くsrcディレクトリ下の、*.c と *.h 等を数えてみた。[^4]

[^4]: 実際にsudoのコア部分のソースがこの範囲だけなのかどうかまでは確認していない。

```
$ find src -type f -name '*.[ch]' | xargs wc -l
      76 src/preload.c
     735 src/exec_monitor.c
<中略>
     113 src/sudo_exec.h
     197 src/get_pty.c
   12612 total
$
```
12600行ほどあるのがわかる。ちなみにソースパッケージに含まれている *.[ch] に該当するファイルをすべて数えると、10万行以上あり驚いた。

## doas のソース規模

これに対して doas は次の通り約1200行であり、sudo の 10分の1程度の規模である。

```
$ ls -F
LICENSE         README.md       doas.1          doas.conf.5     env.c
Makefile        compat/         doas.c          doas.h          parse.y
$ wc *.[chy]
     564    1716   13080 doas.c
      65     238    1673 doas.h
     230     693    4951 env.c
     343    1044    6831 parse.y
    1202    3691   26535 total
$
```

OpenBSDでは無いOSへ移植するために必要なライブラリ等が compat ディレクトリ下にあるが、これらをすべて数えても3000行程度である。

# doasのインストール後 (FreeBSD)

自分の場合 sudoでは -u と -s オプションを使うことが多いが、doasでもこれらは同じオプションで同じ機能を利用できる。こうなるともはやsudoには用が無いので、FreeBSD環境ではシステムからアンインストールした。そして自作のシェルスクリプト内部でsudoを利用しているため、書き換えるまでの当面の対応も含めてsudoはdoasへのシンボリックリンクとした。

```
$ doas ln -s doas /usr/local/bin/sudo
$ ls -l /usr/local/bin/sudo
lrwxr-xr-x  1 root  wheel  4 Oct 22 10:46 /usr/local/bin/sudo -> doas
$ ls -lL /usr/local/bin/sudo
-rwsr-xr-x  1 root  wheel  28800 Oct  4 05:20 /usr/local/bin/sudo
$ sudo id             ← ここからのsudoの実体はdoas
uid=0(root) gid=0(wheel) groups=0(wheel),5(operator)
$ sudo -V             ← -V オプションはdoasには無い
doas: illegal option -- V
usage: doas [-ns] [-a style] [-C config] [-u user] command [args]
$ 
```


# 最後に

sudoに脆弱性が発見されてもリモートから直接sudo経由で攻撃されることはないわけで、特に自分1人あるいは信頼できる人だけで利用しているシステムであれば、慌てて対策を取る必要は無い。その意味では個人で利用しているサーバーでわざわざdoasに乗り替える必要は無いのかもしれない。

個人的には同じ機能を利用するのであればシンプルな方を選ぶことにしているため、今回のdoasの導入に踏み切ったのは正解だと考えている。
