Лабораторная работа №5 по курсу Linux Pro.

Задание:
	1) уменьшить том под / до 8G
	2) выделить том под /home
	3) выделить том под /var (/var - сделать в mirror)
	4) для /home - сделать том для снэпшотов
	5) прописать монтирование в fstab (попробовать с разными опциями и разными файловыми системами на выбор)
	

Решение:
1)
Для выполнения ДЗ использовался стенд с ОС Ubuntu 20.04.
Изначальный объем диска, который смонтирован в корень, составляет 31Гб, общий размер диска 62Гб (см скриншот 1).
Создал временный том и вольюм группу для переноса корневой системы: pvcreate /dev/sdb; vgcreate vg_root /dev/sdb
Создал логикал вольюм lvcreate -n lv_root -l +100%FREE /dev/vg_root
Т.к. исходный локигал вольюм имеет файловую систему ext4, то и на временном логикал вольюме сделал такую же файловую систему: 
mkfs.ext4 /dev/vg_root/lv_root
Подмонтировал логикал вольюм в каталог mnt mount /dev/vg_root/lv_root /mnt
Установил утилиту dump. 
Сделал бэкап корня, файл бэкапа положил в каталог mnt: dump -0uf /mnt/bk /
Переместил бэкап в директорию /home/vagrant: mv /mnt/bk /home/vagrant/
Переместился в каталог mnt и распаковал архив: cd /mnt && restore -rf /home/vagrant/bk /mnt
Перемонтировал каталоги: for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
Перешел в chroot: chroot /mnt/
Обновил grub: grub-mkconfig -o /boot/grub/grub.cfg
Перезагрузил ВМ, корень был смонтирован с временного логикал вольюм.
Удалил исходный логикал вольюм и создал новый на 8Гб: lvremove /dev/ubuntu-vg/ubuntu-lv; lvcreate -n ubuntu-vg/ubuntu-lv -L 8G /dev/ubuntu-vg
Создал на логикал вольюм ubuntu-lv файловую систему ext4 и подмонтировал ее в каталог mnt: mkfs.ext4 /dev/ubuntu-vg/ubuntu-lv; mount /dev/ubuntu-vg/ubuntu-lv /mnt
Повторил процедуру со снятием бэкапа корня с помощью dump и восстановления его в каталог /mnt с помощью restore
Перешел в chroot: chroot /mnt/
Обновил grub: grub-mkconfig -o /boot/grub/grub.cfg
Перезагрузил ВМ, корень был смонтирован с основного логикал вольюм, объемом 8Гб (см скриншот 2)

2)
Создал логикал вольюм под home, объемом 2Гб: lvcreate -n LogVol_Home -L 2G /dev/ubuntu-vg
Создал файловую систему xfs и подномтировал в каталог /mnt: mkfs.xfs /dev/ubuntu-vg/LogVol_Home; mount /dev/ubuntu-vg/LogVol_Home /mnt/
Перенес данные в каталог /mnt: cp -aR /home/* /mnt/
Отмонтировал каталог /mnt: umount /mnt
Прописал монтирование в fstab: echo "`blkid | grep Home | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab
Смонтировал в каталог /home: mount /dev/ubuntu-vg/LogVol_Home /home/ (см скриншот 3)


3)
Создал физикал вольюм, в который включил 2 диска: pvcreate /dev/sd{c,d}
Создал волью группу:  vgcreate vg_var /dev/sd{c,d}
Создал логикал вольюм, объемом 950Мб: lvcreate -L 950M -m1 -n lv_var vg_var
Создал файловую систему на lv_var: mkfs.ext4 /dev/vg_var/lv_var
Смонтировал логикал вольюм в каталог /mnt и перенес туда данные с каталога /var: mount /dev/vg_var/lv_var /mnt; cp -aR /var/* /mnt/
Отмонтировал логикал волью от /mnt: umount /mnt
Поправил fstab: echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab
Смонитровал логикал волью в каталог /var: mount /dev/vg_var/lv_var /var
Каталог успешно смонтировался (см скриншот 4)

4)
Создал файлы в каталоге /home
Создал снэпшот логикал волью LogVol_Home: lvcreate -L 100MB -s -n home_snap /dev/ubuntu-vg/LogVol_Home (см скриншот 5)
Удалил часть созданных файлов
Отмонтировал логикал вольюм /dev/ubuntu-vg/LogVol_Home от каталога /home
Смерджил снэпшот с логикал вольюм: lvconvert --merge /dev/ubuntu-vg/home_snap
Подмонтировал логикал вольюм /dev/ubuntu-vg/LogVol_Home в каталог /home
Удаленные файлы оказались на месте

5)
Все данные прописаны в файле fstab (см скриншот 6)
