## Подготовка машины для выполнения ДЗ
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

... ДЗ


