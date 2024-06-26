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
&ensp;&ensp;На диске /dev/sda в разделе /dev/sda3 посредством LVM реализована группа разделов VolGroup00,<br/> 
на которой,в свою очередь, находятся логические разделы LogVol00 (37.5 GB) для размещения корня<br/> 
исходной операционной системы и LogVol01 (1 GB) для размещения SWAP-раздела. Раздел /dev/sda2   
является разделом для размещения /boot. Диски sdb (10 GB), sdc (2 GB), sdd (1 GB), sde (1 GB) чистые. 
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
строке (linux16), где находится параметр rd.lvm.lv, задать параметр enforcing=0. Данный параметр переключает<br/>
SELinux в режим permissive. <br/>
&ensp;&ensp;7. Загрузка новой системы <br/>
&ensp;&ensp;8. Удаление логического раздела исходной системы:
```shell
lvremove /dev/VolGroup00/LogVol00
```
9. Создание логического раздела размером 8 GB:
```shell
lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00
```
10. Создание файловой системы на разделе и его монтирование для возвращения данных исходной системы:
```shell
mkfs.xfs /dev/VolGroup00/LogVol00
mount /dev/VolGroup00/LogVol00 /mnt
```
11. Копирование данных / исходной системы обратно:
```shell
xfsdump -J - /dev/vg_root/lv_root | /mnt
```
12. Подготовка к работе с прежней системой:
```shell
mount --bind /proc /mnt/proc
mount --bind /sys /mnt/sys
mount --bind /dev /mnt/dev
mount --bind /run /mnt/run
mount --bind /boot /mnt/boot
chroot /mnt
```
13. Обновление образа inird прежней системы:
```shell
dracut /boot/initramfs-$(uname -r).img
```
14. Конфигурирование GRUB для загрузки прежней системы:
```shell
grub2-mkconfig -o /boot/grub2/grub.cfg
```
15. Подготовка зеркала на дисках /dev/sdc /dev/sdd:
```shell
pvcreate /dev/sdc /dev/sdd
vgcreate vg_var /dev/sdc /dev/sdd
lvcreate -L 950M -m1 -n lv_var vg_var
```
16. Создание файловой системы на зеркале и перенос на неё /var:
```shell
mkfs.ext4 /dev/vg_var/lv_var
mount /dev/vg_var/lv_var /mnt
cp -aR /var/* /mnt/
```
17. Монтирование нового var в /mnt/var:
```shell
umount /mnt
mount /dev/vg_var/lv_var /var
```
18. Внесение данных для автоматического монтирования /var в fstab:
```shell
echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab
```
19. Загрузка прежней системы <br/>
&ensp;&ensp;20. Удаление временного логического раздела, группы разделов, физического раздела:
```shell
lvremove /dev/vg_root/lv_root
vgremove /dev/vg_root
pvremove /dev/sdb
```
20. Подготовка тома для /home и его монтирование в указанную точку:
```shell
lvcreate -n LogVol_Home -L 2G /dev/VolGroup0
mkfs.xfs /dev/VolGroup00/LogVol_Home
mount /dev/VolGroup00/LogVol_Home /mnt/
cp -aR /home/* /mnt/
rm -rf /home/*
umount /mnt
mount /dev/VolGroup00/LogVol_Home /home/
```
21. Внесение данных для автоматического монтирования /home в fstab:
```shell
echo "`blkid | grep Home: | awk '{print $2}'` /var xfs defaults 0 0" >> /etc/fstab
```
22. Создание группы файлов в /home:
```shell
touch /home/file{1..20}
```
23. Подготовка снапшота:
```shell
lvcreate -L 100MB -s -n home_snap dev/VolGroup00/LogVol_Home
```
24. Удаление файлов в /home:
```shell
rm -f /home/file{1..20}
```
25. Восстановление данных из снапшота:
```shell
umount /home
lvconvert --merge /dev/VolGroup00/home_snap
mount /home
ls /home | sort
file1
file10
file11
file12
file13
file14
file15
file16
file17
file18
file19
file2
file20
file3
file4
file5
file6
file7
file8
file9
vagrant
```
##### &ensp;&ensp; Более подробный ход решения задачи и вывод команд представлен в прилагающемся логе утилиты Script - course.txt. #####
