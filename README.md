# linuxpro-homework03
## Работа с LVM
[1. Уменьшить том под root / до 8Gb](#1)
[2. Выделить том под /home](#2)
[3. Выделить том под /var (/var - сделать в mirror)](#3)
[4. Для /home - сделать том для снэпшотов](#4)
[5. Прописать монтирование в fstab (попробовать с разными опциями и разными файловыми системами на выбор)](#5)
[6. Работа со снапшотами:](#6)




* * *
<a name="1"/>

### 1. Уменьшить том под root / до 8Gb
Смотрим, что мы имеем
```bash
[root@lvm vagrant]# df -T
Filesystem                      Type     1K-blocks   Used Available Use% Mounted on
/dev/mapper/VolGroup00-LogVol00 xfs       39269648 825012  38444636   3% /
devtmpfs                        devtmpfs    110948      0    110948   0% /dev
tmpfs                           tmpfs       120692      0    120692   0% /dev/shm
tmpfs                           tmpfs       120692   4656    116036   4% /run
tmpfs                           tmpfs       120692      0    120692   0% /sys/fs/cgroup
/dev/sda2                       xfs        1038336  64076    974260   7% /boot
tmpfs                           tmpfs        24140      0     24140   0% /run/user/1000
tmpfs                           tmpfs        24140      0     24140   0% /run/user/0
```
```bash
[root@lvm vagrant]# lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk
├─sda1                    8:1    0    1M  0 part
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part
  ├─VolGroup00-LogVol00 253:0    0 37.5G  0 lvm  /
  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk
sdc                       8:32   0    2G  0 disk
sdd                       8:48   0    1G  0 disk
sde                       8:64   0    1G  0 disk
```

На диске sdb сделаем временный раздел для нашего тома / , так как не лету мы не можем его изменить.

```bash
[root@lvm vagrant]# pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created.
[root@lvm vagrant]# vgcreate vg_tmp_root /dev/sdb
  Volume group "vg_tmp_root" successfully created
[root@lvm vagrant]# lvcreate -n lv_tmp_root -l +100%FREE /dev/vg_tmp_root
  Logical volume "lv_tmp_root" created.
```

Делаем файловую систему, смонтируем ее и посмотрим, что получилось
```bash
[root@lvm vagrant]# mkfs.xfs /dev/vg_tmp_root/lv_tmp_root
meta-data=/dev/vg_tmp_root/lv_tmp_root isize=512    agcount=4, agsize=655104 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2620416, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

[root@lvm vagrant]# mount /dev/vg_tmp_root/lv_tmp_root /mnt
[root@lvm vagrant]# df -T
Filesystem                          Type     1K-blocks   Used Available Use% Mounted on
/dev/mapper/VolGroup00-LogVol00     xfs       39269648 828540  38441108   3% /
devtmpfs                            devtmpfs    110948      0    110948   0% /dev
tmpfs                               tmpfs       120692      0    120692   0% /dev/shm
tmpfs                               tmpfs       120692   4660    116032   4% /run
tmpfs                               tmpfs       120692      0    120692   0% /sys/fs/cgroup
/dev/sda2                           xfs        1038336  64076    974260   7% /boot
tmpfs                               tmpfs        24140      0     24140   0% /run/user/1000
tmpfs                               tmpfs        24140      0     24140   0% /run/user/0
/dev/mapper/vg_tmp_root-lv_tmp_root xfs       10471424  32944  10438480   1% /mnt
[root@lvm vagrant]# lvdisplay
  --- Logical volume ---
  LV Path                /dev/VolGroup00/LogVol00
  LV Name                LogVol00
  VG Name                VolGroup00
  LV UUID                j6b8IV-KEw3-7bTw-Oqy8-1Ud3-juFC-SJBg12
  LV Write Access        read/write
  LV Creation host, time localhost.localdomain, 2018-05-12 18:50:24 +0000
  LV Status              available
  # open                 1
  LV Size                <37.47 GiB
  Current LE             1199
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:0

  --- Logical volume ---
  LV Path                /dev/VolGroup00/LogVol01
  LV Name                LogVol01
  VG Name                VolGroup00
  LV UUID                IAjIC6-ScnM-tvH6-7BTy-TN31-hd82-bgDSzd
  LV Write Access        read/write
  LV Creation host, time localhost.localdomain, 2018-05-12 18:50:25 +0000
  LV Status              available
  # open                 2
  LV Size                1.50 GiB
  Current LE             48
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:1

  --- Logical volume ---
  LV Path                /dev/vg_tmp_root/lv_tmp_root
  LV Name                lv_tmp_root
  VG Name                vg_tmp_root
  LV UUID                lbx4zB-vgia-IAde-PwcJ-oHnB-DW2J-bE6sj4
  LV Write Access        read/write
  LV Creation host, time lvm, 2022-02-14 07:12:37 +0000
  LV Status              available
  # open                 1
  LV Size                <10.00 GiB
  Current LE             2559
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:2

```

Сделаем дамп текущей корневого раздела во временный
```bash
[root@lvm vagrant]# xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt
xfsrestore: using file dump (drive_simple) strategy
xfsrestore: version 3.1.7 (dump format 3.0)
xfsdump: using file dump (drive_simple) strategy
xfsdump: version 3.1.7 (dump format 3.0)
xfsdump: level 0 dump of lvm:/
xfsdump: dump date: Mon Feb 14 07:29:36 2022
xfsdump: session id: a9c50842-33f9-40fb-8215-12df579eed76
xfsdump: session label: ""
xfsdump: ino map phase 1: constructing initial dump list
xfsrestore: searching media for dump
xfsdump: ino map phase 2: skipping (no pruning necessary)
xfsdump: ino map phase 3: skipping (only one dump stream)
xfsdump: ino map construction complete
xfsdump: estimated dump size: 809407232 bytes
xfsdump: creating dump session media file 0 (media 0, file 0)
xfsdump: dumping ino map
xfsdump: dumping directories
xfsrestore: examining media file 0
xfsrestore: dump description:
xfsrestore: hostname: lvm
xfsrestore: mount point: /
xfsrestore: volume: /dev/mapper/VolGroup00-LogVol00
xfsrestore: session time: Mon Feb 14 07:29:36 2022
xfsrestore: level: 0
xfsrestore: session label: ""
xfsrestore: media label: ""
xfsrestore: file system id: b60e9498-0baa-4d9f-90aa-069048217fee
xfsrestore: session id: a9c50842-33f9-40fb-8215-12df579eed76
xfsrestore: media id: dd2dec62-7717-411c-bd9e-7679b4f07ea5
xfsrestore: searching media for directory dump
xfsrestore: reading directories
xfsdump: dumping non-directory files
xfsrestore: 2740 directories and 23787 entries processed
xfsrestore: directory post-processing
xfsrestore: restoring non-directory files
xfsdump: ending media file
xfsdump: media file size 786225888 bytes
xfsdump: dump size (non-dir files) : 772955784 bytes
xfsdump: dump complete: 30 seconds elapsed
xfsdump: Dump Status: SUCCESS
xfsrestore: restore complete: 30 seconds elapsed
xfsrestore: Restore Status: SUCCESS

```

# ! Дальше какая-то магия, очень хочется услышать разбор данных действий на лекции.

Заходим в окружение chroot нашего временного корня:
```bash
[root@lvm vagrant]# for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
[root@lvm vagrant]# chroot /mnt/
```

Запишем новый загрузчик:
```bash
[root@lvm /]# grub2-mkconfig -o /boot/grub2/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
done
```
Обновляем образы загрузки:
```bash
[root@lvm /]# cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done
Executing: /sbin/dracut -v initramfs-3.10.0-862.2.3.el7.x86_64.img 3.10.0-862.2.3.el7.x86_64 --force
dracut module 'busybox' will not be installed, because command 'busybox' could not be found!
dracut module 'crypt' will not be installed, because command 'cryptsetup' could not be found!
dracut module 'dmraid' will not be installed, because command 'dmraid' could not be found!
dracut module 'dmsquash-live-ntfs' will not be installed, because command 'ntfs-3g' could not be found!
dracut module 'multipath' will not be installed, because command 'multipath' could not be found!
dracut module 'busybox' will not be installed, because command 'busybox' could not be found!
dracut module 'crypt' will not be installed, because command 'cryptsetup' could not be found!
dracut module 'dmraid' will not be installed, because command 'dmraid' could not be found!
dracut module 'dmsquash-live-ntfs' will not be installed, because command 'ntfs-3g' could not be found!
dracut module 'multipath' will not be installed, because command 'multipath' could not be found!
*** Including module: bash ***
*** Including module: nss-softokn ***
*** Including module: i18n ***
*** Including module: drm ***
*** Including module: plymouth ***
*** Including module: dm ***
Skipping udev rule: 64-device-mapper.rules
Skipping udev rule: 60-persistent-storage-dm.rules
Skipping udev rule: 55-dm.rules
*** Including module: kernel-modules ***
Omitting driver floppy
*** Including module: lvm ***
Skipping udev rule: 64-device-mapper.rules
Skipping udev rule: 56-lvm.rules
Skipping udev rule: 60-persistent-storage-lvm.rules
*** Including module: qemu ***
*** Including module: resume ***
*** Including module: rootfs-block ***
*** Including module: terminfo ***
*** Including module: udev-rules ***
Skipping udev rule: 40-redhat-cpu-hotplug.rules
Skipping udev rule: 91-permissions.rules
*** Including module: biosdevname ***
*** Including module: systemd ***
*** Including module: usrmount ***
*** Including module: base ***
*** Including module: fs-lib ***
*** Including module: shutdown ***
*** Including modules done ***
*** Installing kernel module dependencies and firmware ***
*** Installing kernel module dependencies and firmware done ***
*** Resolving executable dependencies ***
*** Resolving executable dependencies done***
*** Hardlinking files ***
*** Hardlinking files done ***
*** Stripping files ***
*** Stripping files done ***
*** Generating early-microcode cpio image contents ***
*** No early-microcode cpio image needed ***
*** Store current command line parameters ***
*** Creating image file ***
*** Creating image file done ***
*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***
```

в файле /boot/grub2/grub.cfg меняем
```
lv=VolGroup00/LogVol01
```
на
```
lv=vg_tmp_root/lv_tmp_root
```

Выходим из окружения chroot и перезагружаем компьютер:
```bash
[root@lvm vagrant]# shutdown -r now
Connection to 127.0.0.1 closed by remote host.
Connection to 127.0.0.1 closed.
```
# Тут магия заканчивается.

В теории мы должны загрузится с временного раздела. Проверим :)
```bash
ujack@ubuntu2004:~/linuxpro-homework03$ vagrant ssh
Last login: Mon Feb 14 06:21:41 2022 from 10.0.2.2
[vagrant@lvm ~]$ lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                         8:0    0   40G  0 disk
├─sda1                      8:1    0    1M  0 part
├─sda2                      8:2    0    1G  0 part /boot
└─sda3                      8:3    0   39G  0 part
  ├─VolGroup00-LogVol00   253:1    0 37.5G  0 lvm
  └─VolGroup00-LogVol01   253:2    0  1.5G  0 lvm  [SWAP]
sdb                         8:16   0   10G  0 disk
└─vg_tmp_root-lv_tmp_root 253:0    0   10G  0 lvm  /
sdc                         8:32   0    2G  0 disk
sdd                         8:48   0    1G  0 disk
sde                         8:64   0    1G  0 disk
[vagrant@lvm ~]$
```
Каким-то чудом это все работает :)
Что мы видим - наш рут раздел сейчас находится на диске sdb раздела vg_tmp_root-lv_tmp_root

###  Уменьшаем раздел и возвращаем с него загрузку

Теперь, так как основной раздел / свободен и не подмонтирован, мы можем его изменять (уменьшить! Увеличить можно было бы проще).

Удаляем старый логический том
```bash
[root@lvm vagrant]# lvremove /dev/VolGroup00/LogVol00
Do you really want to remove active logical volume VolGroup00/LogVol00? [y/n]: y
  Logical volume "LogVol00" successfully removed
```

Создаем новый логический том нужного нам размера.
```bash
[root@lvm vagrant]# lvcreate -n LogVol00 -L 8G VolGroup00
WARNING: xfs signature detected on /dev/VolGroup00/LogVol00 at offset 0. Wipe it? [y/n]: y
  Wiping xfs signature on /dev/VolGroup00/LogVol00.
  Logical volume "LogVol00" created.
```

Создаем на нем файловую систему и монтируем его:
```bash
[root@lvm vagrant]# mkfs.xfs /dev/VolGroup00/LogVol00
meta-data=/dev/VolGroup00/LogVol00 isize=512    agcount=4, agsize=524288 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2097152, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
[root@lvm vagrant]# mount /dev/VolGroup00/LogVol00 /mnt
```

Возвращаем обратно содержимое корня.
```bash
[root@lvm vagrant]# xfsdump -J - /dev/vg_tmp_root/lv_tmp_root | xfsrestore -J - /mnt
xfsdump: using file dump (drive_simple) strategy
xfsdump: version 3.1.7 (dump format 3.0)
xfsdump: level 0 dump of lvm:/
xfsdump: dump date: Mon Feb 14 09:01:03 2022
xfsdump: session id: b334719c-7144-402b-adab-1a5eb95b955a
xfsdump: session label: ""
xfsrestore: using file dump (drive_simple) strategy
xfsrestore: version 3.1.7 (dump format 3.0)
xfsrestore: searching media for dump
xfsdump: ino map phase 1: constructing initial dump list
xfsdump: ino map phase 2: skipping (no pruning necessary)
xfsdump: ino map phase 3: skipping (only one dump stream)
xfsdump: ino map construction complete
xfsdump: estimated dump size: 810126336 bytes
xfsdump: creating dump session media file 0 (media 0, file 0)
xfsdump: dumping ino map
xfsdump: dumping directories
xfsrestore: examining media file 0
xfsrestore: dump description:
xfsrestore: hostname: lvm
xfsrestore: mount point: /
xfsrestore: volume: /dev/mapper/vg_tmp_root-lv_tmp_root
xfsrestore: session time: Mon Feb 14 09:01:03 2022
xfsrestore: level: 0
xfsrestore: session label: ""
xfsrestore: media label: ""
xfsrestore: file system id: 3a7594e0-9e1f-42b3-bea3-9ddbae514d53
xfsrestore: session id: b334719c-7144-402b-adab-1a5eb95b955a
xfsrestore: media id: a9b43f8a-9ddf-43d2-abe5-2be94b0bc772
xfsrestore: searching media for directory dump
xfsrestore: reading directories
xfsdump: dumping non-directory files
xfsrestore: 2783 directories and 23935 entries processed
xfsrestore: directory post-processing
xfsrestore: restoring non-directory files
xfsdump: ending media file
xfsdump: media file size 786785832 bytes
xfsdump: dump size (non-dir files) : 773419096 bytes
xfsdump: dump complete: 26 seconds elapsed
xfsdump: Dump Status: SUCCESS
xfsrestore: restore complete: 27 seconds elapsed
xfsrestore: Restore Status: SUCCESS
```

Заходим в окружение chroot нашего временного корня:
```bash
[root@lvm vagrant]# for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
[root@lvm vagrant]# chroot /mnt/
```

Запишем новый загрузчик
```bash
[root@lvm /]# grub2-mkconfig -o /boot/grub2/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
done
```

Обновляем образы загрузки:
```bash
[root@lvm /]# cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done
Executing: /sbin/dracut -v initramfs-3.10.0-862.2.3.el7.x86_64.img 3.10.0-862.2.3.el7.x86_64 --force
dracut module 'busybox' will not be installed, because command 'busybox' could not be found!
dracut module 'crypt' will not be installed, because command 'cryptsetup' could not be found!
dracut module 'dmraid' will not be installed, because command 'dmraid' could not be found!
dracut module 'dmsquash-live-ntfs' will not be installed, because command 'ntfs-3g' could not be found!
dracut module 'multipath' will not be installed, because command 'multipath' could not be found!
dracut module 'busybox' will not be installed, because command 'busybox' could not be found!
dracut module 'crypt' will not be installed, because command 'cryptsetup' could not be found!
dracut module 'dmraid' will not be installed, because command 'dmraid' could not be found!
dracut module 'dmsquash-live-ntfs' will not be installed, because command 'ntfs-3g' could not be found!
dracut module 'multipath' will not be installed, because command 'multipath' could not be found!
*** Including module: bash ***
*** Including module: nss-softokn ***
*** Including module: i18n ***
*** Including module: drm ***
*** Including module: plymouth ***
*** Including module: dm ***
Skipping udev rule: 64-device-mapper.rules
Skipping udev rule: 60-persistent-storage-dm.rules
Skipping udev rule: 55-dm.rules
*** Including module: kernel-modules ***
Omitting driver floppy
*** Including module: lvm ***
Skipping udev rule: 64-device-mapper.rules
Skipping udev rule: 56-lvm.rules
Skipping udev rule: 60-persistent-storage-lvm.rules
*** Including module: qemu ***
*** Including module: resume ***
*** Including module: rootfs-block ***
*** Including module: terminfo ***
*** Including module: udev-rules ***
Skipping udev rule: 40-redhat-cpu-hotplug.rules
Skipping udev rule: 91-permissions.rules
*** Including module: biosdevname ***
*** Including module: systemd ***
*** Including module: usrmount ***
*** Including module: base ***
*** Including module: fs-lib ***
*** Including module: shutdown ***
*** Including modules done ***
*** Installing kernel module dependencies and firmware ***
*** Installing kernel module dependencies and firmware done ***
*** Resolving executable dependencies ***
*** Resolving executable dependencies done***
*** Hardlinking files ***
*** Hardlinking files done ***
*** Stripping files ***
*** Stripping files done ***
*** Generating early-microcode cpio image contents ***
*** No early-microcode cpio image needed ***
*** Store current command line parameters ***
*** Creating image file ***
*** Creating image file done ***
*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***
```

Проверяем, что в /boot/grub2/grub.cfg нужные значения
```
lv=VolGroup00/LogVol00
```

Выходим из окружения chroot и перезагружаем машинку:
```bash
[root@lvm vagrant]# shutdown -r now
Connection to 127.0.0.1 closed by remote host.
Connection to 127.0.0.1 closed.
```

Проверяем результат:
```bash
~/linuxpro-homework03$ vagrant ssh
Last login: Mon Feb 14 08:39:13 2022 from 10.0.2.2
[vagrant@lvm ~]$ lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                         8:0    0   40G  0 disk
├─sda1                      8:1    0    1M  0 part
├─sda2                      8:2    0    1G  0 part /boot
└─sda3                      8:3    0   39G  0 part
  ├─VolGroup00-LogVol00   253:0    0    8G  0 lvm  /
  └─VolGroup00-LogVol01   253:1    0  1.5G  0 lvm  [SWAP]
sdb                         8:16   0   10G  0 disk
└─vg_tmp_root-lv_tmp_root 253:2    0   10G  0 lvm
sdc                         8:32   0    2G  0 disk
sdd                         8:48   0    1G  0 disk
sde                         8:64   0    1G  0 disk
```

Ура, root раздел работает и имеет нужные нам 8Gb.
Удаляем временный том:
```bash
[vagrant@lvm ~]$ sudo su
[root@lvm vagrant]# lvremove /dev/vg_tmp_root/lv_tmp_root
Do you really want to remove active logical volume vg_tmp_root/lv_tmp_root? [y/n]: y
  Logical volume "lv_tmp_root" successfully removed
[root@lvm vagrant]# vgremove vg_tmp_root
  Volume group "vg_tmp_root" successfully removed
[root@lvm vagrant]# pvremove /dev/sdb
  Labels on physical volume "/dev/sdb" successfully wiped.

```

итого
```bash
[root@lvm vagrant]# lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk
├─sda1                    8:1    0    1M  0 part
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part
  ├─VolGroup00-LogVol00 253:0    0    8G  0 lvm  /
  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk
sdc                       8:32   0    2G  0 disk
sdd                       8:48   0    1G  0 disk
sde                       8:64   0    1G  0 disk
```

* * *
<a name="2"/>

### 2. Выделить том под /home

Текущая конфигурация:
```
[root@lvm vagrant]#  lvmdiskscan
  /dev/sda2 [       1.00 GiB]
  /dev/sda3 [     <39.00 GiB] LVM physical volume
  /dev/sdb  [      10.00 GiB]
  /dev/sdc  [       2.00 GiB]
  /dev/sdd  [       1.00 GiB]
  /dev/sde  [       1.00 GiB]
  4 disks
  1 partition
  0 LVM physical volume whole disks
  1 LVM physical volume
```

разметим диск:
```bash
[root@lvm vagrant]# pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created.
```
Создадим Volume Group
```bash
[root@lvm vagrant]# vgcreate test_vg /dev/sdb
  Volume group "test_vg" successfully created
```
Создадим Logical Volume
```
[root@lvm vagrant]# lvcreate -l+80%FREE -n test_lv test_vg
WARNING: xfs signature detected on /dev/test_vg/test_lv at offset 0. Wipe it? [y/n]: y
  Wiping xfs signature on /dev/test_vg/test_lv.
  Logical volume "test_lv" created.
```
Что получилось
```bash
[root@lvm vagrant]# vgdisplay test_vg
  --- Volume group ---
  VG Name               test_vg
  System ID
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  2
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                1
  Open LV               0
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               <10.00 GiB
  PE Size               4.00 MiB
  Total PE              2559
  Alloc PE / Size       2047 / <8.00 GiB
  Free  PE / Size       512 / 2.00 GiB
  VG UUID               Uyxp3W-Pwrh-7g0e-Rh7d-UZA8-0mTD-DWYdGO
```

Создадим на LV файловуĀ систему и смонтируем его
```bash
[root@lvm vagrant]#  mkfs.ext4 /dev/test_vg/test_lv
mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
524288 inodes, 2096128 blocks
104806 blocks (5.00%) reserved for the super user
First data block=0
Maximum filesystem blocks=2147483648
64 block groups
32768 blocks per group, 32768 fragments per group
8192 inodes per group
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done
```

монтируем пока в папку /mnt
```bash
[root@lvm vagrant]# mount /dev/test_vg/test_lv /mnt
```

Копируем содержимое папки home в mnt
```bash
cp -a /home/* /mnt
```

Удаляем файлы из "старого" home
```bash
[root@lvm vagrant]# rm -rf /home/*
[root@lvm vagrant]# ll /home/
total 0
```

Перемонтируем наш диск в home
```bash
[root@lvm vagrant]# umount /mnt/
[root@lvm vagrant]# mount /dev/test_vg/test_lv /home
[root@lvm vagrant]# ll /home
total 20
drwx------. 2 root    root    16384 Feb 14 10:38 lost+found
drwx------. 3 vagrant vagrant  4096 May 12  2018 vagrant
```

Пропишем монтирование а /etc/fstab чтобы у нас все сохранилось при перезагрузки
```bash
[root@lvm vagrant]# echo "/dev/test_vg/test_lv /home      ext4    defaults        0 0" >> /etc/fstab
```

После перезагрузки проверяем как оно там у нас
```bash
[vagrant@lvm ~]$ lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk
├─sda1                    8:1    0    1M  0 part
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part
  ├─VolGroup00-LogVol00 253:0    0    8G  0 lvm  /
  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk
└─test_vg-test_lv       253:2    0    8G  0 lvm  /home
sdc                       8:32   0    2G  0 disk
sdd                       8:48   0    1G  0 disk
sde                       8:64   0    1G  0 disk
```
Отлично - /home в своем разделе.

* * *
<a name="3" />

### 3. выделить том под /var (/var - сделать в mirror)

На свободных дисках sdd, sde сделаем зеркальный том под /var

```bash
[root@lvm vagrant]# pvcreate /dev/sdd /dev/sde
  Physical volume "/dev/sdd" successfully created.
  Physical volume "/dev/sde" successfully created.
```

создаем зеркальный LV
```bash
[root@lvm vagrant]# lvcreate -L 950M -m1 -n lv_var vg_var
  Rounding up size to full physical extent 952.00 MiB
  Logical volume "lv_var" created.
```

Создаем на нем ФС и перемещаем туда /var:
```
[root@lvm vagrant]# mkfs.ext4 /dev/vg_var/lv_var
mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
60928 inodes, 243712 blocks
12185 blocks (5.00%) reserved for the super user
First data block=0
Maximum filesystem blocks=249561088
8 block groups
32768 blocks per group, 32768 fragments per group
7616 inodes per group
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376

Allocating group tables: done
Writing inode tables: done
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

[root@lvm vagrant]# mount /dev/vg_var/lv_var /mnt
[root@lvm vagrant]# cp -aR /var/* /mnt/
[root@lvm vagrant]# ll /mnt/
total 80
drwxr-xr-x.  2 root root  4096 Apr 11  2018 adm
drwxr-xr-x.  5 root root  4096 May 12  2018 cache
drwxr-xr-x.  3 root root  4096 May 12  2018 db
drwxr-xr-x.  3 root root  4096 May 12  2018 empty
drwxr-xr-x.  2 root root  4096 Apr 11  2018 games
drwxr-xr-x.  2 root root  4096 Apr 11  2018 gopher
drwxr-xr-x.  3 root root  4096 May 12  2018 kerberos
drwxr-xr-x. 28 root root  4096 Feb 14 09:01 lib
drwxr-xr-x.  2 root root  4096 Apr 11  2018 local
lrwxrwxrwx.  1 root root    11 Feb 14 09:01 lock -> ../run/lock
drwxr-xr-x.  8 root root  4096 Feb 14 11:53 log
drwx------.  2 root root 16384 Feb 14 12:06 lost+found
lrwxrwxrwx.  1 root root    10 Feb 14 09:01 mail -> spool/mail
drwxr-xr-x.  2 root root  4096 Apr 11  2018 nis
drwxr-xr-x.  2 root root  4096 Apr 11  2018 opt
drwxr-xr-x.  2 root root  4096 Apr 11  2018 preserve
lrwxrwxrwx.  1 root root     6 Feb 14 09:01 run -> ../run
drwxr-xr-x.  8 root root  4096 May 12  2018 spool
drwxrwxrwt.  5 root root  4096 Feb 14 11:53 tmp
drwxr-xr-x.  2 root root  4096 Apr 11  2018 yp
```

На всāкий случай сохранāем содержимое старого
```
[root@lvm vagrant]# mkdir /tmp/oldvar && mv /var/* /tmp/oldvar
mv: cannot move ‘/var/tmp’ to ‘/tmp/oldvar/tmp’: Device or resource busy
[root@lvm vagrant]# ll /var
total 0
drwxrwxrwt. 5 root root 243 Feb 14 11:53 tmp
```
Видимл папку tmp нельзя удалить, пока на нее можно забить.

Отмантируем старую, примонтируем новую, и проверим
```
[root@lvm vagrant]# umount /mnt
[root@lvm vagrant]# mount /dev/vg_var/lv_var /var
[root@lvm vagrant]# df -T
Filesystem                      Type     1K-blocks   Used Available Use% Mounted on
/dev/mapper/VolGroup00-LogVol00 xfs        8378368 829412   7548956  10% /
devtmpfs                        devtmpfs    111840      0    111840   0% /dev
tmpfs                           tmpfs       120692      0    120692   0% /dev/shm
tmpfs                           tmpfs       120692   4620    116072   4% /run
tmpfs                           tmpfs       120692      0    120692   0% /sys/fs/cgroup
/dev/mapper/test_vg-test_lv     ext4       8121784  36880   7649296   1% /home
/dev/sda2                       xfs        1038336  62292    976044   6% /boot
tmpfs                           tmpfs        24140      0     24140   0% /run/user/1000
/dev/mapper/vg_var-lv_var       ext4        943128 186140    691864  22% /var

```

Правим fstab длā автоматического монтированиā /var: (как в методичке)
```
[root@lvm vagrant]# cat /etc/fstab

#
# /etc/fstab
# Created by anaconda on Sat May 12 18:50:26 2018
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/VolGroup00-LogVol00 /                       xfs     defaults        0 0
UUID=570897ca-e759-4c81-90cf-389da6eee4cc /boot                   xfs     defaults        0 0
/dev/mapper/VolGroup00-LogVol01 swap                    swap    defaults        0 0
#VAGRANT-BEGIN
# The contents below are automatically generated by Vagrant. Do not modify.
#VAGRANT-END
/dev/test_vg/test_lv /home      ext4    defaults        0 0
UUID="7d01c93b-a483-498b-a10e-3819133898e5" /var ext4 defaults 0 0
```

На этот раз примонтировали по UUID
Перезагрузимся, посмотрим, что получилось
```
~/linuxpro-homework03$ vagrant ssh
Last login: Mon Feb 14 11:53:19 2022 from 10.0.2.2
[vagrant@lvm ~]$ lsblk
NAME                     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                        8:0    0   40G  0 disk
├─sda1                     8:1    0    1M  0 part
├─sda2                     8:2    0    1G  0 part /boot
└─sda3                     8:3    0   39G  0 part
  ├─VolGroup00-LogVol00  253:0    0    8G  0 lvm  /
  └─VolGroup00-LogVol01  253:1    0  1.5G  0 lvm  [SWAP]
sdb                        8:16   0   10G  0 disk
└─test_vg-test_lv        253:2    0    8G  0 lvm  /home
sdc                        8:32   0    2G  0 disk
sdd                        8:48   0    1G  0 disk
├─vg_var-lv_var_rmeta_0  253:3    0    4M  0 lvm
│ └─vg_var-lv_var        253:7    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_0 253:4    0  952M  0 lvm
  └─vg_var-lv_var        253:7    0  952M  0 lvm  /var
sde                        8:64   0    1G  0 disk
├─vg_var-lv_var_rmeta_1  253:5    0    4M  0 lvm
│ └─vg_var-lv_var        253:7    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_1 253:6    0  952M  0 lvm
  └─vg_var-lv_var        253:7    0  952M  0 lvm  /var
```
Прекрасно!
* * *
<a name="4"/>
<a name="6"/>

### 4. для /home - сделать том для снэпшотов
### 6. Работа со снапшотами:

*далее 2 пуккта ДЗ вместе*

Сгенерируем файлý в /home/
```
[root@lvm vagrant]#  touch /home/file{1..20}
[root@lvm vagrant]# ll /home/
total 20
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file1
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file10
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file11
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file12
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file13
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file14
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file15
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file16
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file17
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file18
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file19
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file2
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file20
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file3
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file4
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file5
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file6
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file7
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file8
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file9
drwx------. 2 root    root    16384 Feb 14 10:38 lost+found
drwx------. 3 vagrant vagrant  4096 Feb 14 11:52 vagrant
```

Снимем снапшот:
```
[root@lvm vagrant]# lvcreate -L 100MB -s -n home_snap /dev/test_vg/test_lv
  Logical volume "home_snap" created.
```

Удалим часть файлов
```
[root@lvm vagrant]# rm -f /home/file{11..20}
[root@lvm vagrant]# ll /home/
total 20
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file1
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file10
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file2
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file3
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file4
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file5
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file6
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file7
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file8
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file9
drwx------. 2 root    root    16384 Feb 14 10:38 lost+found
drwx------. 3 vagrant vagrant  4096 Feb 14 11:52 vagrant
```

Процесс восстановлениā со снапшота:
```
[root@lvm /]# umount /home
[root@lvm /]# lvconvert --merge /dev/test_vg/home_snap
  Merging of volume test_vg/home_snap started.
  test_vg/test_lv: Merged: 99.99%
[root@lvm /]# mount /home
[root@lvm /]# ll /home/
total 20
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file1
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file10
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file11
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file12
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file13
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file14
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file15
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file16
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file17
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file18
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file19
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file2
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file20
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file3
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file4
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file5
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file6
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file7
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file8
-rw-r--r--. 1 root    root        0 Feb 14 12:42 file9
drwx------. 2 root    root    16384 Feb 14 10:38 lost+found
drwx------. 3 vagrant vagrant  4096 Feb 14 11:52 vagrant
```
все на месте!

При ошибке
```
[root@lvm vagrant]# umount /home/
umount: /home: target is busy.
        (In some cases useful info about processes that use
         the device is found by lsof(8) or fuser(1))
```
значит папка используется и не может быть отмантирована. Необходимо по user vagrant выйти из нее
```
cd /
```
и только после делать
```
sudo su
```

* * *
<a name="5"/>

### 5. прописать монтирование в fstab (попробовать с разными опциями и разными файловыми системами на выбор)

выполнено в процессе предыдущих пунктов
```
[root@lvm /]# cat /etc/fstab

#
# /etc/fstab
# Created by anaconda on Sat May 12 18:50:26 2018
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/VolGroup00-LogVol00 /                       xfs     defaults        0 0
UUID=570897ca-e759-4c81-90cf-389da6eee4cc /boot                   xfs     defaults        0 0
/dev/mapper/VolGroup00-LogVol01 swap                    swap    defaults        0 0
#VAGRANT-BEGIN
# The contents below are automatically generated by Vagrant. Do not modify.
#VAGRANT-END
/dev/test_vg/test_lv /home      ext4    defaults        0 0
UUID="7d01c93b-a483-498b-a10e-3819133898e5" /var ext4 defaults 0 0
```



