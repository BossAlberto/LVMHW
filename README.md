Домашнее задание "Файловые системы и LVM"
Первым делом из задание требоваться написать Bash скрипт
В первую очередь скрипт должен запускать через root иначе он не сможет проихвести хотя бы установку)
С помощью команды наш скрипт получет root права
ле чего мы устанавливаем нужные нам утилиты с помощью которых мы и будем работать.
Посл этого создаём ВМ в которой будем работать, не забываем заранеее вложить туда диски. Я сделал 3 диска разных размеров.
Готовим времененный том для раздела командами:
pvcreate /dev/sdb
vgcreate vg_root /dev/sdb
lvcreate -n lv_root -l +100%FREE /dev/vg_root
После чего создаём в нем файловую систему и монтируем для переноса данных
mkfs.ext4 /dev/vg_root/lv_root
mount /dev/vg_root/lv_root /mnt
Копируем в /mnt
rsync -avxHAX --progress / /mnt/
Проверяем командой
ls /mnt
Затем сконфигурируем grub
for i in /proc/ /sys/ /dev/ /run/ /boot/; \
do mount --bind $i /mnt/$i; done
chroot /mnt/
grub-mkconfig -o /boot/grub/grub.cfg
Обновим образ initrd, после чего делаем перезагруску
update-initramfs -u
Не забываем смотреть на диски
lsblk
Теперь нам нужно изменить размер старой VG и вернуть на него рут. Для этого удаляем старый LV размером в 40G и создаём новый на 8G
lvremove /dev/ubuntu-vg/ubuntu-lv
lvcreate -n ubuntu-vg/ubuntu-lv -L 8G /dev/ubuntu-vg
И повторно проделывем все действия,что мы делали изначально - вплоть до шага с initrd и grub
Можно перезагузиться, можно сделать сразу задания с /var
На свободных дисках создаем зеркало
pvcreate /dev/sdc /dev/sdd
vgcreate vg_var /dev/sdc /dev/sdd
lvcreate -L 950M -m1 -n lv_var vg_var
Создаем на нем ФС и перемещаем туда /var
mkfs.ext4 /dev/vg_var/lv_var
mount /dev/vg_var/lv_var /mnt
cp -aR /var/* /mnt/
Можно засейвить содержимое старого /var
mkdir /tmp/oldvar && mv /var/* /tmp/oldvar
И создать новый каталог:
umount /mnt
mount /dev/vg_var/lv_var /var
Правим fstab для автоматического монтирования /var:
echo "`blkid | grep var: | awk '{print $2}'` \
 /var ext4 defaults 0 0" >> /etc/fstab
После чего можно успешно перезагружаться в новый (уменьшенный root) и удалять
временную Volume Group:
lvremove /dev/vg_root/lv_root
vgremove /dev/vg_root
pvremove /dev/sdb
Выделяем том под /home по тому же принципу что делали для /var:
lvcreate -n LogVol_Home -L 2G /dev/ubuntu-vg
mkfs.ext4 /dev/ubuntu-vg/LogVol_Home
mount /dev/ubuntu-vg/LogVol_Home /mnt/
cp -aR /home/* /mnt/
rm -rf /home/*
umount /mnt
mount /dev/ubuntu-vg/LogVol_Home /home/
Правим fstab для автоматического монтирования /home:
echo "`blkid | grep Home | awk '{print $2}'` \
 /home xfs defaults 0 0" >> /etc/fstab
Генерируем файлы в /home/:
touch /home/file{1..20}
Делаем снапшот:
lvcreate -L 100MB -s -n home_snap \
 /dev/ubuntu-vg/LogVol_Home
Удаляем файлы:
rm -f /home/file{11..20}
Восстанавливаем файлы:
umount /home
lvconvert --merge /dev/ubuntu-vg/home_snap
mount /dev/mapper/ubuntu--vg-LogVol_Home /home
Проверка файлов:
ls -al /home
