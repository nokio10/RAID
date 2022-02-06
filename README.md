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

Начну попытки сломать рейд зафейлив один из дисков
```
[vagrant@otuslinux mdadm]$ sudo mdadm /dev/md0 --fail /dev/sde
mdadm: set /dev/sde faulty in /dev/md0
[vagrant@otuslinux mdadm]$ cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4]
md0 : active raid6 sdf[4] sde[3](F) sdd[2] sdc[1] sdb[0]
      761856 blocks super 1.2 level 6, 512k chunk, algorithm 2 [5/4] [UUU_U]
```
Успешно! Удаляю "сломаный" диск из массива
```
[vagrant@otuslinux mdadm]$ sudo mdadm /dev/md0 --remove /dev/sde
mdadm: hot removed /dev/sde from /dev/md0
```
Теперь доставлю этот диск в качестве нового в массив
```
[vagrant@otuslinux mdadm]$ sudo mdadm /dev/md0 --add /dev/sde
mdadm: added /dev/sde
[vagrant@otuslinux mdadm]$ cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4]
md0 : active raid6 sde[5] sdf[4] sdd[2] sdc[1] sdb[0]
      761856 blocks super 1.2 level 6, 512k chunk, algorithm 2 [5/5] [UUUUU]
```
Пронаблюдать процесс восстановления диска не удалось, так как при таком объеме оно происходит очень быстро. Рейд восстановлен.

## Создать GPT раздел, пять партиций и смонтировать их на диск

Создаю раздел GTP на RAID и партиции
```
[vagrant@otuslinux mdadm]$ sudo parted -s /dev/md0 mklabel gpt
[vagrant@otuslinux mdadm]$ sudo parted /dev/md0 mkpart primary ext4 0% 20%
[vagrant@otuslinux mdadm]$ sudo parted /dev/md0 mkpart primary ext4 20% 40%
[vagrant@otuslinux mdadm]$ sudo parted /dev/md0 mkpart primary ext4 40% 60%
[vagrant@otuslinux mdadm]$ sudo parted /dev/md0 mkpart primary ext4 60% 80%
[vagrant@otuslinux mdadm]$ sudo parted /dev/md0 mkpart primary ext4 80% 100%
```

Создаю файловую систему на партициях
```
[vagrant@otuslinux mdadm]$ for i in $(seq 1 5); do sudo mkfs.ext4 /dev/md0p$i; done
mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=1024 (log=0)
Fragment size=1024 (log=0)
Stride=512 blocks, Stripe width=1536 blocks
37696 inodes, 150528 blocks
7526 blocks (5.00%) reserved for the super user
First data block=1
Maximum filesystem blocks=33816576
19 block groups
8192 blocks per group, 8192 fragments per group
1984 inodes per group
Superblock backups stored on blocks:
        8193, 24577, 40961, 57345, 73729

Allocating group tables: done
Writing inode tables: done
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=1024 (log=0)
Fragment size=1024 (log=0)
Stride=512 blocks, Stripe width=1536 blocks
38152 inodes, 152064 blocks
7603 blocks (5.00%) reserved for the super user
First data block=1
Maximum filesystem blocks=33816576
19 block groups
8192 blocks per group, 8192 fragments per group
2008 inodes per group
Superblock backups stored on blocks:
        8193, 24577, 40961, 57345, 73729

Allocating group tables: done
Writing inode tables: done
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=1024 (log=0)
Fragment size=1024 (log=0)
Stride=512 blocks, Stripe width=1536 blocks
38456 inodes, 153600 blocks
7680 blocks (5.00%) reserved for the super user
First data block=1
Maximum filesystem blocks=33816576
19 block groups
8192 blocks per group, 8192 fragments per group
2024 inodes per group
Superblock backups stored on blocks:
        8193, 24577, 40961, 57345, 73729

Allocating group tables: done
Writing inode tables: done
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=1024 (log=0)
Fragment size=1024 (log=0)
Stride=512 blocks, Stripe width=1536 blocks
38152 inodes, 152064 blocks
7603 blocks (5.00%) reserved for the super user
First data block=1
Maximum filesystem blocks=33816576
19 block groups
8192 blocks per group, 8192 fragments per group
2008 inodes per group
Superblock backups stored on blocks:
        8193, 24577, 40961, 57345, 73729

Allocating group tables: done
Writing inode tables: done
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=1024 (log=0)
Fragment size=1024 (log=0)
Stride=512 blocks, Stripe width=1536 blocks
37696 inodes, 150528 blocks
7526 blocks (5.00%) reserved for the super user
First data block=1
Maximum filesystem blocks=33816576
19 block groups
8192 blocks per group, 8192 fragments per group
1984 inodes per group
Superblock backups stored on blocks:
        8193, 24577, 40961, 57345, 73729

Allocating group tables: done
Writing inode tables: done
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done
```
Монтирую их по каталогам
```
[vagrant@otuslinux mdadm]$ sudo mkdir -p /raid/part{1,2,3,4,5}
[vagrant@otuslinux mdadm]$ for i in $(seq 1 5); do sudo mount /dev/md0p$i /raid/part$i; done
```
Получился рейд 
```
[vagrant@otuslinux mdadm]$ sudo fdisk -l
........
Disk /dev/md0: 780 MB, 780140544 bytes, 1523712 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 524288 bytes / 1572864 bytes
Disk label type: gpt
Disk identifier: EFA6E935-8165-4D67-BE39-1281EDDB5776
.........
[vagrant@otuslinux mdadm]$ df -h
......
/dev/md0p1      139M  1.6M  127M   2% /raid/part1
/dev/md0p2      140M  1.6M  128M   2% /raid/part2
/dev/md0p3      142M  1.6M  130M   2% /raid/part3
/dev/md0p4      140M  1.6M  128M   2% /raid/part4
/dev/md0p5      139M  1.6M  127M   2% /raid/part5
.......

```
