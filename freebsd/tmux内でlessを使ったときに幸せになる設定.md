# tmux内でlessを使ったときに幸せになれる設定

## はじめに

tmuxを使うようになってずっと不満だったのが、lessで「**q**」を入力して終了したときに画面が切り替わってしまい、lessの終了直前に見ていた部分が残らないことでした。特にmanコマンドで目的の項目をみつけたところで終了すると見ていた情報が消えて、とても不便でした。数年間我慢してtmuxを使っていたのですが、しびれを切らして設定があるはずだとtmuxのmanを調べました。

## 結論

最初に結論です。**~/.tmux.conf** に次の1行を追加すれば、tmux内のlessで終了時に見ていた部分の画面が残ります。

```text
set-option -g alternate-screen off
```

## 解説

多くのターミナルエミュレータにはtmuxのalternate-screen制御するような別の画面に切り替える機能があり[^tite]、lessは起動時に画面を切り換え、終了時に元に画面に戻します。元の画面に戻ることで直前に表示していた部分が消えるというわけです。画面が切り替わるのを抑止できれば、lessの終了時に最後に見ていたところを残すことができます。

[^tite]:端末の機能としてはカーソル移動機能利用のON/OFFということのようです。

### less -X

実はless自身にも、別画面への切り替えを抑止する**-X**オプションが用意されています。実際最初に試したのが-Xオプションでした。コマンドラインでlessを使うときに-Xオプションを追加すれば、問題無く動作するのは確認できました。

ちなみにlessの-Xオプションの説明には次のように記述があります。

> Disables  sending  the  termcap  initialization and deinitialization strings to the  terminal.  This is sometimes desirable if the deinitialization  string  does  something unnecessary, like clearing the screen.

Google翻訳による日本語

> 端末への termcap 初期化および非初期化文字列の送信を無効にします。非初期化文字列が画面をクリアするなど、不要な処理を実行する場合、これが望ましいことがあります。

そこで常時-Xオプションを有効にするため環境変数のLESSを指定したら、単純にファイル見る場合は問題なかったのですが、git diffで変更分の差分を見るときにエスケープシーケンスの処理がうまくいかず、正しく表示できなくなってしまいました。

### tmuxのalternate-screen

lessの-Xはあきらめて、tmuxにもalternate-screenの抑止機能があるはずなのでtmuxのmanから探すことにしました（正直tmuxのmanは巨大なので探すのがめんどくさかったわけです😅）。思いつくキーワードで検索したところ、alternate-screenはすぐに見つかり、tmux内で

```text
: set-option alternate-screen off
```

を実行したところ、期待した動作を確認できました。

設定する変数が判明したので、あとはいちいち指定しなくても済むよう ~/.tmux.conf に追加すれば完了です。

次の行

```text
set-option alternate-screen off
```

を追加してtmuxを起動したところエラーとなりました。

```text
/home/minmin/.tmux.conf:3: no current window
```

これは.tmux.confを読み込む時点ではカレントウィンドウが存在しないことによるエラーのようです。このエラーは、`set-option`に`-g`オプションを指定すれば回避できます。こうして最初に説明した設定となります。
