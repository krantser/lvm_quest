## **Уменьшаем том LVM**

Исходная структура файловой системы:
```
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
Разметим диск `/dev/sdb/`, на который будет помещёна временная корневая 
директория:
```
sudo fdisk /dev/sdb

Command (m for help): n
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended
Select (default p): 
Using default response p
Partition number (1-4, default 1): 
First sector (2048-20971519, default 2048): 
Using default value 2048
Last sector, +sectors or +size{K,M,G} (2048-20971519, default 20971519): 
Using default value 20971519
Partition 1 of type Linux and of size 10 GiB is set

Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.
```

Далее создадим файловую систему формата xfs на получившемся разделе:
```
sudo mkfs.xfs /dev/sdb1
```

Создадим директорию куда будем монтировать устройство для временного корневого
раздела:
```
sudo mkdir /mnt/back_root
```

Монтируем подготовленный диск в созданную точку монтирования `/mnt/back_root/`:
```
sudo mount /dev/sdb1 /mnt/back_root/
```

Теперь скопируем полностью корневую директорию во временную папку, куда
примонтирован наш новый подготовленный диск. Делается это с помощью утилиты `cp`
с параметрами -ax, где -a (--archive) указывает по возможности сохранять 
структуру и атрибуты исходных файлов при копировании, и -x (--one-file-system)
указывает пропускать подкаталоги, которые расположены на файловых системах, 
отличных от той, где начиналось копирование:
```
sudo cp -ax / /mnt/back_root/
```
Просмотрим статистику файловой системы:
```
df -h
```
Вывод команды:
```
Filesystem                       Size  Used Avail Use% Mounted on
/dev/mapper/VolGroup00-LogVol00   38G  1.7G   36G   5% /
devtmpfs                         110M     0  110M   0% /dev
tmpfs                            118M     0  118M   0% /dev/shm
tmpfs                            118M  4.6M  114M   4% /run
tmpfs                            118M     0  118M   0% /sys/fs/cgroup
/dev/sda2                       1014M   61M  954M   7% /boot
tmpfs                             24M     0   24M   0% /run/user/1000
/dev/sdb1                         10G  1.7G  8.4G  17% /mnt/back_root
```

Далее перейдём во временную директорию с корневой файловой системой:
```
cd /mnt/back_root/
```

Выполним монитрование базовых файловых систем:
```
[vagrant@lvm back_root]$ sudo mount -o bind /dev /mnt/back_root/dev/
[vagrant@lvm back_root]$ sudo mount -t proc none /mnt/back_root/proc/
[vagrant@lvm back_root]$ sudo mount -t sysfs none /mnt/back_root/sys/
```
Скопируем директорию с конфигурацией загрузки и загрузчиком системы:
```
sudo cp -ax /boot/ /mnt/back_root/
```

Теперь сгенерируем новую конфигурацию для загрузчика grub2:
```
sudo grub2-mkconfig -o /mnt/back_root/boot/grub2/grub.cfg
```

Вывод команды:
```
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
Found CentOS Linux release 7.8.2003 (Core) on /dev/sdb1
done
```

Далее необходимо отредактировать файл fstab, убрав настройку монтирования 
текущего логического тома, который мы хотим уменьшить, и указать монтирование
нового диска в корень:
``` 
sudo vi /etc/fstab
```

Приведём файл к виду:
```
#
# /etc/fstab
# Created by anaconda on Sat May 12 18:50:26 2018
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
#/dev/mapper/VolGroup00-LogVol00 /                   xfs     defaults        0 0
/dev/sdb1    /                                       xfs     defaults        0 0
UUID=570897ca-e759-4c81-90cf-389da6eee4cc /boot      xfs     defaults        0 0
/dev/mapper/VolGroup00-LogVol01 swap                 swap    defaults        0 0
```

Сделаем резервную копию старой конфигурации и скопируем новую в основную 
загрочную директорию:
```
sudo mv /boot/grub2/grub.cfg /boot/grub2/grub.cfg.backup
sudo cp /mnt/back_root/boot/grub2/grub.cfg /boot/grub2/
```

Теперь сгенерируем новый initramfs образ, необходимый при загрузке операционной
системы Linux, для этого используем низкоуровневую утилиту gracut, которая и
предназначена для этих целей. Просмотрим текущие параметры загрузки:
```
sudo dracut --print-cmdline
```

Вывод команды:
```
 rd.lvm.lv=VolGroup00/LogVol01 
 rd.lvm.lv=VolGroup00/LogVol00 
 root=/dev/mapper/VolGroup00-LogVol00 rootflags=rw,relatime,seclabel,attr2,inode64,noquota rootfstype=xfs
```

Сгенерируем новый образ, указа dracut два флага --fstab (указывает использовать 
при создании образа данные о монтировании, взятые из /etc/fstab) и --force 
(указывает перезаписать текущий initramfs файл):
```
sudo dracut --fstab --force
```

Теперь перезагрузим систему:
```
sudo reboot
```

В графическом окне виртуальной машины при запуске машины в загрузочном меню
выбираем пункт:
>>>> CentOS Linux release 7.8.2003 (Core) (on /dev/sdb1) <<<<
На данном этапе мы загружаемся с нового временного раздела.

После перезагрузки ещё раз выведем параметры ядра для запуска системы:
```
sudo dracut --print-cmdline 
```

Вывод команды:
```
 rd.lvm.lv=VolGroup00/LogVol01 
 root=UUID=cd5e6bf7-3ff9-4ae2-9f57-aa163deba9f5 rootflags=rw,relatime,seclabel,attr2,inode64,noquota rootfstype=xfs
```

Просмотрим статистику и структуру нашей фаловой системы и устройств:
```
df -h
```

Вывод команды:
```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdb1        10G  1.7G  8.4G  17% /
devtmpfs        110M     0  110M   0% /dev
tmpfs           118M     0  118M   0% /dev/shm
tmpfs           118M  4.6M  114M   4% /run
tmpfs           118M     0  118M   0% /sys/fs/cgroup
/dev/sda2      1014M   62M  953M   7% /boot
tmpfs            24M     0   24M   0% /run/user/1000
```

```
lsblk
```

Вывод команды:
``` 
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk 
├─sda1                    8:1    0    1M  0 part 
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00 253:0    0 37.5G  0 lvm
  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk 
└─sdb1                    8:17   0   10G  0 part /
sdc                       8:32   0    2G  0 disk 
sdd                       8:48   0    1G  0 disk 
sde                       8:64   0    1G  0 disk 
```

Теперь перейдём к изменению размера lmv-тома. Для этого удалим существующий:
```
sudo lvremove /dev/VolGroup00/LogVol00

Do you really want to remove active logical volume VolGroup00/LogVol00? [y/n]: y
  Logical volume "LogVol00" successfully removed
```

И создадим новый том, нужного нам размера (8 Гб):
```
sudo lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00

WARNING: xfs signature detected on /dev/VolGroup00/LogVol00 at offset 0. Wipe it? [y/n]: y
  Wiping xfs signature on /dev/VolGroup00/LogVol00.
  Logical volume "LogVol00" created.
```

Теперь развернём на новом диске файловую систему (xfs):
```
sudo mkfs.xfs /dev/VolGroup00/LogVol00
```

Вывод команды:
```
meta-data=/dev/VolGroup00/LogVol00 isize=512    agcount=4, agsize=524288 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2097152, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```

Примонтируем новый том во временную директорию, что бы перенести корневую
файловую систему:
```
sudo mount /dev/VolGroup00/LogVol00 /mnt/back_root/
```

Скопируем корневую фаловую систему на новый том:
```
sudo cp --archive --one-file-system / /mnt/back_root/
```

Снова редактируем конфигурационный файл fstab и копируем конфигурацию на новый
том с корневой файловой системой:
```
sudo vi /etc/fstab
sudo cp /etc/fstab /mnt/back_root/etc/ 
```

Приводим конфигурацию к виду:
```
#
# /etc/fstab
# Created by anaconda on Sat May 12 18:50:26 2018
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/VolGroup00-LogVol00 /                         xfs    defaults    0 0
#/dev/sdb1    /                                           xfs    defaults    0 0
UUID=570897ca-e759-4c81-90cf-389da6eee4cc /boot           xfs    defaults    0 0
/dev/mapper/VolGroup00-LogVol01 swap                      swap   defaults    0 0
```

Снова генерируем новый initramfs образ:
```
sudo dracut --fstab --force
```

## **Создание тома типа mirror под директорию /var**
Теперь создаём физические тома для lvm, выберем диски sdd и sde:
```
sudo pvcreate /dev/sdd /dev/sde 
```

Вывод команды:
```
  Physical volume "/dev/sdd" successfully created.
  Physical volume "/dev/sde" successfully created.
```

Создадим шруппу томов и добавим в неё наши тома:
```
sudo vgcreate vg_var /dev/sdd /dev/sde
  Volume group "vg_var" successfully created
```

Теперь создадим логический том типа mirror с именем lv_var и добавим в группу
vg_var:
```
sudo lvcreate -L 950M -m1 -n lv_var vg_var 
```

Вывод команды:
```
  Rounding up size to full physical extent 952.00 MiB
  Logical volume "lv_var" created.
```

Создадим поверх нового тома файловую систему (ext4):
```
sudo mkfs.ext4 /dev/vg_var/lv_var 
```

Вывод команды:
```
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
```

Создадим временную директорию куда в дальнейшем примонтируем новый том, где
будет располагаться директория /var:
```
sudo mkdir /mnt/new_var
```

Монтируем новый том lv_var в созданную временную директорию:
```
sudo mount /dev/vg_var/lv_var /mnt/new_var/
```

Копируем данные из директории /var во временную директорию с помощью утилиты 
rsync:
```
sudo rsync --archive --hard-links --sparse --acls --xattrs --info=progress2 /var/* /mnt/new_var
```

Теперь отредактируем fstab, что бы указазть системе куда монитровать новый том:
```
sudo vi /etc/fstab
```

Приведём файл к следующему виду:
```
#
# /etc/fstab
# Created by anaconda on Sat May 12 18:50:26 2018
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/VolGroup00-LogVol00 /                    xfs     defaults        0 0
#/dev/sdb1    /                                      xfs     defaults        0 0
/dev/vg_var/lv_var    /var                           ext4    defaults        0 0
UUID=570897ca-e759-4c81-90cf-389da6eee4cc /boot      xfs     defaults        0 0
/dev/mapper/VolGroup00-LogVol01 swap                 swap    defaults        0 0
```

Теперь перезагрузим систему. В меню загрузки в графическом режиме теперь ничего
выбирать не нужно, система по умолчанию грузится с тома LogVol00:
```
sudo reboot
```

Просмотрим структуру получившейся файловой системы:
```
lsblk 
```

Вывод команды:
```
NAME                     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                        8:0    0   40G  0 disk 
├─sda1                     8:1    0    1M  0 part 
├─sda2                     8:2    0    1G  0 part /boot
└─sda3                     8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00  253:0    0    8G  0 lvm  /
  └─VolGroup00-LogVol01  253:1    0  1.5G  0 lvm  [SWAP]
sdb                        8:16   0   10G  0 disk 
└─sdb1                     8:17   0   10G  0 part 
sdc                        8:32   0    2G  0 disk 
sdd                        8:48   0    1G  0 disk 
├─vg_var-lv_var_rmeta_0  253:2    0    4M  0 lvm
│ └─vg_var-lv_var        253:6    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_0 253:3    0  952M  0 lvm
  └─vg_var-lv_var        253:6    0  952M  0 lvm  /var
sde                        8:64   0    1G  0 disk 
├─vg_var-lv_var_rmeta_1  253:4    0    4M  0 lvm
│ └─vg_var-lv_var        253:6    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_1 253:5    0  952M  0 lvm
  └─vg_var-lv_var        253:6    0  952M  0 lvm  /var
```

## **Выделение тома для /home и снэпшоты**
Теперь можем перейти к переносу директории /home на отдельный том. Для этого
создадим логический том lvm с именем LogVolHome размером 2 Гб и добавим его в
группу VolGroup00:
```
sudo lvcreate -n LogVolHome -L 2G /dev/VolGroup00
```

Вывод команды:
```
  Logical volume "LogVolHome" created.
```

На получившемся томе создадим файловую систему типа xfs:
```
sudo mkfs.xfs /dev/VolGroup00/LogVolHome 
```

Вывод команды:
```
meta-data=/dev/VolGroup00/LogVolHome isize=512    agcount=4, agsize=131072 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=524288, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```

Дале создадим временную директорию кудабудет смонтирован новый том для переноса
/home:
```
sudo mkdir /mnt/new_home
```

Монтируем только что созданный том во временную директорию:
```
sudo mount /dev/VolGroup00/LogVolHome /mnt/new_home/
```

Копируем содержимое /home во временную директорию куда примонтирован новый том:
```
sudo rsync --archive --hard-links --sparse --acls --xattrs --info=progress2 /home/* /mnt/new_home/
            831 100%  793.95kB/s    0:00:00 (xfr#4, to-chk=0/6)
```

Удалим содержимое в /home:
```
sudo rm -rf /home/*
```

Примонтируем новый том LogVolHome в /home:
```
sudo mount /dev/VolGroup00/LogVolHome /home/
```

Теперь отредактируем fstab  и добавим новые параметры для монитрования /home:
```
sudo vi /etc/fstab 
```

Приведём файл к виду:
```
#
# /etc/fstab
# Created by anaconda on Sat May 12 18:50:26 2018
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/VolGroup00-LogVol00 /                    xfs     defaults        0 0
/dev/vg_var/lv_var    /var                           ext4    defaults        0 0
/dev/VolGroup00/LogVolHome    /home                  xfs     defaults        0 0
#/dev/sdb1    /                                      xfs     defaults        0 0
UUID=570897ca-e759-4c81-90cf-389da6eee4cc /boot      xfs     defaults        0 0
/dev/mapper/VolGroup00-LogVol01 swap                 swap    defaults        0 0
```

Перезагрузим систему:
```
sudo reboot
```

Просмотрим структуру файловой системы:
```
lsblk
```

Вывод команды:
```
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                         8:0    0   40G  0 disk 
├─sda1                      8:1    0    1M  0 part 
├─sda2                      8:2    0    1G  0 part /boot
└─sda3                      8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00   253:0    0    8G  0 lvm  /
  ├─VolGroup00-LogVol01   253:1    0  1.5G  0 lvm  [SWAP]
  └─VolGroup00-LogVolHome 253:7    0    2G  0 lvm  /home
sdb                         8:16   0   10G  0 disk 
└─sdb1                      8:17   0   10G  0 part 
sdc                         8:32   0    2G  0 disk 
sdd                         8:48   0    1G  0 disk 
├─vg_var-lv_var_rmeta_0   253:2    0    4M  0 lvm
│ └─vg_var-lv_var         253:6    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_0  253:3    0  952M  0 lvm
  └─vg_var-lv_var         253:6    0  952M  0 lvm  /var
sde                         8:64   0    1G  0 disk 
├─vg_var-lv_var_rmeta_1   253:4    0    4M  0 lvm
│ └─vg_var-lv_var         253:6    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_1  253:5    0  952M  0 lvm
  └─vg_var-lv_var         253:6    0  952M  0 lvm  /var
```

Теперь загрузим в нашу домашнюю директорию несколько файлов для эксперимента со
снэпшотами, качаем первый файл:
```
curl https://download.samba.org/pub/rsync/rsync.html --output rsync_man.html
```

Вывод команды:
```
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  194k  100  194k    0     0   181k      0  0:00:01  0:00:01 --:--:--  181k
```

Загружаем второй файл:
```
curl http://man7.org/linux/man-pages/man1/cp.1.html --output cp_man.html
```

Вывод команды:
```
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 13772  100 13772    0     0  33761      0 --:--:-- --:--:-- --:--:-- 33837
```

Загружаем третий файл:
```
curl http://man7.org/linux/man-pages/man1/less.1.html --output less_man.html
```

Вывод команды:
```
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 95452  100 95452    0     0   172k      0 --:--:-- --:--:-- --:--:--  172k
```

Теперь делаем снэпшот (под именем home_snapshot) нашей файловой системы, указывая
том LogVolHome, где расположена директория /home:
```
sudo lvcreate --size 250M --snapshot --name home_snapshot /dev/VolGroup00/LogVolHome
```

Вывод команды:
```
  Rounding up size to full physical extent 256.00 MiB
  Logical volume "home_snapshot" created.
```

Просмотрим содержимое домашней директории:
```
ls -lh
```

Вывод команды:
```
total 308K
-rw-rw-r--. 1 vagrant vagrant  14K May 25 16:20 cp_man.html
-rw-rw-r--. 1 vagrant vagrant  94K May 25 16:28 less_man.html
-rw-rw-r--. 1 vagrant vagrant 195K May 25 16:19 rsync_man.html
```

Просмотрим статистику по томам lvm:
```
sudo lvs
```

Вывод команды:
```
  LV            VG         Attr       LSize   Pool Origin     Data%  Meta%  Move Log Cpy%Sync Convert
  LogVol00      VolGroup00 -wi-ao----   8.00g
  LogVol01      VolGroup00 -wi-ao----   1.50g
  LogVolHome    VolGroup00 owi-aos---   2.00g
  home_snapshot VolGroup00 swi-a-s--- 256.00m      LogVolHome 0.01
  lv_var        vg_var     rwi-aor--- 952.00m                                        100.00
```

Удалим один файл:
```
rm less_man.html 
```

Просмотрим текущее содержимое домашней директории:
```
ls -lh
```

Вывод команды:
```
total 212K
-rw-rw-r--. 1 vagrant vagrant  14K May 25 16:20 cp_man.html
-rw-rw-r--. 1 vagrant vagrant 195K May 25 16:19 rsync_man.html
```

Теперь выходим из доманшней директории, что бы далее от неё можно было отмонтировать
том и произвести откат изменений в файловой системе, а именно удаление файла:
```
sudo umount /home/
```

Делаем откат изменений с помощью команды `lvconvert`:
```
sudo lvconvert --merge /dev/VolGroup00/home_snapshot 
```

Вывод команды:
```
  Merging of volume VolGroup00/home_snapshot started.
  VolGroup00/LogVolHome: Merged: 100.00%
```

Снова монтируем том в директорию /home:
```
sudo mount /home/
```

Возвращаемся в домашнюю директорию:
```
[vagrant@lvm /]$ cd
```

Просматриваем содержимое директории и видим, что удалённый файл снова на месте:
```
ls -lh
```

Вывод команды:
```
total 308K
-rw-rw-r--. 1 vagrant vagrant  14K May 25 16:20 cp_man.html
-rw-rw-r--. 1 vagrant vagrant  94K May 25 16:28 less_man.html
-rw-rw-r--. 1 vagrant vagrant 195K May 25 16:19 rsync_man.html
```

## **Создание тома для /opt с поддержкой кэширования и файловой системой btrfs**
Теперь создадим том lvm для директории /opt с поддержкой кэширования. Для этого
создадим из имеющихся дисков sdb1 и sdc1 физические тома для lvm:
```
sudo pvcreate /dev/sdb1 /dev/sdc1
WARNING: xfs signature detected on /dev/sdb1 at offset 0. Wipe it? [y/n]: y
```
Вывод команды:
```
  Wiping xfs signature on /dev/sdb1.
  Physical volume "/dev/sdb1" successfully created.
  Physical volume "/dev/sdc1" successfully created.
```

Создадим группу my_vg и добавим в неё новые тома:
```
sudo vgcreate my_vg /dev/sdb1 /dev/sdc1
```

Вывод команды:
```
  Volume group "my_vg" successfully created
```

Создадим логический том размером 8 Гб на основе диска sdb1 и добавим его
в группу my_vg:
```
sudo lvcreate -L 8G --name slow_lv my_vg /dev/sdb1
```

Вывод команды:
```
  Logical volume "slow_lv" created.
```

Теперь создадим том на основе диска sdc1 для кэша и так же добавим его в группу 
my_vg:
```
sudo lvcreate -L 1.5G --name cache_lv my_vg /dev/sdc1
```

Вывод команды:
```
  Logical volume "cache_lv" created.
```

Далее создадим логический том для кэширования метаданных и добавим его в группу
my_vg:
```
sudo lvcreate -L 200M --name metacache_lv my_vg /dev/sdc1
```

Вывод команды:
```
  Logical volume "metacache_lv" created.
```

Создадим логический том пула кэша путём комбинирования тома кэша метаданных и
тома кэша данных:
```
sudo lvconvert --type cache-pool --cachemode writethrough --poolmetadata my_vg/metacache_lv my_vg/cache_lv
  WARNING: Converting my_vg/cache_lv and my_vg/metacache_lv to cache pool's data and metadata volumes with metadata wiping.
  THIS WILL DESTROY CONTENT OF LOGICAL VOLUME (filesystem etc.)
Do you really want to convert my_vg/cache_lv and my_vg/metacache_lv? [y/n]: y
```

Вывод команды:
```
  Converted my_vg/cache_lv and my_vg/metacache_lv to cache pool.
```

Просмотрим информацию о группах и томах lvm:
```
sudo lvs -a -o +devices
```

Вывод команды:
```
  LV                VG         Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert Devices
  LogVol00          VolGroup00 -wi-ao----   8.00g                                                     /dev/sda3(0)
  LogVol01          VolGroup00 -wi-ao----   1.50g                                                     /dev/sda3(1199)
  LogVolHome        VolGroup00 -wi-ao----   2.00g                                                     /dev/sda3(256)
  cache_lv          my_vg      Cwi---C---   1.50g                                                     cache_lv_cdata(0)
  [cache_lv_cdata]  my_vg      Cwi-------   1.50g                                                     /dev/sdc1(0)
  [cache_lv_cmeta]  my_vg      ewi------- 200.00m                                                     /dev/sdc1(384)
  [lvol0_pmspare]   my_vg      ewi------- 200.00m                                                     /dev/sdb1(2048)
  slow_lv           my_vg      -wi-a-----   8.00g                                                     /dev/sdb1(0)
  lv_var            vg_var     rwi-aor--- 952.00m                                    100.00           lv_var_rimage_0(0),lv_var_rimage_1(0)
  [lv_var_rimage_0] vg_var     iwi-aor--- 952.00m                                                     /dev/sdd(1)
  [lv_var_rimage_1] vg_var     iwi-aor--- 952.00m                                                     /dev/sde(1)
  [lv_var_rmeta_0]  vg_var     ewi-aor---   4.00m                                                     /dev/sdd(0)
  [lv_var_rmeta_1]  vg_var     ewi-aor---   4.00m                                                     /dev/sde(0)
```

Теперь создадим логический том с поддержкой кэширования путём комбинирования
исходного логического тома slow_lv и логического тома типа cache-pool с именем
cache_lv:
```
sudo lvconvert --type cache --cachepool my_vg/cache_lv my_vg/slow_lv
Do you want wipe existing metadata of cache pool my_vg/cache_lv? [y/n]: y
```

Вывод команды:
```
  Logical volume my_vg/slow_lv is now cached.
```

Ещё раз просмотрим информацию о группах и томах lvm:
```
sudo lvs -a -o +devices
```

Вывод команды:
```
  LV                VG         Attr       LSize   Pool       Origin          Data%  Meta%  Move Log Cpy%Sync Convert Devices
  LogVol00          VolGroup00 -wi-ao----   8.00g                                                                    /dev/sda3(0)
  LogVol01          VolGroup00 -wi-ao----   1.50g                                                                    /dev/sda3(1199)
  LogVolHome        VolGroup00 -wi-ao----   2.00g                                                                    /dev/sda3(256)
  [cache_lv]        my_vg      Cwi---C---   1.50g                            0.04   0.12            0.00             cache_lv_cdata(0)
  [cache_lv_cdata]  my_vg      Cwi-ao----   1.50g                                                                    /dev/sdc1(0)
  [cache_lv_cmeta]  my_vg      ewi-ao---- 200.00m                                                                    /dev/sdc1(384)
  [lvol0_pmspare]   my_vg      ewi------- 200.00m                                                                    /dev/sdb1(2048)
  slow_lv           my_vg      Cwi-a-C---   8.00g [cache_lv] [slow_lv_corig] 0.04   0.12            0.00             slow_lv_corig(0)
  [slow_lv_corig]   my_vg      owi-aoC---   8.00g                                                                    /dev/sdb1(0)
  lv_var            vg_var     rwi-aor--- 952.00m                                                   100.00           lv_var_rimage_0(0),lv_var_rimage_1(0)
  [lv_var_rimage_0] vg_var     iwi-aor--- 952.00m                                                                    /dev/sdd(1)
  [lv_var_rimage_1] vg_var     iwi-aor--- 952.00m                                                                    /dev/sde(1)
  [lv_var_rmeta_0]  vg_var     ewi-aor---   4.00m                                                                    /dev/sdd(0)
  [lv_var_rmeta_1]  vg_var     ewi-aor---   4.00m                                                                    /dev/sde(0) 
```

На данном этапе, когда логический том с поддержкой кэширования готов, создадим
на нём файловую систему типа btrfs:
```
sudo mkfs.btrfs /dev/my_vg/slow_lv
```

Вывод команды:
```
btrfs-progs v4.9.1
See http://btrfs.wiki.kernel.org for more information.

Performing full device TRIM /dev/my_vg/slow_lv (8.00GiB) ...
Label:              (null)
UUID:               46e2df46-aef0-42b8-8206-11a0f8f462e6
Node size:          16384
Sector size:        4096
Filesystem size:    8.00GiB
Block group profiles:
  Data:             single            8.00MiB
  Metadata:         DUP             409.56MiB
  System:           DUP               8.00MiB
SSD detected:       no
Incompat features:  extref, skinny-metadata
Number of devices:  1
Devices:
   ID        SIZE  PATH
    1     8.00GiB  /dev/my_vg/slow_lv
```

Теперь перейдём к переносу директории /opt  на новый том с поддержкой кэшироавния.
Создадим временную директорию куда примонтируем новый том:
```
sudo mkdir /mnt/back_opt
```
Примонтируем новый том в созданную директорию:
```
sudo mount /dev/my_vg/slow_lv /mnt/back_opt/
```

Копируем данные из /opt во временную директорию /mnt/back_opt  с помощью утилиты
rsync:
```
sudo rsync --archive --hard-links --sparse --acls --xattrs --info=progress2 /opt/* /mnt/back_opt
     20,728,246  99%   54.40MB/s    0:00:00 (xfr#324, to-chk=0/366)
```

Удаляем всё содержимое /opt:
```
sudo rm -rf /opt/*
```

Отмонтирум том с поддержкой кэширования от временной директории:
```
sudo umount /mnt/back_opt/
```

Примонитруем том с кэшированием в /opt:
```
sudo mount /dev/my_vg/slow_lv /opt/
```

Просмотрим получившуюся в результате нашей работы структуру файловой системы:
```
lsblk 
```

Вывод команды:
```
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                         8:0    0   40G  0 disk 
├─sda1                      8:1    0    1M  0 part 
├─sda2                      8:2    0    1G  0 part /boot
└─sda3                      8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00   253:0    0    8G  0 lvm  /
  ├─VolGroup00-LogVol01   253:1    0  1.5G  0 lvm  [SWAP]
  └─VolGroup00-LogVolHome 253:7    0    2G  0 lvm  /home
sdb                         8:16   0   10G  0 disk 
└─sdb1                      8:17   0   10G  0 part 
  └─my_vg-slow_lv_corig   253:11   0    8G  0 lvm
    └─my_vg-slow_lv       253:8    0    8G  0 lvm  /opt
sdc                         8:32   0    2G  0 disk 
└─sdc1                      8:33   0    2G  0 part 
  ├─my_vg-cache_lv_cdata  253:9    0  1.5G  0 lvm
  │ └─my_vg-slow_lv       253:8    0    8G  0 lvm  /opt
  └─my_vg-cache_lv_cmeta  253:10   0  200M  0 lvm
    └─my_vg-slow_lv       253:8    0    8G  0 lvm  /opt
sdd                         8:48   0    1G  0 disk 
├─vg_var-lv_var_rmeta_0   253:2    0    4M  0 lvm
│ └─vg_var-lv_var         253:6    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_0  253:3    0  952M  0 lvm
  └─vg_var-lv_var         253:6    0  952M  0 lvm  /var
sde                         8:64   0    1G  0 disk 
├─vg_var-lv_var_rmeta_1   253:4    0    4M  0 lvm
│ └─vg_var-lv_var         253:6    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_1  253:5    0  952M  0 lvm
  └─vg_var-lv_var         253:6    0  952M  0 lvm  /var
```

Просмотрим содержимое директории /opt:
```
ls -lh /opt/
```

Вывод команды:
```
total 0
drwxr-xr-x. 1 root root 122 May 25 09:08 VBoxGuestAdditions-6.0.22
```

На данном этапе работа завершена.
