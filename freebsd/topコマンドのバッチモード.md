<!-- https://qiita.com/belgianbeer/items/f5b0461698ad9d3a3c02 -->

# topコマンドのバッチモード

## はじめに

topコマンドは、LinuxやMac等ほとんどのUNIX系OSに用意されているコマンドの一つです。topを起動すると、1秒、あるいは数秒間隔でプロセスや負荷の状況を表示して、システムの状態の変化を確認するために使われます。知ってる人は知ってる話ですが、topにはバッチモードが用意されています。ということでtopのバッチモードの紹介です。

## FreeBSDのtopとバッチモード

topコマンドは、負荷の高いプロセスを調べられるだけでなく、ps、uptime、swapinfo、vmstat等で個別に調べる内容をまとめて見ることができる点が便利です。さらにZFSを使っていればARCのメモリの使用状況まで確認できます。

topのオプションに-bを指定するとバッチモードとなり、通常のtopの1回分を表示して終了します。システムの負荷の状態のサマリを1コマンドで得られるのはかなり便利ではないでしょうか。

```console
$ top -b
last pid: 46319;  load averages:  0.26,  0.16,  0.12  up 12+18:17:21    07:34:08
41 processes:  3 running, 38 sleeping
CPU:  0.1% user,  0.0% nice,  1.6% system,  0.0% interrupt, 98.3% idle
Mem: 1439M Active, 751M Inact, 3855M Wired, 40K Buf, 1819M Free
ARC: 3201M Total, 584M MFU, 2368M MRU, 896K Anon, 161M Header, 64M Other
     2825M Compressed, 4857M Uncompressed, 1.72:1 Ratio
Swap: 8192M Total, 8192M Free

  PID USERNAME    THR PRI NICE   SIZE    RES STATE    C   TIME    WCPU COMMAND
 1432 root         13  20    0  2135M  1677M CPU3     3  17.7H   4.10% bhyve
 1190 nobody        1  20    0    18M  7616K CPU3     3  25:52   0.00% openvpn
 1205 root          1  20    0    13M  2304K select   2   6:46   0.00% powerd
35433 bind         10  52    0    85M    36M sigwai   3   3:27   0.00% named
 1231 ntpd          2  20    0    21M  7128K select   2   2:15   0.00% ntpd
 1272 root          1  20    0    18M  7436K select   1   0:18   0.00% sendmail
  850 root          1  20    0    13M  2504K select   0   0:16   0.00% routed
 1253 root          1  20    0    13M  2584K nanslp   3   0:07   0.00% cron
 1450 minmin        1  20    0    21M  9560K select   2   0:05   0.00% ssh
 1022 root          1  20    0    13M  2836K select   3   0:04   0.00% syslogd
 2386 minmin        1  20    0    21M    10M select   2   0:02   0.00% sshd
  888 root          4  52    0    12M  2228K rpcsvc   0   0:02   0.00% nfscbd
 1198 _dhcp         1  20    0    16M  5988K select   3   0:02   0.00% dhcpd
26924 root          1  20    0    18M  8108K select   1   0:02   0.00% ssh
26911 root          1  20    0    18M  8088K select   1   0:02   0.00% ssh
 1330 minmin        1  20    0    18M  7356K select   1   0:01   0.00% ssh-agent
  862 root          1  20    0    13M  2348K select   3   0:01   0.00% nfsuserd
  864 root          1  20    0    13M  2348K select   2   0:01   0.00% nfsuserd

$
```

デフォルトで表示するプロセスの数は上位20個となっていますが、数値(`top -b 100`等)を指定すれば変更できます。もちろん他のtopのオプションも使えます。

ちなみに、Linuxのtopコマンドも-bでバッチモードになりますが、FreeBSDのtopとは挙動が違うので興味ある方はお試しください。macOSのtopにはバッチモードは用意されていないようです。

## cronでのtopの利用

個人的にはtopのバッチモードはcronを使ってシステムの状況を定期的に記録するような場合に真価を発揮すると思ってます。筆者は10分(あるいはもっと短かい間隔)でtop -bの実行結果を記録し夜間のサーバーの負荷の状況を調べることがあります。

普通にtopを起動した場合はインタラクティブモードと言いますが、インタラクティブモードでは表示できるプロセス数は画面サイズで制限を受けるところを、バッチモードであればそのような制限はありません。cronで定期的にtopの結果を記録する場合には、1000なり2000なり十分大きい数を指定すれば全プロセスを記録できます。

crontab の例

```text
*/10 * * * * top -b 2000 > /var/log/top/$(date +\%Y\%m\%d\%H\%M).top
```

## おわりに

連続してプロセスの変化を観測するなら普通にtopを起動すればいいわけけですが、プロセスの状況を調べ続けるのはかなりの負荷をシステムに与え続けることになります。その点でサマリーを確認するだけであれば、バッチモードはとても便利と言えます。
