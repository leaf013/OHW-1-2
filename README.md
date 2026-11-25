# Создание и управление программным RAID с помощью утилиты mdadm

## Содержание
- [Подготовка](#подготовка)
- [Сборка RAID](#сборка-raid)
- [Проведение теста на отказоустойчивость](#проведение-теста-на-отказоустойчивость)
- [Структура проекта](#структура-проекта)

## Подготовка
Запускаем ВМ с установленной Ubuntu с несколькими подключенными виртуальными дисками.\
Подключаемся к терминалу любым удобным способом.\
Для создания и управления программными RAID используется утилита `mdadm`

## Сборка RAID

1. Смотрим какие диски устрановлены в систему
   
   bash:
   ```bash
   lsblk
   ```
   результат:
   ```text
   NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
   sda                         8:0    0   10G  0 disk
   ├─sda1                      8:1    0    1M  0 part
   ├─sda2                      8:2    0  1.8G  0 part /boot
   └─sda3                      8:3    0  8.2G  0 part
     └─ubuntu--vg-ubuntu--lv 253:0    0  8.2G  0 lvm  /
   sdb                         8:16   0  100M  0 disk
   sdc                         8:32   0  100M  0 disk
   sdd                         8:48   0  100M  0 disk
   sde                         8:64   0  100M  0 disk
   sdf                         8:80   0  100M  0 disk
   sdg                         8:96   0  100M  0 disk
   sr0                        11:0    1 1024M  0 rom
   ```
   мы имеем 6 дисков **sda, sdb, sdc, sde, sdf и sdg** по 100 МБ каждый

2. Для создания RAID6 будем использовать первые 5 дисков, а шестой **sdg** прибережем для тестов

   bash:   
   ```bash
   mdadm --create --verbose /dev/md0 -l 6 -n 5 /dev/sd{b,c,d,e,f}
   ```
      <details>
      <summary>Описание параметров</summary>
      --create — создание массива <br>
      --verbose — вывод информации о процессе создания    <br>
      /dev/md0 — имя создаваемого устройства <br>
      -l 6 — уровень RAID 6  <br>
      -n 5 — количество диков,используемых в массиве <br>
      /dev/sd{b,c,d,e,f} — список дисков
      </details>

   вывод в терминале:
   ```
   mdadm: layout defaults to left-symmetric
   mdadm: layout defaults to left-symmetric
   mdadm: chunk size defaults to 512K
   mdadm: size set to 100352K
   mdadm: Defaulting to version 1.2 metadata
   mdadm: array /dev/md0 started.
   ```

3. Проверяем статус RAID
   
   bash:
   ```bash
   cat /proc/mdstat
   ```
   результат:
   ```text
   Personalities : [linear] [multipath] [raid0] [raid1] [raid6] [raid5] [raid4] [raid10]
   md0 : active raid6 sdf[4] sde[3] sdd[2] sdc[1] sdb[0]
      301056 blocks super 1.2 level 6, 512k chunk, algorithm 2 [5/5] [UUUUU]
   unused devices: <none>
   ```

   детальная информация:
   
   bash:
   ```bash
   mdadm -D /dev/md0
   ```
   результат:
   ```text
   /dev/md0:
           Version : 1.2
     Creation Time : Tue Nov 24 19:44:23 2026
        Raid Level : raid6
        Array Size : 301056 (294.00 MiB 308.28 MB)
     Used Dev Size : 100352 (98.00 MiB 102.76 MB)
      Raid Devices : 5
     Total Devices : 5
    Active Devices : 5
   Working Devices : 5
    Failed Devices : 0
     Spare Devices : 0
    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       2       8       48        2      active sync   /dev/sdd
       3       8       64        3      active sync   /dev/sde
       4       8       80        4      active sync   /dev/sdf
   ```
   В конце видим список из 5 активных дисков
   
4. Создаем раздел GPT на нашем массиве

   bash:
   ```bash
   parted -s /dev/md0 mklabel gpt
   ```

5. Создаем партиции

   bash:
   ```bash
   parted /dev/md0 mkpart primary ext4 0% 20%
   parted /dev/md0 mkpart primary ext4 20% 40%
   parted /dev/md0 mkpart primary ext4 40% 60%
   parted /dev/md0 mkpart primary ext4 60% 80%
   parted /dev/md0 mkpart primary ext4 80% 100%
    ```

   
6. На каждой партиции создаем файловую систему ext4

   bash:
   ```bash
   for i in $(seq 1 5); do sudo mkfs.ext4 /dev/md0p$i; done
   ```

7. Создаем каталоги и монтруем

   bash:
   ```bash
   mkdir -p /raid/part{1,2,3,4,5}
   for i in $(seq 1 5); do mount /dev/md0p$i /raid/part$i; done
   ```

   bash:
   ```bash
   ll /dev/md0
   ```

   результат:
   ```text	
   brw-rw---- 1 root disk 9, 0 Nov 24 19:46 /dev/md0
   leaf@ubuntu261122:~$ ll /dev/md0*
   brw-rw---- 1 root disk   9, 0 Nov 24 19:46 /dev/md0
   brw-rw---- 1 root disk 259, 1 Nov 24 19:46 /dev/md0p1
   brw-rw---- 1 root disk 259, 4 Nov 24 19:46 /dev/md0p2
   brw-rw---- 1 root disk 259, 5 Nov 24 19:46 /dev/md0p3
   brw-rw---- 1 root disk 259, 8 Nov 24 19:46 /dev/md0p4
   brw-rw---- 1 root disk 259, 9 Nov 24 19:46 /dev/md0p5
   ```


## Проведение теста на отказоустойчивость

   Смоделируем выход из строя одного из дисков. Пусть это будет **sdf**

   bash:
   ```bash
   sudo mdadm /dev/md0 --fail /dev/sdf
   ```

   результат:
   ```text
   mdadm: set /dev/sdf faulty in /dev/md0
   ```

   Проверяем статус RAID

   bash:
   ```bash
   cat /proc/mdstat
   ```

   результат:
   ```text
   Personalities : [linear] [multipath] [raid0] [raid1] [raid6] [raid5] [raid4] [raid10]
   md0 : active raid6 sdf[4](F) sde[3] sdd[2] sdc[1] sdb[0]
         301056 blocks super 1.2 level 6, 512k chunk, algorithm 2 [5/4] [UUUU_]
   
   unused devices: <none>
   ```
   Видим fail диска **sdf**

   Детальнее
   
   bash:
   ```bash
   mdadm -D /dev/md0
   ```

   результат:
   ```text
   /dev/md0:
           Version : 1.2
     Creation Time : Tue Nov 24 19:44:23 2026
        Raid Level : raid6
        Array Size : 301056 (294.00 MiB 308.28 MB)
     Used Dev Size : 100352 (98.00 MiB 102.76 MB)
      Raid Devices : 5
     Total Devices : 5
    Active Devices : 4
   Working Devices : 4
    Failed Devices : 1
       Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       2       8       48        2      active sync   /dev/sdd
       3       8       64        3      active sync   /dev/sde
       -       0        0        4      removed

       4       8       80        -      faulty   /dev/sdf
   ```
   

   Удаляем из массива этот диск
   
   ```bash
   sudo mdadm /dev/md0 --remove /dev/sdf
   ```

   результат:
   ```text
   mdadm: hot removed /dev/sdf from /dev/md0
   ```

   Взамен "неисправного" **sdf** добавим в RAID заранее подготовленный для этого диск **sdg**:
   
   bash:
   ```bash
   sudo mdadm /dev/md0 --add /dev/sdg
   ```

   результат:
   ```text
   mdadm: added /dev/sdg
   ```

   После добавления диск будет проходить rebuilding. В звисимости от объема данных, это может занять разное время.

   Смотрим текущее состояние массива:

   bash:
   ```bash
   cat /proc/mdstat
   ```

   результат:
   ```text
   Personalities : [linear] [multipath] [raid0] [raid1] [raid6] [raid5] [raid4] [raid10]
   md0 : active raid6 sdg[5] sde[3] sdd[2] sdc[1] sdb[0]
         301056 blocks super 1.2 level 6, 512k chunk, algorithm 2 [5/5] [UUUUU]
   
   unused devices: <none>
   ```
   

   bash:
   ```bash
   sudo mdadm -D /dev/md0
   ```

   результат:
   ```text
   /dev/md0:
           Version : 1.2
     Creation Time : Tue Nov 24 19:44:23 2026
        Raid Level : raid6
        Array Size : 301056 (294.00 MiB 308.28 MB)
     Used Dev Size : 100352 (98.00 MiB 102.76 MB)
      Raid Devices : 5
     Total Devices : 5
    Active Devices : 5
   Working Devices : 5
    Failed Devices : 0
     Spare Devices : 0

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       2       8       48        2      active sync   /dev/sdd
       3       8       64        3      active sync   /dev/sde
       5       8       96        4      active sync   /dev/sdg
   ```
   Видим, что все диски активны и диск **sdg** занял место диска **sdf**

   

## Структура проекта
* `README.md` — документация проекта.
* `mdadm.log` — скрипт исполнения команд в терминале
