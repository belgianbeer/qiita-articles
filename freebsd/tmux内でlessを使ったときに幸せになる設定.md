# tmux内でlessを使ったときに幸せになれる設定

## はじめに

tmuxを使うようになって数年経ちます。しかしtmuxでずっと不満だったのが、lessで`q`を入力して終了したときに画面が切り替わり、終了直前に見えていた部分が残らないことでした。例えばmanコマンドなどで目的の項目をみつけたところで終了すると肝心の項目が消えて参照できなくなるので、極端な場合は別のターミナルウィンドウを開いて確認することもありました。長期にわたりこのままtmuxを使っていたのですが、ようやく思い立ってtmuxのmanを調べました。

## 結論

最初に結論です。**~/.tmux.conf** に次の1行を追加すれば、tmux内のlessで終了時に見ていた部分が画面に残ります。

```text
set-option -g alternate-screen off
```

## 解説

多くのターミナルエミュレータにはtmuxのalternate-screenで制御するような別の画面に切り替える機能があり[^tite]、lessは起動時に画面を切り換え、終了時に元の画面に戻します。元の画面に戻ることで直前に表示していた部分が消えるというわけです。画面が切り替わるのを抑止できれば、lessの終了時に最後に見ていたところを残すことができます。

[^tite]:端末制御としてはカーソル移動機能利用のON/OFFということのようです。

### lessの-Xオプション

実はless自身にも、別画面への切り替えを抑止する`-X`オプションが用意されています。実際最初に試したのが-Xオプションでした。コマンドラインでlessを使うときに-Xオプションを追加すれば、問題無く動作するのは確認できました。

ちなみにlessの-Xオプションの説明には次のように記述があります。

> Disables sending the termcap initialization and deinitialization strings to the terminal. This is sometimes desirable if the deinitialization string does something unnecessary, like clearing the screen.

> 端末への termcap 初期化および非初期化文字列の送信を無効にします。非初期化文字列が画面をクリアするなど、不要な処理を実行する場合、これが望ましいことがあります。
>（Google翻訳による日本語）

常時-Xオプションを有効にするため環境変数のLESSを指定したところ、単純にファイル見る場合は問題なく、manコマンド内で起動されるlessでもうまく働いてくれました。しかし`git diff`を実行した際、通常であれば変更の差分をカラーで表示するところを、カラーのためのエスケープシーケンスの処理がうまくいかず正しく表示できなくなってしまいました。それで-Xオプションはあきらめました[^color]。

[^color]:lessに関しては-Xオプションの存在は以前から知っていたこともあってそれ以上調べませんでした。その後調べたところ、-Rオプションを一緒に指定すれば希望の動作となることがわかりました。

### tmuxのalternate-screen

自分では以前に、screenコマンドで別画面への切り替えの抑止を設定したことがありました(最近確認したらその設定は不要になっていました)。それでtmuxにも別画面への切り替えを抑止する設定があるはずだと確信して、tmuxのmanから探すことにしました。正直tmuxのmanは巨大なので探すのがめんどくさかったわけです😅。

思いつくキーワードでmanを検索したところ、すぐにalternate-screenオプションが見つかりました。tmux内のコマンド入力で

```text
: set-option alternate-screen off
```

を実行したところ、期待した動作を確認できました。

設定するオプションが判明したので、あとはいちいち指定しなくても済むよう ~/.tmux.conf に追加すれば完了です。

しかし次の行

```text
set-option alternate-screen off
```

を追加してtmuxを起動したところエラーとなりました。

```text
/home/minmin/.tmux.conf:3: no current window
```

`no current window`は`.tmux.conf`を読み込む時点ではカレントウィンドウが存在しないことによるエラーのようです。このエラーは、`set-option`に`-g`オプションを指定すれば回避できます[^tsuika]。こうして最初に説明した設定を行えば完了です。

[^tsuika]:実は当初`set-option`コマンドの`-g`オプションに気付かず、他の設定方法を調べ `set-hook -g window-linked 'set-option alternate-screen off'` として利用していました。
