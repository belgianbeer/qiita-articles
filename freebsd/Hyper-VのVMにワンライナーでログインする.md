# Hyper-VのVMにワンライナーでログインする

## はじめに

筆者はFreeBSDを日常的に使っていますが、デスクトップ環境はWindowsやmacOSを使っています。なのでFreeBSDを使う場合は、デスクトップ環境からssh等を使ってサーバーにログインするわけです。また筆者のWindowsノートPCには、手軽にFreeBSDを利用できるようにHyper-Vを使ったVM環境にFreeBSDを用意してあります。もちろんFreeBSDだけでなくLinux環境としてDebian/GNU Linuxも用意してあります。

## Hyper-VのVMを利用する

Hyper-VのVMを使う一つの方法は、Hyper-VマネージャーからVMに接続してコンソールからログインすることです。しかしこの方法ではクリップボードの利用に制限があって[^clip]扱いにくく、日本語の入力や表示もできません。そこでTera Term等のターミナルエミュレータのアプリケーションやssh.exe等で直接VMにログインすることになります。

[^clip]:コンソール表示のコピーは画像イメージになり、テキストだけのコピーできません。

## PowerShellからVMのIPアドレスを調べる方法

当たり前のことですがVMに直接ログインするには、VMのIPアドレスが必要です。しかしVMのネットワーク設定が「Default Switch」(通常はDefault Switchで、WindowsのIPアドレスを利用して外部への接続はNAT経由になる)の場合、Windowsの起動毎にVMのIPアドレスが変わるため、現在のIPアドレスを都度確認する必要があります。

VMのIPアドレスを知る方法としてはHyper-Vマネージャーを使う方法がありますが、アプリを起動してVMを選びネットワークを表示する手順が必要で少々煩雑です。

実はPowerShellには、VMのIPアドレスを調べるコマンドが用意されています。`Get-VMNetworkAdapter`でVM名を指定すればIPアドレスを表示できます。ただし`Get-VMNetworkAdapter`を実行するには管理者権限が必要なので、管理者権限でWindows Terminalを実行する必要があります[^admin]。

[^admin]:管理者権限のWindows Terminalを起動するキーボード操作は、[Win]+[X] を入力し[A]で選択するのが手っ取り早い。

```PowerShell
PowerShell 7.4.6
PS C:\Users\minmin> GET-VMnetworkAdapter -VMName minfreebsd

Name                    IsManagementOs VMName     SwitchName     MacAddress   Status IPAddresses
----                    -------------- ------     ----------     ----------   ------ -----------
ネットワーク アダプター False          minfreebsd Default Switch 00155D207102 {Ok}   {172.28.245.161, fe80::215:5dff:fe20:7102}

PS C:\Users\minmin>
```

見て明らかなように、IPAddressesでIPv4とIPv6の両方のアドレスを確認できます。

## PowerShellからワンライナーでVMにログインする

これをもう一歩進めて、ワンライナーでVMにログインしてみましょう。Select-Object を使えばIPAddressesだけを選択できます。

```PowerShell
PowerShell 7.4.6
PS C:\Users\minmin> Get-VMNetworkAdapter -VMName minfreebsd | Select-Object -ExpandProperty IPAddresses
172.28.245.161
fe80::215:5dff:fe20:7102
PS C:\Users\minmin>
```

さらにIPv4アドレスを抽出するためにWhere-Objectを使います[^string]。

[^string]:Where-Objectの代わりにSelect-Stringを使う方法もあります

```PowerShell
PowerShell 7.4.6
PS C:\Users\minmin> Get-VMNetworkAdapter -VMName minfreebsd | Select-Object -ExpandProperty IPAddresses | Where-Object { $_ -match "^\d{1,3}(\.\d{1,3}){3}$" }
172.28.245.161
PS C:\Users\minmin>
```

> なおIPv6アドレスであれば、Where-Objectのパラメータを`{ $_ -match ":" }`と変更して「:」(コロン)にマッチさせます。

IPv4アドレスを抽出できたので、あとはこれを () で囲んでssh.exeの引数にすれば、目的のVMにログインできます。

```PowerShell
PowerShell 7.4.6
PS C:\Users\minmin> ssh (Get-VMNetworkAdapter -VMName minfreebsd | Select-Object -ExpandProperty IPAddresses | Where-Object { $_ -match "^\d{1,3}(\.\d{1,3}){3}$" })
Last login: Thu Jan  9 07:19:52 2025 from mintest-cf-sx2.mshome.net
FreeBSD 14.2-RELEASE (GENERIC) releng/14.2-n269506-c8918d6c7412

Welcome to FreeBSD!

Release Notes, Errata: https://www.FreeBSD.org/releases/
<以下略>
```

PowerShellはコマンドラインヒストリーの編集が充実しているので、次からは [Ctrl]+[R] のリバースインクリメンタルサーチでVM名等を検索すれば再利用でき長いコマンドを入力する必要はありません。またsshまで入力すると直近のsshのコマンド行が補完できるので、同じVM名であればそのまま再利用できます。

(2025-03-14追記)
> このようにIPアドレスを調べて使わなくても、Hyper-Vで設定したVM名を使って`vmname.mshome.net`でアクセスできます。
