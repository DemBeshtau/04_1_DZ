# Файловые системы и LVM #
  - На образе centos/7 - v. 1804.2:<br/>
1. Уменьшить том под корневую директорию / до 8 GB;<br/>
2. Выделить том под /home;<br/>
3. Выделить том под /var - реализовать в mirror;<br/>
4. В /home - подготовить том для снапшотов;<br/>
5. Реализовать автомонтирование разделов /home и /var в fstab;<br/>
6. Продемонстрировать работу со снапшотами:<br/>
&ensp;&ensp; a. создать файлы в /home/;<br/>
&ensp;&ensp; b. снять снэпшот;<br/>
&ensp;&ensp; c. удалить файлы в /home/;<br/>
&ensp;&ensp; d. восстановить содержимое /home/ со снапшота;<br/>
### Исходные данные ###
&ensp;&ensp;На диске /dev/sda в разделе /dev/sda3 посредством LVM реализована группа разделов VolGroup00, на которой,<br/>
в свою очередь, находятся логические разделы LogVol00 (37.5 GB) для размещения корня исходной операционной<br/>
системы и LogVol01 (1 GB) для размещения SWAP-раздела. Раздел /dev/sda2 является разделом для размещения<br/> 
/boot. Диски sdb (10 GB), sdc (2 GB), sdd (1 GB), sde (GB) чистые. 
### Ход решения ###
1. Подготовка временного тома для корневого раздела / исходной системы:
```shell
pvcreate /dev/sdb
vgcreate vg_root /dev/sdb
lvcreate -n lv_root -l +100%FREE /dev/vg_root
```
2. Создание файловой системы на временном томе и его монтирование для переноса данных исходной системы:
```shell
mkfs.xfs /dev/vg_root/lv_root
mount /dev/vg_root/lv_root /mnt
```
3. Копирование данных раздела / исходной системы:
```shell
xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt
```
4. Подготовка к работе с новой системой:
```shell
mount --bind /proc /mnt/proc
mount --bind /sys /mnt/sys
mount --bind /dev /mnt/dev
mount --bind /run /mnt/run
mount --bind /boot /mnt/boot
chroot /mnt
```
5. Обновление образа inird:
```shell
dracut /boot/initramfs-$(uname -r).img
```
6. Конфигурирование GRUB для загрузки новой системы:
```shell
grub2-mkconfig -o /boot/grub2/grub.cfg
```
&ensp;&ensp; Для загрузки новой системы необходимо в файле grub.cfg изменить значения параметра<br/>
rd.lvm.lv=VolGroup00/LogVol00 на rd.lvm.lv=vg_root/lv_root. Кроме того, с целью предупреждения проблем <br/> 
с загрузкой системы, возникающих из-за работы системы безопасности SELinux, необходимо в той же<br/> 
строке (linux16), где находится параметр rd.lvm.lv, задать параметр enforcing=0. Данный параметр переводит<br/>    


