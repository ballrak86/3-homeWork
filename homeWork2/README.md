------------------------------------------------------------
Описание файлов в директории
logFileFull.log - полный лог выполнения
used_commands.txt - команды которые использовал
usedInetResorce.txt - ресурсы в интернете которые мне помогли решить ДЗ

Vagrant_folder - все что понадобится для поднятия VM и краткое описание файлов в ней
Vagrantfile - вагрант файл
------------------------------------------------------------
Почему я сделал так а не иначе.
zfs я расположил на дисках LVM. Просто хотелось еще раз попрактиковаться с LVM.
В принципе можно было сразу свободные диски добавить в zpool
------------------------------------------------------------
Описание как запустить виртуальную машину (кратко)
Выполнить команду
vagrant up
vagrant ssh
   [root@lvm ~]# lsblk
   NAME                       MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
   sda                          8:0    0   40G  0 disk
   ├─sda1                       8:1    0    1M  0 part
   ├─sda2                       8:2    0    1G  0 part /boot
   └─sda3                       8:3    0   39G  0 part
     ├─VolGroup00-LogVol00    253:0    0    8G  0 lvm  /
     ├─VolGroup00-LogVol01    253:1    0  1.5G  0 lvm  [SWAP]
     └─VolGroup00-LogVol_Home 253:9    0    2G  0 lvm  /home
   sdb                          8:16   0   10G  0 disk
   ├─zfs_VG-opt_LV            253:2    0  5.5G  0 lvm
   └─zfs_VG-cache_LV          253:3    0  5.5G  0 lvm
   sdc                          8:32   0    2G  0 disk
   ├─vg_var-lv_var_rmeta_0    253:4    0    4M  0 lvm
   │ └─vg_var-lv_var          253:8    0  952M  0 lvm  /var
   └─vg_var-lv_var_rimage_0   253:5    0  952M  0 lvm
     └─vg_var-lv_var          253:8    0  952M  0 lvm  /var
   sdd                          8:48   0    1G  0 disk
   ├─vg_var-lv_var_rmeta_1    253:6    0    4M  0 lvm
   │ └─vg_var-lv_var          253:8    0  952M  0 lvm  /var
   └─vg_var-lv_var_rimage_1   253:7    0  952M  0 lvm
     └─vg_var-lv_var          253:8    0  952M  0 lvm  /var
   sde                          8:64   0    1G  0 disk
   └─zfs_VG-cache_LV          253:3    0  5.5G  0 lvm
   [root@lvm ~]# zfs list -t snapshot
   NAME                  USED  AVAIL  REFER  MOUNTPOINT
   zfsPool/opt@snapOpt    14K      -  39.5K  -
   [root@lvm ~]# zpool status
     pool: zfsPool
    state: ONLINE
     scan: none requested
   config:
   
           NAME                       STATE     READ WRITE CKSUM
           zfsPool                    ONLINE       0     0     0
             opt_LV                   ONLINE       0     0     0
           cache
             dm-name-zfs_VG-cache_LV  ONLINE       0     0     0
   
   errors: No known data errors
   [root@lvm ~]# zfs list
   NAME          USED  AVAIL  REFER  MOUNTPOINT
   zfsPool       235K  5.30G    24K  /mnt/
   zfsPool/opt  53.5K  5.30G  39.5K  /opt

------------------------------------------------------------
Полное описание vagrant up

------------Vagrantfile (только интересные моменты)------------
#Указал свой vagrant Cloud с подготовленным box файлом
:box_name => "alenchik/centos-7",


------------------------------------------------------------
Интересные моменты по коммандам
1)vi /etc/yum.repos.d/zfs.repo
Отключил в файле репозиторий с zfs DKMS и включил репозиторий  kABI. При обновлении ядра zfs kABI не нужно заново пересобирать.

2)lsmod - посмотреть загруженные модули

3)modprobe - позволяет добавлять/удалять модули ядра

4)zpool позволяет работать с пулом zfs

5)zfs - работа с файловой системой

6)zpool add zfsPool cache dm-name-zfs_VG-cache_LV
В zfs два механизма кеширования для чтения
(1)ARC - это кэш ZFS, расположенный в оперативной памяти
(2)L2ARC - его продолжение (Layer 2, второй уровень), но на более медленном, чем оперативная память носителе. Обычно, в роли носителей для L2ARC используются SSD диски.
В комманде выше я включил кеш L2ARC. Проверить его статус можно через комманду:
zpool status

Чтобы включить кеш ARC
Можно настроить его с помощью параметров ядра. Для GRUB поправить следующую строку в файле /etc/default/grub:
   GRUB_CMDLINE_LINUX_DEFAULT="quiet"
Добавить строку “zfs.zfs_arc_max=(size)” указав размер в байтах:
   GRUB_CMDLINE_LINUX_DEFAULT="quiet zfs.zfs_arc_max=1073741824" # For 1GiB
После чего перезапи конфигурационный файл GRUB с учётом изменений:
sudo grub-mkconfig -o /boot/grub/grub.cfg