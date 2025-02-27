# この御時世にSSDを使うのを止めた話

```console
$ zpool list -v
NAME             SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zroot            228G  23.0G   205G        -         -    15%    10%  1.00x    ONLINE  -
  ada0p3         229G  23.0G   205G        -         -    15%  10.1%      -    ONLINE
zvol0           5.45T  2.77T  2.68T        -         -     1%    50%  1.00x    ONLINE  -
  mirror-0      5.45T  2.77T  2.68T        -         -     1%  50.8%      -    ONLINE
    gpt/sdisk1  5.46T      -      -        -         -      -      -      -    ONLINE
    gpt/sdisk2  5.46T      -      -        -         -      -      -      -    ONLINE
```

```console
$ zpool detach zvol1 gpt/sdisk1
$ zpool labelclear /dev/ada1p1
```

```console
$ gpart show ada1
=>         40  11721045088  ada1  GPT  (5.5T)
           40  11721045088     1  freebsd-zfs  (5.5T)

$ gpart delete -i 1 ada1
ada1p1 deleted
$ gpart add -a 4k -t freebsd-boot -s 512K -l gptboot0 ada1
ada1p1 added
$ gpart add -a 4k -t efi          -s 260M -l efiboot0 ada1
ada1p2 added
$ gpart add -a 4k -t freebsd-swap -s 4G   -l swap0    ada1
ada1p3 added
$ gpart add -a 4k -t freebsd-zfs  -s 5367G  -l hdpool0     ada1
ada1p4 added
$ gpart show ada1
=>         40  11721045088  ada1  GPT  (5.5T)
           40         1024     1  freebsd-boot  (512K)
         1064       532480     2  efi  (260M)
       533544      8388608     3  freebsd-swap  (4.0G)
      8922152  11255414784     4  freebsd-zfs  (5.2T)
  11264336936    456708192        - free -  (218G)

$
```

```console
$ gpart bootcode -b /boot/pmbr -p /boot/gptzfsboot -i 1 ada1
partcode written to ada1p1
bootcode written to ada1
$ zpool create -o altroot=/mnt -O compress=lz4 -O atime=off -m none -f zroot2 gpt/hdpool0
$ zpool list
NAME     SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zroot    228G  23.0G   205G        -         -    15%    10%  1.00x    ONLINE  -
zroot2  5.23T   384K  5.23T        -         -     0%     0%  1.00x    ONLINE  /mnt
zvol0   5.45T  2.77T  2.68T        -         -     1%    50%  1.00x    ONLINE  -
$
```

```console
$ zfs snapshot -r zroot@copy
$ zfs send -R zroot@copy | zfs receive -v -F -u zroot2
$
```

```console
$ zfs snapshot -r zroot@copy1
$ zfs send -R -I zroot@copy zroot@copy1 | zfs receive -v -u zroot2
```

USBでbootする

```console
$ zpool import -R /mnt -N zroot2 zroot
$ zpool export zroot
```

reboot

```console
$ zpool list -v
NAME            SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zroot          5.23T  22.9G  5.21T        -         -     0%     0%  1.00x    ONLINE  -
  gpt/hdpool0  5.24T  22.9G  5.21T        -         -     0%  0.42%      -    ONLINE
zvol0          5.45T  2.77T  2.68T        -         -     1%    50%  1.00x    ONLINE  -
  gpt/sdisk2   5.46T  2.77T  2.68T        -         -     1%  50.8%      -    ONLINE
```

```console

$ zfs create zroot/zvol0
$ zfs send -R zvol0@_daily_2025-03-07 | zfs receive -v -F -u zroot/zvol0
    29134.60 real         2.76 user      3231.93 sys
```

```console
$ zfs snapshot -r zvol0@moving
$ zfs send -R -I zvol0@_daily_2025-03-07 zvol0@moving | zfs receive -v -u zroot/zvol0
      278.42 real         1.69 user        50.69 sys
```

```console
$ kill 1
```

```console
$ zpool export zvol0

zfs rename zroot/zvol0/annually  zroot/annually || exit
sleep 1
zfs rename zroot/zvol0/archive   zroot/archive || exit
sleep 1
zfs rename zroot/zvol0/backup    zroot/backup || exit
sleep 1
zfs rename zroot/zvol0/backupold zroot/backupold || exit
sleep 1
zfs rename zroot/zvol0/home      zroot/home || exit
sleep 1
zfs rename zroot/zvol0/opt       zroot/opt || exit
```

```console
$ mkdir /zvol0
$ zpool import -R /zvol0 -N zvol0
$ zpool destroy zvol0
```

$ gpart delete -i 1 ada1
$ gpart add -a 4k -t freebsd-boot -s 512K -l gptboot1 ada1
ada1p1 added
$ gpart add -a 4k -t efi          -s 260M -l efiboot1 ada1
ada1p2 added
$ gpart add -a 4k -t freebsd-swap -s 4G   -l swap1    ada1
ada1p3 added
$ gpart add -a 4k -t freebsd-zfs  -s 5367G  -l hdpool1     ada1
ada1p4 added
$
```

```console
$ zpool attach zroot gpt/hdpool0 gpt/hdpool1
```

```console
$ zpool status
  pool: zroot
 state: ONLINE
status: One or more devices is currently being resilvered.  The pool will
        continue to function, possibly in a degraded state.
action: Wait for the resilver to complete.
  scan: resilver in progress since Fri Mar  7 23:42:43 2025
        4.39G / 2.79T scanned at 321M/s, 0B / 2.79T issued
        0B resilvered, 0.00% done, no estimated completion time
config:

        NAME             STATE     READ WRITE CKSUM
        zroot            ONLINE       0     0     0
          mirror-0       ONLINE       0     0     0
            gpt/hdpool0  ONLINE       0     0     0
            gpt/hdpool1  ONLINE       0     0     0

errors: No known data errors
$
```
