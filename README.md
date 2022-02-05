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


