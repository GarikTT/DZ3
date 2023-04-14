На имеющемся образе centos/7 - v. 1804.2
1) Уменьшить том под / до 8G
2) Выделить том под /home
3) Выделить том под /var - сделать в mirror
4) /home - сделать том для снапшотов
5) Прописать монтирование в fstab. Попробовать с разными опциями и разными
файловыми системами ( на выбор)
Работа со снапшотами:
- сгенерить файлы в /home/
- снять снапшот
- удалить часть файлов
- восстановится со снапшота

1.	//Скачиваем по ссылке Vagrantfile
	script --timing=time_loading_log loading.log
	vagrant up
	vagrant ssh
	exit
	exit
	// Получили файл загрузки.
	// Воспроизведение действий -
	scriptreplay --timing=time_loading_log loading.log -d 30

2.	// Прохождение урока.
	script --timing=time_lesson_log lesson.log
	vagrant up
	vagrant ssh
	sudo -i
	lsblk
	lvmdiskscan
	pvcreate /dev/sdb
	vgcreate otus /dev/sdb
	lvcreate -l+80%FREE -n test otus
	vgdisplay otus
	vgdisplay -v otus | grep 'PV NAME'
	lvdisplay /dev/otus/test
	vgs
	lvs
	lvcreate -L100M -n small otus
	lvs
	mkfs.ext4 /dev/otus/test
	mkdir /data
	mount /dev/otus/test /data/
	mount | grep /data
	pvcreate /dev/sdc
	vgextend otus /dev/sdc
	vgdisplay -v otus | grep 'PV Name'
	vgs
	dd if=/dev/zero of=/data/test.log bs=1M count=8000 status=progress
	df -Th /data/
	lvextend -l+80%FREE /dev/otus/test
	lvs /dev/otus/test
	df -Th /data/
	umount /data/
	e2fsck -fy /dev/otus/test
	resize2fs /dev/otus/test 10G
	lvreduce /dev/otus/test -L 10G
	mount /dev/otus/test /data/
	df -Th /data/
	lvs /dev/otus/test
	lvcreate -L 500M -s -n test-snap /dev/otus/test
	vgs -o +lv_size,lv_name | grep test
	lsblk
	mkdir /data-snap
	mount /dev/otus/test-snap /data-snap/
	ll /data-snap/
	rm /data-snap/test.log
	umount /data-snap
	umount /data
	lvconvert --merge /dev/otus/test-snap
	mount /dev/otus/test /data
	ll /data
	pvcreate /dev/sd{d,e}
	vgcreate vg0 /dev/sd{d,e}
	lvcreate -l+80%FREE -m1 -n mirror vg0
	lvs
	// Воспроизведение действий -
	scriptreplay --timing=time_lesson_log lesson.log -d 30

3.	Домашнее задание:
	script --timing=time_homework_log homework.log
	vagrant up
	vagrant ssh
	sudo -i
	lsblk
	pvcreate /dev/sdb
	vgcreate vg_root /dev/sdb
	lvcreate -n lv_root -l +100%FREE /dev/vg_root
	mkfs.xfs /dev/vg_root/lv_root
	mount /dev/vg_root/lv_root /mnt
	//yum install -y xfsdump   Устанавливаем через Vagrantfile
	xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt
	for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
	chroot /mnt/
	grub2-mkconfig -o /boot/grub2/grub.cfg
	cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g;
	s/.img//g"` --force; done

	//Ну и для того, чтобы при загрузке был смонтирован нужный root нужно в файле
	// /boot/grub2/grub.cfg заменить rd.lvm.lv=VolGroup00/LogVol00 на rd.lvm.lv=vg_root/lv_root
	sed -i 's*rd.lvm.lv=VolGroup00/LogVol00*rd.lvm.lv=vg_root/lv_root*' /boot/grub2/grub.cfg

	//Перезагружаемся успешно с новым рут томом.
	exit
	shutdown -r now

	vagrant ssh
	sudo -i
	lsblk
	lvremove /dev/VolGroup00/LogVol00
	lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00
	mkfs.xfs /dev/VolGroup00/LogVol00
	mount /dev/VolGroup00/LogVol00 /mnt
	xfsdump -J - /dev/vg_root/lv_root | xfsrestore -J - /mnt
	for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
	chroot /mnt/
	grub2-mkconfig -o /boot/grub2/grub.cfg

	cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g;
	s/.img//g"` --force; done

	pvcreate /dev/sdc /dev/sdd
	vgcreate vg_var /dev/sdc /dev/sdd
	lvcreate -L 950M -m1 -n lv_var vg_var
	mkfs.ext4 /dev/vg_var/lv_var
	mount /dev/vg_var/lv_var /mnt
	cp -aR /var/* /mnt/ # rsync -avHPSAX /var/ /mnt/
	mkdir /tmp/oldvar && mv /var/* /tmp/oldvar
	umount /mnt
	mount /dev/vg_var/lv_var /var
	echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab
	exit

	// Делаем перезагрузку

	shutdown -r now
	vagrant ssh
	sudo -i
	lvremove /dev/vg_root/lv_root
	vgremove /dev/vg_root
	pvremove /dev/sdb
	lvcreate -n LogVol_Home -L 2G /dev/VolGroup00
	mkfs.xfs /dev/VolGroup00/LogVol_Home
	mount /dev/VolGroup00/LogVol_Home /mnt/
	cp -aR /home/* /mnt/
	rm -rf /home/*
	umount /mnt
	mount /dev/VolGroup00/LogVol_Home /home/
	echo "`blkid | grep Home | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab
	touch /home/file{1..20}
	lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home
	rm -f /home/file{11..20}
	umount /home
	lvconvert --merge /dev/VolGroup00/home_snap
	mount /home
	#Выход из root
	exit

	//Выход из vagrant ssh
	exit

	#Выход из script
	exit

	// Воспроизведение действий -
	scriptreplay --timing=time_homework_log homework.log -d 30
