# Домашнее задание на тему "Работа с LVM"

Задание:
На виртуальной машине с Ubuntu 24.04 и LVM.
1. Уменьшить том под / до 8G.
2. Выделить том под /var - сделать в mirror.
3. Выделить том под /home.
4. /home - сделать том для снапшотов.
5. Прописать монтирование в fstab. Попробовать с разными опциями и разными файловыми системами (на выбор).
6. Работа со снапшотами:
7. сгенерить файлы в /home/;
8. снять снапшот;
9. удалить часть файлов;
10. восстановится со снапшота.

Для выполнения работы была поднята ВМ с ОС Ubuntu server 24.04 и добавлены диски на 10Gb, 2Gb, 1Gb и 1Gb.
Все действия будем выполнять с правами суперпользователя:
```
nikser@linux-lvm:~$ lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0   25G  0 disk 
├─sda1                      8:1    0    1M  0 part 
├─sda2                      8:2    0    2G  0 part /boot
└─sda3                      8:3    0   23G  0 part 
  └─ubuntu--vg-ubuntu--lv 252:0    0 11,5G  0 lvm  /
sdb                         8:16   0   10G  0 disk 
sdc                         8:32   0    2G  0 disk 
sdd                         8:48   0    1G  0 disk 
sde                         8:64   0    1G  0 disk 
sr0                        11:0    1 1024M  1 rom  
nikser@linux-lvm:~$ sudo -i
[sudo] password for nikser: 
root@linux-lvm:~# 
```

### 1. Уменьшить том под / до 8G.

Подготовлю временный том для / раздела:
```
root@linux-lvm:~# pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created.

root@linux-lvm:~# vgcreate vg_tmproot /dev/sdb
  Volume group "vg_tmproot" successfully created

root@linux-lvm:~# lvcreate -n lv_tmproot -l +100%FREE /dev/vg_tmproot
  Logical volume "lv_tmproot" created.
```

Теперь создам на нём файловую систему и смонтирую его для переноса **/** раздела:
```
root@linux-lvm:~# mkfs.ext4 /dev/vg_tmproot/lv_tmproot
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 2620416 4k blocks and 655360 inodes
Filesystem UUID: 5b1aa97d-1b00-4f69-aa5f-90605d033222
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done 

root@linux-lvm:~# mount /dev/vg_tmproot/lv_tmproot /mnt
```

Затем с помощью **rsync** скопирую данные с **/** в **/mnt**
```
root@linux-lvm:~# rsync -avxHAX --progress / /mnt/

sent 4.799.767.812 bytes  received 1.592.035 bytes  34.918.980,71 bytes/sec
total size is 4.796.912.198  speedup is 1,00
```

```
root@linux-lvm:~# ls -l  /mnt
total 2097252
lrwxrwxrwx   1 root root          7 апр 22  2024 bin -> usr/bin
drwxr-xr-x   2 root root       4096 июн  5 14:57 bin.usr-is-merged
drwxr-xr-x   2 root root       4096 окт 18 11:23 boot
dr-xr-xr-x   2 root root       4096 сен  9 09:11 cdrom
drwxr-xr-x   2 root root       4096 окт 18 12:52 dev
drwxr-xr-x 108 root root       4096 окт 18 11:57 etc
drwxr-xr-x   3 root root       4096 окт 18 11:15 home
lrwxrwxrwx   1 root root          7 апр 22  2024 lib -> usr/lib
lrwxrwxrwx   1 root root          9 апр 22  2024 lib64 -> usr/lib64
drwxr-xr-x   2 root root       4096 июн  5 14:57 lib.usr-is-merged
drwx------   2 root root      16384 окт 18 11:06 lost+found
drwxr-xr-x   2 root root       4096 сен  9 06:29 media
drwxr-xr-x   2 root root       4096 окт 18 12:57 mnt
drwxr-xr-x   2 root root       4096 сен  9 06:29 opt
dr-xr-xr-x   2 root root       4096 окт 18 11:59 proc
drwx------   3 root root       4096 сен  9 06:37 root
drwxr-xr-x   2 root root       4096 окт 18 12:00 run
lrwxrwxrwx   1 root root          8 апр 22  2024 sbin -> usr/sbin
drwxr-xr-x   2 root root       4096 мар 18  2026 sbin.usr-is-merged
drwxr-xr-x   2 root root       4096 окт 18 11:57 snap
drwxr-xr-x   2 root root       4096 сен  9 06:29 srv
-rw-------   1 root root 2147483648 окт 18 11:14 swap.img
dr-xr-xr-x   2 root root       4096 окт 18 11:59 sys
drwxrwxrwt  14 root root       4096 окт 18 12:00 tmp
drwxr-xr-x  12 root root       4096 сен  9 06:29 usr
drwxr-xr-x  13 root root       4096 окт 18 11:57 var
```

Теперь нужно сконфигурировать **grub**, чтобы после перезагрузки попасть в новый **/**. Для этого нужно будет сымитировать текущий **root**, сделать в него **chroot** и обновить **grub**:
```
root@linux-lvm:~# for i in /proc/ /sys/ /dev/ /boot/ /run/; do mount --bind $i /mnt/$i; done 

root@linux-lvm:~# chroot /mnt/    

root@linux-lvm:/# grub-mkconfig -o /boot/grub/grub.cfg 
Sourcing file `/etc/default/grub'
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-6.8.0-147-generic
Found initrd image: /boot/initrd.img-6.8.0-147-generic
Warning: os-prober will not be executed to detect other bootable partitions.
Systems on them will not be added to the GRUB boot configuration.
Check GRUB_DISABLE_OS_PROBER documentation entry.
Adding boot menu entry for UEFI Firmware Settings ...
done
```

Обновлю **initrd** и перезагружусь для работы с новым разделом:
```
root@linux-lvm:/# update-initramfs -u
update-initramfs: Generating /boot/initrd.img-6.8.0-147-generic

root@linux-lvm:/# exit
exit

root@linux-lvm:~# reboot
```

Проверяю, что произошло после перезагрузки:
```
root@linux-lvm:~# lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0   25G  0 disk 
├─sda1                      8:1    0    1M  0 part 
├─sda2                      8:2    0    2G  0 part /boot
└─sda3                      8:3    0   23G  0 part 
  └─ubuntu--vg-ubuntu--lv 252:1    0 11,5G  0 lvm  
sdb                         8:16   0   10G  0 disk 
└─vg_tmproot-lv_tmproot   252:0    0   10G  0 lvm  /
sdc                         8:32   0    2G  0 disk 
sdd                         8:48   0    1G  0 disk 
sde                         8:64   0    1G  0 disk 
```

Теперь изменю размер старой **vg** с 12Gb на 8Gb и верну на него **/**. Для этого удалю старый **lv** на 12Gb и создам новый на 8Gb:
```
root@linux-lvm:~# lvremove /dev/ubuntu-vg/ubuntu-lv 
Do you really want to remove and DISCARD active logical volume ubuntu-vg/ubuntu-lv? [y/n]: y
  Logical volume "ubuntu-lv" successfully removed.

root@linux-lvm:~# lvcreate -n ubuntu-lv -L 8G /dev/ubuntu-vg
WARNING: ext4 signature detected on /dev/ubuntu-vg/ubuntu-lv at offset 1080. Wipe it? [y/n]: y
  Wiping ext4 signature on /dev/ubuntu-vg/ubuntu-lv.
  Logical volume "ubuntu-lv" created.
```

Далее создаю ФС, монтирую, копирую данные и конфигурирую grub:
```
root@linux-lvm:~# mkfs.ext4 /dev/ubuntu-vg/ubuntu-lv 
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 2097152 4k blocks and 524288 inodes
Filesystem UUID: bbe4a2fa-a295-40cf-9f64-e349368bdce3
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done 

root@linux-lvm:~# mount /dev/ubuntu-vg/ubuntu-lv /mnt

root@linux-lvm:~# rsync -avxHAX --progress / /mnt/
sent 4.825.156.768 bytes  received 1.592.174 bytes  34.600.350,84 bytes/sec
total size is 4.822.291.351  speedup is 1,00

root@linux-lvm:~# for i in /proc/ /sys/ /dev/ /boot/ /run/; do mount --bind $i /mnt/$i; done

root@linux-lvm:~# chroot /mnt/

root@linux-lvm:/# grub-mkconfig -o /boot/grub/grub.cfg 
Sourcing file `/etc/default/grub'
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-6.8.0-147-generic
Found initrd image: /boot/initrd.img-6.8.0-147-generic
Warning: os-prober will not be executed to detect other bootable partitions.
Systems on them will not be added to the GRUB boot configuration.
Check GRUB_DISABLE_OS_PROBER documentation entry.
Adding boot menu entry for UEFI Firmware Settings ...
done

root@linux-lvm:/# update-initramfs -u
update-initramfs: Generating /boot/initrd.img-6.8.0-147-generic
W: Couldn't identify type of root file system for fsck hook
```

### 2. Выделить том под /var - сделать в mirror

Перед перезагрузкой выделю раздел **/var** в зеркало.
Для этого задействую два диска по 1Gb:
```
root@linux-lvm:/# pvcreate /dev/sd{d,e}
  Physical volume "/dev/sdd" successfully created.
  Physical volume "/dev/sde" successfully created.

root@linux-lvm:/# vgcreate vg_var /dev/sd{d,e}
  Volume group "vg_var" successfully created

root@linux-lvm:/# lvcreate -L 950M -m1 -n lv_var lv_var
  Volume group "lv_var" not found
  Cannot process volume group lv_var

root@linux-lvm:/# lvcreate -L 950M -m1 -n lv_var vg_var
  Rounding up size to full physical extent 952,00 MiB
  Logical volume "lv_var" created.

root@linux-lvm:/# mkfs.ext4 /dev/vg_var/lv_var 
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 243712 4k blocks and 60928 inodes
Filesystem UUID: 450ee7b6-f288-4355-9b15-3e83e631028f
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

root@linux-lvm:/# mount /dev/vg_var/lv_var /mnt
```

Копирую содержимое **/var** в **/mnt**
```
root@linux-lvm:/# cp -aR /var/* /mnt/
```

Сохраню содержимое старого каталога **/var**
```
root@linux-lvm:/# mkdir /tmp/oldvar && mv /var/* /tmp/oldvar
```

Монтирую новый каталог в каталог **/var**
```
root@linux-lvm:/# umount /mnt/

root@linux-lvm:/# mount /dev/vg_var/lv_var /var

root@linux-lvm:~# echo "/dev/mapper/vg_var-lv_var  /var  ext4  defaults  0  2" >> /etc/fstab
```

Перезагружаю ВМ и удаляю временную VG
```
root@linux-lvm:~# lvremove /dev/vg_tmproot/lv_tmproot 
Do you really want to remove and DISCARD active logical volume vg_tmproot/lv_tmproot? [y/n]: y
  Logical volume "lv_tmproot" successfully removed.

root@linux-lvm:~# vgremove /dev/vg_tmproot
  Volume group "vg_tmproot" successfully removed
```

### 3. Выделить том под /home.

Выделяю том для **/home** в размере 2Gb, добавляю ФС, монтирую в **/mnt**, копирую содержимое **/home > /mnt**, перемонтирую **/mnt** в **/home** и вношу правки в **/eth/fstab** для автомантирования нового тома **/home**:
```
root@linux-lvm:~# lvcreate -n lv_home -L 2G /dev/ubuntu-vg
  Logical volume "lv_home" created.

root@linux-lvm:~# mkfs.ext4 /dev/ubuntu-vg/lv_home 
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 524288 4k blocks and 131072 inodes
Filesystem UUID: 94010e95-dcd5-4c4d-9fe5-c66af0a00491
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done 

root@linux-lvm:~# mount /dev/ubuntu-vg/lv_home /mnt/

root@linux-lvm:~# cp -aR /home/* /mnt/

root@linux-lvm:~# rm -rf /home/*

root@linux-lvm:~# umount /mnt 

root@linux-lvm:~# mount /dev/ubuntu-vg/lv_home /home/

root@linux-lvm:~# echo "/dev/mapper/ubuntu--vg-lv_home  /home  ext4  defaults  0  2" >> /etc/fstab
```

### 4. Работа со снапшотами

Генерирую файлы в **/home**:
```
root@linux-lvm:~# touch /home/file{1..20}
```

```
root@linux-lvm:~# ll /home/
total 28
drwxr-xr-x  4 root   root    4096 окт 18 18:10 ./
drwxr-xr-x 23 root   root    4096 окт 18 11:14 ../
-rw-r--r--  1 root   root       0 окт 18 18:10 file1
-rw-r--r--  1 root   root       0 окт 18 18:10 file10
-rw-r--r--  1 root   root       0 окт 18 18:10 file11
-rw-r--r--  1 root   root       0 окт 18 18:10 file12
-rw-r--r--  1 root   root       0 окт 18 18:10 file13
-rw-r--r--  1 root   root       0 окт 18 18:10 file14
-rw-r--r--  1 root   root       0 окт 18 18:10 file15
-rw-r--r--  1 root   root       0 окт 18 18:10 file16
-rw-r--r--  1 root   root       0 окт 18 18:10 file17
-rw-r--r--  1 root   root       0 окт 18 18:10 file18
-rw-r--r--  1 root   root       0 окт 18 18:10 file19
-rw-r--r--  1 root   root       0 окт 18 18:10 file2
-rw-r--r--  1 root   root       0 окт 18 18:10 file20
-rw-r--r--  1 root   root       0 окт 18 18:10 file3
-rw-r--r--  1 root   root       0 окт 18 18:10 file4
-rw-r--r--  1 root   root       0 окт 18 18:10 file5
-rw-r--r--  1 root   root       0 окт 18 18:10 file6
-rw-r--r--  1 root   root       0 окт 18 18:10 file7
-rw-r--r--  1 root   root       0 окт 18 18:10 file8
-rw-r--r--  1 root   root       0 окт 18 18:10 file9
```

Снимаю снапшот:
```
root@linux-lvm:~# lvcreate -L 100M -s -n home_snap /dev/ubuntu-vg/lv_home
  Logical volume "home_snap" created.
```

Удалю несколько файлов:
```
root@linux-lvm:~# rm -f /home/file{11..20}

root@linux-lvm:~# ll /home/
total 28
drwxr-xr-x  4 root   root    4096 окт 18 18:14 ./
drwxr-xr-x 23 root   root    4096 окт 18 11:14 ../
-rw-r--r--  1 root   root       0 окт 18 18:10 file1
-rw-r--r--  1 root   root       0 окт 18 18:10 file10
-rw-r--r--  1 root   root       0 окт 18 18:10 file2
-rw-r--r--  1 root   root       0 окт 18 18:10 file3
-rw-r--r--  1 root   root       0 окт 18 18:10 file4
-rw-r--r--  1 root   root       0 окт 18 18:10 file5
-rw-r--r--  1 root   root       0 окт 18 18:10 file6
-rw-r--r--  1 root   root       0 окт 18 18:10 file7
-rw-r--r--  1 root   root       0 окт 18 18:10 file8
-rw-r--r--  1 root   root       0 окт 18 18:10 file9
```

Восстановлю удалённые файлы из снапшота:
```
root@linux-lvm:~# umount /home 

root@linux-lvm:~# lvconvert --merge /dev/ubuntu-vg/home_snap 
  Merging of volume ubuntu-vg/home_snap started.
  ubuntu-vg/lv_home: Merged: 100,00%

root@linux-lvm:~# mount /dev/ubuntu-vg/lv_home /home/
mount: (hint) your fstab has been modified, but systemd still uses
       the old version; use 'systemctl daemon-reload' to reload.

root@linux-lvm:~# ls -al /home
total 28
drwxr-xr-x  4 root   root    4096 окт 18 18:10 .
drwxr-xr-x 23 root   root    4096 окт 18 11:14 ..
-rw-r--r--  1 root   root       0 окт 18 18:10 file1
-rw-r--r--  1 root   root       0 окт 18 18:10 file10
-rw-r--r--  1 root   root       0 окт 18 18:10 file11
-rw-r--r--  1 root   root       0 окт 18 18:10 file12
-rw-r--r--  1 root   root       0 окт 18 18:10 file13
-rw-r--r--  1 root   root       0 окт 18 18:10 file14
-rw-r--r--  1 root   root       0 окт 18 18:10 file15
-rw-r--r--  1 root   root       0 окт 18 18:10 file16
-rw-r--r--  1 root   root       0 окт 18 18:10 file17
-rw-r--r--  1 root   root       0 окт 18 18:10 file18
-rw-r--r--  1 root   root       0 окт 18 18:10 file19
-rw-r--r--  1 root   root       0 окт 18 18:10 file2
-rw-r--r--  1 root   root       0 окт 18 18:10 file20
-rw-r--r--  1 root   root       0 окт 18 18:10 file3
-rw-r--r--  1 root   root       0 окт 18 18:10 file4
-rw-r--r--  1 root   root       0 окт 18 18:10 file5
-rw-r--r--  1 root   root       0 окт 18 18:10 file6
-rw-r--r--  1 root   root       0 окт 18 18:10 file7
-rw-r--r--  1 root   root       0 окт 18 18:10 file8
-rw-r--r--  1 root   root       0 окт 18 18:10 file9
```
