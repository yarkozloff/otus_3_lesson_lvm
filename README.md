# Первые попытки
[Этап выполнения ДЗ](https://github.com/yarkozloff/otus_3_lesson_lvm/edit/main/README.md#%D0%B4%D0%BE%D0%BC%D0%B0%D1%88%D0%BD%D0%B5%D0%B5-%D0%B7%D0%B0%D0%B4%D0%B0%D0%BD%D0%B8%D0%B5)
## Подготовка машины
В Vagrantfile добавлены 2 диска (будущие физические тома) по 100мб
```
:disks => {
                 :sata1 => {
                         :dfile => 'sata1.vdi',
                         :size => 100,
                         :port => 1
                 },
                 :sata2 => {
                         :dfile => 'sata2.vdi',
                         :size => 100, # Megabytes
                         :port => 2
                 }
 
         }
```
Выполнены необходимые для network настройки
```
:ip_addr => '192.168.237.136',
  if boxconfig.key?(:forwarded_port)
              boxconfig[:forwarded_port].each do |port|
              box.vm.network "forwarded_port", port
              end
           end
           # Additional network config if present
           if boxconfig.key?(:net)
              boxconfig[:net].each do |ipconf|
              box.vm.network "private_network", ipconf
              end
           end
```
## Запуск машины и обновление приложений:
```
sudo vagrant up
sudo vagrant ssh
sudo yum update
```
Имеем следующую ситуацию по устройствам (lsblk):
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   40G  0 disk
L-sda1   8:1    0   40G  0 part /
sdb      8:16   0  100M  0 disk
sdc      8:32   0  100M  0 disk

## Подготовка LVM
Разметим диск для будущего использования LVM - создадим PV:
```
sudo pvcreate /dev/sdb
```
Ошибка pvcreate: command not found. Нужно поставить lvm2:
```
sudo yum install lvm2
```
Еще раз размечаем диск. Успех
>   Physical volume "/dev/sdb" successfully created.
Затем можно создавать первый уровень абстракции - VG:
```
sudo vgcreate otus /dev/sdb
```
>   Volume group "otus" successfully created
Создаем Logical Volume:
```
 lvcreate -l+80%FREE -n test otus
```
Передумал с названием. Отключил группу томов и удалил физические тома для vg:
```
sudo vgremove otus
sudo pvremove /dev/sdb
```
Создал заново. Посмотрел инфо по lsblk:
NAME       MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda          8:0    0   40G  0 disk
└─sda1       8:1    0   40G  0 part /
sdb          8:16   0  100M  0 disk
└─vgb-test 253:0    0   76M  0 lvm
sdc          8:32   0  100M  0 disk
Посмотрел инфо по Volume Group (vgdisplay):
  --- Volume group ---
  VG Name               vgb
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
  VG Size               96.00 MiB
  PE Size               4.00 MiB
  Total PE              24
  Alloc PE / Size       19 / 76.00 MiB
  Free  PE / Size       5 / 20.00 MiB
  VG UUID               s6jrTs-vt7C-mZbr-QWDf-H1uB-WLo3-AlzyKn
Создаем еще один один LV из свободного места абсолютным значением в МБ:
```
sudo lvcreate -L20M -n small vgb
sudo lvs
```
LV    VG  Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  small vgb -wi-a----- 20.00m
  test  vgb -wi-a----- 76.00m
Создадим на LV файловую систему и смонтируем его:
```
sudo mkfs.ext4 /dev/vgb/test
sudo mkdir /data
sudo mount /dev/vgb/test /data
```
Смотрим lsblk:
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda           8:0    0   40G  0 disk
└─sda1        8:1    0   40G  0 part /
sdb           8:16   0  100M  0 disk
├─vgb-test  253:0    0   76M  0 lvm  /data
└─vgb-small 253:1    0   20M  0 lvm
sdc           8:32   0  100M  0 disk

## Расширение LVM
Создаем PV и добавляем в VG новый диск:
```
sudo pvcreate /dev/sdc
sudo vgextend vgb /dev/sdc
sudo vgdisplay -v vgb | grep 'PV Name'
```
PV Name               /dev/sdb
  PV Name               /dev/sdc
С помощью команды dd (дубликатор данных), заполним место в /data до 100%
```
sudo dd if=/dev/zero of=/data/test.log bs=1M count=8000 status=progress
sudo df -Th /data/
```
Filesystem           Type  Size  Used Avail Use% Mounted on
/dev/mapper/vgb-test ext4   70M   69M     0 100% /data
Увеличиваем LV за счет появившегося свободного места. Смотри на сколько расширен:
```
sudo lvextend -l+70%FREE /dev/vgb/test
sudo lvs /dev/vgb/test
```
LV   VG  Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  test vgb -wi-ao---- 144.00m
Производим resize файловой системы и смотрим место:
```
sudo resize2fs /dev/vgb/test
sudo df -Th /data
```
Filesystem           Type  Size  Used Avail Use% Mounted on
/dev/mapper/vgb-test ext4  136M   69M   60M  54% /data

## Уменьшение LVM
Отмонтирование файловой системы, проверка на ошибки, уменьшение размера, примонтирование фс обратно:
```
sudo umount /data/
sudo e2fsck -fy /dev/vgb/test
```
e2fsck 1.42.9 (28-Dec-2013)
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
/dev/vgb/test: 12/35136 files (8.3% non-contiguous), 78233/147456 blocks
```
sudo resize2fs /dev/vgb/test 100M
```
resize2fs 1.42.9 (28-Dec-2013)
Resizing the filesystem on /dev/vgb/test to 102400 (1k) blocks.
The filesystem on /dev/vgb/test is now 102400 blocks long.
```
sudo lvreduce /dev/vgb/test -L 100M
```  
  WARNING: Reducing active logical volume to 100.00 MiB.
  THIS MAY DESTROY YOUR DATA (filesystem etc.)
Do you really want to reduce vgb/test? [y/n]: y
  Size of logical volume vgb/test changed from 144.00 MiB (36 extents) to 100.00 MiB (25 extents).
  Logical volume vgb/test successfully resized.
```
sudo mount /dev/vgb/test /data/
sudo df -Th /data
```
Filesystem           Type  Size  Used Avail Use% Mounted on
/dev/mapper/vgb-test ext4   93M   69M   19M  79% /data

## LVM Snapshot
С флагом -s создаем снапшот:
```
sudo lvcreate -L 10M -s -n test-snap /dev/vgb/test
lsblk
```
NAME                 MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                    8:0    0   40G  0 disk
└─sda1                 8:1    0   40G  0 part /
sdb                    8:16   0  100M  0 disk
├─vgb-small          253:1    0   20M  0 lvm
└─vgb-test-real      253:2    0  100M  0 lvm
  ├─vgb-test         253:0    0  100M  0 lvm  /data
  └─vgb-test--snap   253:4    0  100M  0 lvm
sdc                    8:32   0  100M  0 disk
├─vgb-test-real      253:2    0  100M  0 lvm
│ ├─vgb-test         253:0    0  100M  0 lvm  /data
│ └─vgb-test--snap   253:4    0  100M  0 lvm
└─vgb-test--snap-cow 253:3    0   12M  0 lvm
  └─vgb-test--snap   253:4    0  100M  0 lvm
Монтируем, смотри результат:
```
sudo mkdir /data-snap
sudo mount /dev/vgb/test-snap /data-snap/
ll /data-snap
```
total 68158
drwx------. 2 root root    12288 Feb 21 22:18 lost+found
-rw-r--r--. 1 root root 69779456 Feb 21 22:30 test.log

# Домашнее задание
02.03.2022 HashiCorp всё. VagrantCloud для РФ нет: This content is not currently available in your region.
Бокс для вагранта подгрузил локально:
```
sudo vagrant box add centos7 CentOS-7-x86_64-Vagrant-1804_02.VirtualBox.box
sudo vagrant init centos7
```
Подредактировал Vagrantfile под новый бокс, запустился, выполнил update, установил xfsdump
## Уменьшить том под / до 8G
Подготовим временный том для / раздела:
```
[vagrant@lvm ~]$ sudo pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created.
[vagrant@lvm ~]$ sudo vgcreate vg_root /dev/sdb
  Volume group "vg_root" successfully created
[vagrant@lvm ~]$ sudo lvcreate -n lv_root -l +100%FREE /dev/vg_root
  Logical volume "lv_root" created.
```
Создадим на нем файловую систему и смонтируем его, чтобы перенести туда данные:
```
[vagrant@lvm ~]$ sudo mkfs.xfs /dev/vg_root/lv_root
meta-data=/dev/vg_root/lv_root   isize=512    agcount=4, agsize=655104 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2620416, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
[vagrant@lvm ~]$ sudo mount /dev/vg_root/lv_root /mnt
```
Cкопируем все данные с / раздела в /mnt:
```
[vagrant@lvm ~]$ sudo xfsdump -J - /dev/VolGroup00/LogVol00 | sudo xfsrestore -J - /mnt
xfsdump: using file dump (drive_simple) strategy
xfsdump: version 3.1.7 (dump format 3.0)
xfsrestore: using file dump (drive_simple) strategy
xfsrestore: version 3.1.7 (dump format 3.0)
xfsdump: level 0 dump of lvm:/
xfsdump: dump date: Wed Mar  2 23:32:31 2022
xfsdump: session id: 01901006-2623-4f7c-ac90-d2b06536173b
xfsdump: session label: ""
xfsrestore: searching media for dump
xfsdump: ino map phase 1: constructing initial dump list
xfsdump: ino map phase 2: skipping (no pruning necessary)
xfsdump: ino map phase 3: skipping (only one dump stream)
xfsdump: ino map construction complete
xfsdump: estimated dump size: 1762501632 bytes
xfsdump: creating dump session media file 0 (media 0, file 0)
xfsdump: dumping ino map
xfsdump: dumping directories
xfsrestore: examining media file 0
xfsrestore: dump description:
xfsrestore: hostname: lvm
xfsrestore: mount point: /
xfsrestore: volume: /dev/mapper/VolGroup00-LogVol00
xfsrestore: session time: Wed Mar  2 23:32:31 2022
xfsrestore: level: 0
xfsrestore: session label: ""
xfsrestore: media label: ""
xfsrestore: file system id: b60e9498-0baa-4d9f-90aa-069048217fee
xfsrestore: session id: 01901006-2623-4f7c-ac90-d2b06536173b
xfsrestore: media id: dfdcc710-fc12-4ba4-9134-7481941c3b0c
xfsrestore: searching media for directory dump
xfsrestore: reading directories
xfsdump: dumping non-directory files
xfsrestore: 4321 directories and 35188 entries processed
xfsrestore: directory post-processing
xfsrestore: restoring non-directory files
xfsdump: ending media file
xfsdump: media file size 1726625520 bytes
xfsdump: dump size (non-dir files) : 1706217704 bytes
xfsdump: dump complete: 322 seconds elapsed
xfsdump: Dump Status: SUCCESS
xfsrestore: restore complete: 322 seconds elapsed
xfsrestore: Restore Status: SUCCESS
[vagrant@lvm mnt]$  ls /mnt
bin   dev  home  lib64  mnt  proc  run   srv  tmp  vagrant
boot  etc  lib   media  opt  root  sbin  sys  usr  var
```
Сымитируем текущий root -> сделаем в него chroot (операция изменения корневого каталога диска) и обновим grub:
```
[vagrant@lvm mnt]$ for i in /proc/ /sys/ /dev/ /run/ /boot/; do sudo mount --bind $i /mnt/$i; done
[vagrant@lvm mnt]$ sudo chroot /mnt/
[root@lvm /]# grub2-mkconfig -o /boot/grub2/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-1160.59.1.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-1160.59.1.el7.x86_64.img
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
done
```
Обновим образ initrd (используется для начальной инициализации перед монтированием «настоящих» файловых систем)
```
[root@lvm /]# cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done
Executing: /sbin/dracut -v initramfs-3.10.0-1160.59.1.el7.x86_64.img 3.10.0-1160.59.1.el7.x86_64 --force
...
*** Creating image file ***
*** Creating image file done ***
*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***
```
В файле /boot/grub2/grub.cfg (в двух местах)
заменить rd.lvm.lv=VolGroup00/LogVol00 
на rd.lvm.lv=vg_root/lv_root
reboot
```
[vagrant@lvm ~]$ lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk
├─sda1                    8:1    0    1M  0 part
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part
  ├─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
  └─VolGroup00-LogVol00 253:2    0 37.5G  0 lvm
sdb                       8:16   0   10G  0 disk
└─vg_root-lv_root       253:0    0   10G  0 lvm  /
sdc                       8:32   0    2G  0 disk
sdd                       8:48   0    1G  0 disk
sde                       8:64   0    1G  0 disk
```
Теперь меняем размер старой VG и возвращаем на него рут. 
Удаляем старый LV размером в 40G и создаем новый на 8G:
```
[vagrant@lvm ~]$ sudo  lvremove /dev/VolGroup00/LogVol00
Do you really want to remove active logical volume VolGroup00/LogVol00? [y/n]: y
  Logical volume "LogVol00" successfully removed
[vagrant@lvm ~]$ sudo lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00
WARNING: xfs signature detected on /dev/VolGroup00/LogVol00 at offset 0. Wipe it? [y/n]: y
  Wiping xfs signature on /dev/VolGroup00/LogVol00.
  Logical volume "LogVol00" created.
```
Проделываем те же операции что и в первый раз:
```
[vagrant@lvm ~]$ sudo mkfs.xfs /dev/VolGroup00/LogVol00
meta-data=/dev/VolGroup00/LogVol00 isize=512    agcount=4, agsize=524288 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2097152, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
[vagrant@lvm ~]$ sudo mount /dev/VolGroup00/LogVol00 /mnt
[vagrant@lvm ~]$ sudo xfsdump -J - /dev/vg_root/lv_root | sudo xfsrestore -J - /mnt
xfsrestore: using file dump (drive_simple) strategy
...
xfsdump: Dump Status: SUCCESS
xfsrestore: restore complete: 91 seconds elapsed
xfsrestore: Restore Status: SUCCESS
```
Так же как в первый раз переконфигурируем grub, за исключением правки /etc/grub2/grub.cfg
```
[vagrant@lvm ~]$  for i in /proc/ /sys/ /dev/ /run/ /boot/; do sudo mount --bind $i /mnt/$i; done
[vagrant@lvm ~]$ sudo chroot /mnt/
[root@lvm /]# grub2-mkconfig -o /boot/grub2/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-1160.59.1.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-1160.59.1.el7.x86_64.img
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
done
[root@lvm /]# cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g;
> s/.img//g"` --force; done
Executing: /sbin/dracut -v initramfs-3.10.0-1160.59.1.el7.x86_64.img 3.10.0-1160.59.1.el7.x86_64 --force
...
*** Creating image file ***
*** Creating image file done ***
*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***
```
Диск уменьшился до 8ГБ :
```
[root@lvm boot]# lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk
├─sda1                    8:1    0    1M  0 part
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part
  ├─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
  └─VolGroup00-LogVol00 253:2    0    8G  0 lvm  /
sdb                       8:16   0   10G  0 disk
└─vg_root-lv_root       253:0    0   10G  0 lvm
sdc                       8:32   0    2G  0 disk
sdd                       8:48   0    1G  0 disk
sde                       8:64   0    1G  0 disk
```
## Выделить том под /var в зеркало
На свободных дисках создаем зеркало:
```
[root@lvm boot]# pvcreate /dev/sdc /dev/sdd
  Physical volume "/dev/sdc" successfully created.
  Physical volume "/dev/sdd" successfully created.
[root@lvm boot]# vgcreate vg_var /dev/sdc /dev/sdd
  Volume group "vg_var" successfully created
[root@lvm boot]# lvcreate -L 950M -m1 -n lv_var vg_var
  Rounding up size to full physical extent 952.00 MiB
  Logical volume "lv_var" created.
```
Создаем на нем ФС и перемещаем туда /var, сохраняем содержимое старого var, монтируем новый var в каталог /var, Правим fstab для автоматического монтирования /var:
```
[root@lvm boot]# vgcreate vg_var /dev/sdc /dev/sdd
  Volume group "vg_var" successfully created
[root@lvm boot]# lvcreate -L 950M -m1 -n lv_var vg_var
  Rounding up size to full physical extent 952.00 MiB
  Logical volume "lv_var" created.
[root@lvm boot]# mkfs.ext4 /dev/vg_var/lv_var
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
[root@lvm boot]# mount /dev/vg_var/lv_var /mnt
[root@lvm boot]# cp -aR /var/* /mnt/ # rsync -avHPSAX /var/ /mnt/
[root@lvm boot]# mkdir /tmp/oldvar && mv /var/* /tmp/oldvar
[root@lvm boot]#  umount /mnt
[root@lvm boot]# mount /dev/vg_var/lv_var /var
[root@lvm boot]# echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab
```
reboot, удаление новой Volume Group:
```
[vagrant@lvm ~]$ sudo lvremove /dev/vg_root/lv_root
Do you really want to remove active logical volume vg_root/lv_root? [y/n]: y
  Logical volume "lv_root" successfully removed
[vagrant@lvm ~]$ sudo vgremove /dev/vg_root
  Volume group "vg_root" successfully removed
[vagrant@lvm ~]$ sudo pvremove /dev/sdb
  Labels on physical volume "/dev/sdb" successfully wiped.
[vagrant@lvm ~]$ lsblk
NAME                     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                        8:0    0   40G  0 disk
├─sda1                     8:1    0    1M  0 part
├─sda2                     8:2    0    1G  0 part /boot
└─sda3                     8:3    0   39G  0 part
  ├─VolGroup00-LogVol00  253:0    0    8G  0 lvm  /
  └─VolGroup00-LogVol01  253:1    0  1.5G  0 lvm  [SWAP]
sdb                        8:16   0   10G  0 disk
sdc                        8:32   0    2G  0 disk
├─vg_var-lv_var_rmeta_0  253:3    0    4M  0 lvm
│ └─vg_var-lv_var        253:7    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_0 253:4    0  952M  0 lvm
  └─vg_var-lv_var        253:7    0  952M  0 lvm  /var
sdd                        8:48   0    1G  0 disk
├─vg_var-lv_var_rmeta_1  253:5    0    4M  0 lvm
│ └─vg_var-lv_var        253:7    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_1 253:6    0  952M  0 lvm
  └─vg_var-lv_var        253:7    0  952M  0 lvm  /var
sde                        8:64   0    1G  0 disk
```
## Выделить том под /home
Выделяем том под /home по тому же принципу что делали для /var:
```
[vagrant@lvm ~]$ sudo lvcreate -n LogVol_Home -L 2G /dev/VolGroup00
  Logical volume "LogVol_Home" created.
[vagrant@lvm ~]$ sudo mkfs.xfs /dev/VolGroup00/LogVol_Home
meta-data=/dev/VolGroup00/LogVol_Home isize=512    agcount=4, agsize=131072 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=524288, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
[vagrant@lvm ~]$ sudo mount /dev/VolGroup00/LogVol_Home /mnt/
[vagrant@lvm ~]$ sudo cp -aR /home/* /mnt/
[vagrant@lvm ~]$ sudo rm -rf /home/*
[vagrant@lvm ~]$ sudo umount /mnt
[vagrant@lvm ~]$ sudo mount /dev/VolGroup00/LogVol_Home /home/
```
(Переключился на root для прав записи в fstab)
```
[root@lvm ~]# echo "`blkid | grep Home | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab
```
Том под home выделен:
```
[root@lvm ~]# lsblk
NAME                       MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                          8:0    0   40G  0 disk
├─sda1                       8:1    0    1M  0 part
├─sda2                       8:2    0    1G  0 part /boot
└─sda3                       8:3    0   39G  0 part
  ├─VolGroup00-LogVol00    253:0    0    8G  0 lvm  /
  ├─VolGroup00-LogVol01    253:1    0  1.5G  0 lvm  [SWAP]
  └─VolGroup00-LogVol_Home 253:2    0    2G  0 lvm  /home
sdb                          8:16   0   10G  0 disk
sdc                          8:32   0    2G  0 disk
├─vg_var-lv_var_rmeta_0    253:3    0    4M  0 lvm
│ └─vg_var-lv_var          253:7    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_0   253:4    0  952M  0 lvm
  └─vg_var-lv_var          253:7    0  952M  0 lvm  /var
sdd                          8:48   0    1G  0 disk
├─vg_var-lv_var_rmeta_1    253:5    0    4M  0 lvm
│ └─vg_var-lv_var          253:7    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_1   253:6    0  952M  0 lvm
  └─vg_var-lv_var          253:7    0  952M  0 lvm  /var
sde                          8:64   0    1G  0 disk
```
## /home - сделать том для снапшотов
Сгенерируем файлы в /home/:
```
[root@lvm ~]# touch /home/file{1..20}
```
Снять снапшот:
```
[root@lvm ~]#  lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home
  Rounding up size to full physical extent 128.00 MiB
  Logical volume "home_snap" created.
[root@lvm ~]# lsblk
NAME                            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                               8:0    0   40G  0 disk
├─sda1                            8:1    0    1M  0 part
├─sda2                            8:2    0    1G  0 part /boot
└─sda3                            8:3    0   39G  0 part
  ├─VolGroup00-LogVol00         253:0    0    8G  0 lvm  /
  ├─VolGroup00-LogVol01         253:1    0  1.5G  0 lvm  [SWAP]
  ├─VolGroup00-LogVol_Home-real 253:8    0    2G  0 lvm
  │ ├─VolGroup00-LogVol_Home    253:2    0    2G  0 lvm  /home
  │ └─VolGroup00-home_snap      253:10   0    2G  0 lvm
  └─VolGroup00-home_snap-cow    253:9    0  128M  0 lvm
    └─VolGroup00-home_snap      253:10   0    2G  0 lvm
```
Удалить часть файлов:
```
[root@lvm ~]# rm -f /home/file{11..20}
```
Процесс восстановления со снапшота:
```
[root@lvm ~]# umount /home
[root@lvm ~]# lvconvert --merge /dev/VolGroup00/home_snap
  Merging of volume VolGroup00/home_snap started.
  VolGroup00/LogVol_Home: Merged: 100.00%
[root@lvm ~]# mount /home
[root@lvm ~]# lsblk
NAME                       MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                          8:0    0   40G  0 disk
├─sda1                       8:1    0    1M  0 part
├─sda2                       8:2    0    1G  0 part /boot
└─sda3                       8:3    0   39G  0 part
  ├─VolGroup00-LogVol00    253:0    0    8G  0 lvm  /
  ├─VolGroup00-LogVol01    253:1    0  1.5G  0 lvm  [SWAP]
  └─VolGroup00-LogVol_Home 253:2    0    2G  0 lvm  /home
sdb                          8:16   0   10G  0 disk
sdc                          8:32   0    2G  0 disk
├─vg_var-lv_var_rmeta_0    253:3    0    4M  0 lvm
│ └─vg_var-lv_var          253:7    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_0   253:4    0  952M  0 lvm
  └─vg_var-lv_var          253:7    0  952M  0 lvm  /var
sdd                          8:48   0    1G  0 disk
├─vg_var-lv_var_rmeta_1    253:5    0    4M  0 lvm
│ └─vg_var-lv_var          253:7    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_1   253:6    0  952M  0 lvm
  └─vg_var-lv_var          253:7    0  952M  0 lvm  /var
sde                          8:64   0    1G  0 disk
```
