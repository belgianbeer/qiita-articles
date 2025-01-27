# システムのZFSミラープールを縮小する

<!-- 
https://blog.naa0yama.com/p/02w17-srcjj0yk/
-->

## はじめに

ZFSを使っているシステム

```Console
$ zpool list -v zroot
NAME         SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zroot        230G   699M   229G        -         -     0%     0%  1.00x    ONLINE  -
  mirror-0   230G   699M   229G        -         -     0%  0.29%      -    ONLINE
    da0p4    231G      -      -        -         -      -      -      -    ONLINE
    da1p4    231G      -      -        -         -      -      -      -    ONLINE
$
```

パーティションサイズ

```Console
$ gpart show da0
=>       40  488397088  da0  GPT  (233G)
         40     532480    1  efi  (260M)
     532520       1024    2  freebsd-boot  (512K)
     533544        984       - free -  (492K)
     534528    4194304    3  freebsd-swap  (2.0G)
    4728832  483667968    4  freebsd-zfs  (231G)
  488396800        328       - free -  (164K)

$ gpart show da1
=>       34  625142381  da1  GPT  (298G)
         34          6       - free -  (3.0K)
         40     532480    1  efi  (260M)
     532520       1024    2  freebsd-boot  (512K)
     533544        984       - free -  (492K)
     534528    4194304    3  freebsd-swap  (2.0G)
    4728832  483667968    4  freebsd-zfs  (231G)
  488396800  136745615       - free -  (65G)

$
```

ミラー解除

```Console
$ zpool detach zroot da1p4
$ zpool list -v zroot
NAME        SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zroot       230G   698M   229G        -         -     0%     0%  1.00x    ONLINE  -
  da0p4     231G   698M   229G        -         -     0%  0.29%      -    ONLINE
$
```

100GB にサイズ変更

```Console
$ gpart resize -i 4 -s 100g da1
da1p4 resized
$ gpart show da1
=>       34  625142381  da1  GPT  (298G)
         34          6       - free -  (3.0K)
         40     532480    1  efi  (260M)
     532520       1024    2  freebsd-boot  (512K)
     533544        984       - free -  (492K)
     534528    4194304    3  freebsd-swap  (2.0G)
    4728832  209715200    4  freebsd-zfs  (100G)
  214444032  410698383       - free -  (196G)
$
```

パーティションを作る

```Console
$ zpool add zroot da1p4
$ zpool list -v
NAME        SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zroot       330G   699M   329G        -         -     0%     0%  1.00x    ONLINE  -
  da0p4     231G   699M   229G        -         -     0%  0.29%      -    ONLINE
  da1p4     100G    68K  99.5G        -         -     0%  0.00%      -    ONLINE
$ 
```

remove してみる

```Console
$ zpool remove zroot da0p4
$ zpool list -v
NAME           SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zroot         99.5G   698M  98.8G        -         -     0%     0%  1.00x    ONLINE  -
  indirect-0      -      -      -        -         -      -      -      -    ONLINE
  da1p4        100G   698M  98.8G        -         -     0%  0.68%      -    ONLINE
$
```


```Console
$ zpool status
  pool: zroot
 state: ONLINE
  scan: resilvered 577M in 00:00:15 with 0 errors on Mon Jan 27 14:18:54 2025
remove: Evacuation of /dev/da0p4 in progress since Mon Jan 27 16:10:59 2025
        158M copied out of 698M at 39.4M/s, 22.57% done, 0h0m to go
config:

        NAME        STATE     READ WRITE CKSUM
        zroot       ONLINE       0     0     0
          da0p4     ONLINE       0     0     0  (removing)
          da1p4     ONLINE       0     0     0

errors: No known data errors
$
```

```Console
$ gpart resize -i 4 -s 100g da0
da0p4 resized
$ gpart show da0
=>       40  488397088  da0  GPT  (233G)
         40     532480    1  efi  (260M)
     532520       1024    2  freebsd-boot  (512K)
     533544        984       - free -  (492K)
     534528    4194304    3  freebsd-swap  (2.0G)
    4728832  209715200    4  freebsd-zfs  (100G)
  214444032  273953096       - free -  (131G)

$
```

```Console
$ zpool status
  pool: zroot
 state: ONLINE
status: One or more devices is currently being resilvered.  The pool will
        continue to function, possibly in a degraded state.
action: Wait for the resilver to complete.
  scan: resilver in progress since Mon Jan 27 16:17:09 2025
        698M / 698M scanned, 161M / 698M issued at 40.2M/s
        148M resilvered, 23.06% done, 00:00:13 to go
remove: Removal of vdev 0 copied 698M in 0h0m, completed on Mon Jan 27 16:11:11 2025
        10.1K memory used for removed device mappings
config:

        NAME          STATE     READ WRITE CKSUM
        zroot         ONLINE       0     0     0
          mirror-1    ONLINE       0     0     0
            da1p4     ONLINE       0     0     0
            da0p4     ONLINE       0     0     0  (resilvering)

errors: No known data errors
$
```

```Console
$ zpool attach zroot da1p4 da0p4
$ zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zroot  99.5G   698M  98.8G        -         -     0%     0%  1.00x    ONLINE  -
$ zpool list -v zroot
NAME           SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zroot         99.5G   698M  98.8G        -         -     0%     0%  1.00x    ONLINE  -
  indirect-0      -      -      -        -         -      -      -      -    ONLINE
  mirror-1    99.5G   698M  98.8G        -         -     0%  0.68%      -    ONLINE
    da1p4      100G      -      -        -         -      -      -      -    ONLINE
    da0p4      100G      -      -        -         -      -      -      -    ONLINE
$
```
