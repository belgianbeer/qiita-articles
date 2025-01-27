# システムのZFSミラープールを縮小する

## はじめに

ZFSを使っているシステム



```cmd
minmin@zfstest(0)$ /sbin/zpool list -v zroot
NAME         SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zroot        230G   699M   229G        -         -     0%     0%  1.00x    ONLINE  -
  mirror-0   230G   699M   229G        -         -     0%  0.29%      -    ONLINE
    da0p4    231G      -      -        -         -      -      -      -    ONLINE
    da1p4    231G      -      -        -         -      -      -      -    ONLINE
minmin@zfstest(0)$
```
