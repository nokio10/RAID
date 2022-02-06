# Подготовка

Форкнул репу https://github.com/erlong15/otus-linux, склонировал https://github.com/nokio10/otus-linux на машину командой 
```
git clone https://github.com/nokio10/otus-linux.git
```
Добавил 5ый диск в vagrantfile
```
,
		:sata5 => {
                        :dfile => './sata5.vdi',
                        :size => 250, # Megabytes
                        :port => 5
                }
```
Поднял машину командой:
```
vagrant up
```
И подключился к ней по ssh
```
vagrant ssh
```

# Собрать RAID0/1/5/10 - на выбор

Проверяю наличие нужного количества дисков 
```
[vagrant@otuslinux ~]$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   40G  0 disk
`-sda1   8:1    0   40G  0 part /
sdb      8:16   0  250M  0 disk
sdc      8:32   0  250M  0 disk
sdd      8:48   0  250M  0 disk
sde      8:64   0  250M  0 disk
sdf      8:80   0  250M  0 disk
```
Собираю рейд 6.
```
[vagrant@otuslinux ~]$ sudo mdadm --create --verbose /dev/md0 -l 6 -n 5 /dev/sd{b,c,d,e,f}
mdadm: layout defaults to left-symmetric
mdadm: layout defaults to left-symmetric
mdadm: chunk size defaults to 512K
mdadm: size set to 253952K
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md0 started.
```
Проверяю, что рейд собран.
```
[vagrant@otuslinux ~]$ cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4]
md0 : active raid6 sdf[4] sde[3] sdd[2] sdc[1] sdb[0]
      761856 blocks super 1.2 level 6, 512k chunk, algorithm 2 [5/5] [UUUUU]

unused devices: <none>
[vagrant@otuslinux ~]$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sda      8:0    0   40G  0 disk
`-sda1   8:1    0   40G  0 part  /
sdb      8:16   0  250M  0 disk
`-md0    9:0    0  744M  0 raid6
sdc      8:32   0  250M  0 disk
`-md0    9:0    0  744M  0 raid6
sdd      8:48   0  250M  0 disk
`-md0    9:0    0  744M  0 raid6
sde      8:64   0  250M  0 disk
`-md0    9:0    0  744M  0 raid6
sdf      8:80   0  250M  0 disk
`-md0    9:0    0  744M  0 raid6
```
## Создание конфигурационного файла mdadm.conf

Создаю файл mdadm.conf
```
[vagrant@otuslinux ~]$ sudo mkdir /etc/mdadm
[vagrant@otuslinux mdadm]$ touch mdadm.conf
```
Наполняю файл
```
[vagrant@otuslinux mdadm]$ sudo echo "DEVICE partitions" > /etc/mdadm/mdadm.conf
[vagrant@otuslinux mdadm]$ sudo mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf
[vagrant@otuslinux mdadm]$ ls
mdadm.conf
[vagrant@otuslinux mdadm]$ cat mdadm.conf
DEVICE partitions
ARRAY /dev/md0 level=raid6 num-devices=5 metadata=1.2 name=otuslinux:0 UUID=825c35c6:7d0514f4:dc127643:1d5fd3d7
```

## Сломать/починить RAID

